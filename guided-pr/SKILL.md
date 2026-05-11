---
name: guided-pr
description: Add a "guided tour" to a GitHub pull request — a sequence of linked inline review comments that walk the reviewer through the change in narrative order with Prev/Next navigation. Use when the user says "add a guided tour", "guided PR", "tour this PR", "write a walkthrough for #N", or asks to help reviewers understand a large PR. Inspired by David Sheldrick (@ds300)'s reviews on artsy/eigen #3771 and #4055.
---

# Guided PR

Add a "guided tour" to a pull request: an ordered chain of inline review comments that walks the reviewer through the change in narrative order, each with Prev/Next links. Authored by the PR author, for their own PR, before requesting review.

## Attribution

This skill is inspired by [@ds300](https://github.com/ds300)'s guided tours on [artsy/eigen #3771](https://github.com/artsy/eigen/pull/3771) and [artsy/eigen #4055](https://github.com/artsy/eigen/pull/4055). The format is his; this skill just teaches an agent to apply it. Credit belongs to him.

## When to use

- User owns a PR and wants reviewers to grasp it faster.
- User explicitly asks for a "guided tour", "guided PR", "tour", "walkthrough", "guide me through reviewing".
- The PR is large or non-obvious enough that linear file-by-file review will lose context.

## What you produce

For the specified PR:

1. **A "Start here" pointer in the PR body** linking to the first tour comment.
2. **An ordered chain of inline review comments**, each anchored to a specific file + line. Each comment has:
   - Prose explaining what changed at this location and *why*.
   - A footer with a step number, section title, and Prev/Next links.

## Inputs

Accept a PR reference: full GitHub URL, `#N`, `pr:N`, bare number, or "current branch". Resolve to `owner/repo` + PR number with `gh`.

If no reference is given, default to the current branch's PR: `gh pr view --json url,number,headRepository,headRefOid`.

## Footer format

Match ds300's style exactly. For tour stop `N`:

```
🔛 [N] {section title}\
_- - - - - - -_\
[{prev title}]({pr_url}/files#r{prev_id}) Prev ⬅️ ••• ➡️ Next **[{next title}]({pr_url}/files#r{next_id})**
```

Rules:
- Trailing backslash on each of the first two lines is required (Markdown hard line break). A plain newline collapses.
- `_- - - - - - -_` is the literal separator (italicised dashes with spaces).
- First stop: replace the `Prev` segment with `🏁 Start`.
- Last stop: replace the `Next` segment with `🎉 End`.
- The `🔛` prefix on the section title is ds300's "you are here" marker. Keep it.
- The section title in the footer should match the one-line summary the reviewer just read.

## Workflow

### 1. Gather context

```sh
gh pr view <ref> --json number,title,body,url,headRefOid,baseRefName,headRefName,files,headRepository
gh pr diff <ref>
```

Read the changed files in full where useful. Context outside the diff usually matters for a good tour.

### 2. Plan the tour

Talk to the user. Propose an ordered list of stops:

```
1. <file>:<line> — <one-line: what + why this is the right opener>
2. <file>:<line> — ...
...
N. <file>:<line> — <closer>
```

Iterate until the user is happy. Ordering is narrative ("how I'd explain this to a colleague over coffee"), not file path order. A typical tour is 8–30 stops; fewer is fine, more is usually too many.

### 3. Draft each comment

For each stop draft:
- Step number `[N]`
- Section title (will also appear in the footer)
- Body prose, 2–6 sentences. Link to other files/lines with full GitHub URLs liberally — guided tours are link-heavy by design.

Show all drafts to the user before posting. Editing posted comments works, but getting the prose right up front is cheaper than iterating after.

### 4. Post the chain (two-pass)

Posting needs two passes because Prev/Next reference future comment IDs.

**Pass 1 — create each comment with body prose only:**

```sh
gh api -X POST repos/{owner}/{repo}/pulls/{n}/comments \
  -f commit_id={head_sha} \
  -f path={path} \
  -F line={line} \
  -f side=RIGHT \
  -f body="<prose>" \
  --jq .id
```

Record each returned id in order. `commit_id` must be the current HEAD of the PR (`headRefOid` from step 1) or GitHub will mark the comment as outdated.

For comments anchored to a deleted line, use `side=LEFT` and the line number in the base file.

**Pass 2 — patch each comment to append the footer:**

Once all IDs are known, for each stop `i`:

```sh
gh api -X PATCH repos/{owner}/{repo}/pulls/comments/{id_i} \
  -f body="<prose>

<footer with real prev/next ids>"
```

Use a heredoc or `--field body=@file` for safety; embedded backticks and quotes in the prose will break shell escaping.

### 5. Update the PR body

Prepend a "Guided tour" section to the existing PR body:

```markdown
## Guided tour

This PR is a guided tour. Each comment links to the next; if a link doesn't scroll the page, open it in a new tab and wait a few seconds for GitHub to catch up.

### [Start here]({pr_url}/files#r{first_id})

1. [{section title 1}]({pr_url}/files#r{id_1})
2. [{section title 2}]({pr_url}/files#r{id_2})
...
N. [{section title N}]({pr_url}/files#r{id_N})

---
```

The numbered list is a table of contents. Generate it from the same step numbers and section titles used in the comment footers, in tour order. Reviewers can jump to any stop directly, or follow the chain in order from "Start here".

Then:

```sh
gh pr edit <ref> --body "<new body>"
```

Read the existing body first and preserve it below the inserted section.

## Common pitfalls

- **`commit_id` must match current PR HEAD.** If the user pushes between Pass 1 and Pass 2, the comments still work but new comments would attach to a stale SHA. Refetch `headRefOid` if a push happened.
- **`line` is the line number in the file at that side**, not the diff hunk position. `side=RIGHT` uses the HEAD file's line numbering; `side=LEFT` uses the base.
- **GitHub silently rejects comments outside the diff** with a 422. Before posting, verify each `(path, line)` pair against `gh pr diff` output.
- **Hard line breaks need trailing `\` + newline.** Single newlines collapse into one line, breaking the footer layout.
- **Links don't always scroll on first click.** This is a GitHub quirk, not a skill bug. The PR body should warn reviewers (the template above does).
- **Don't post too many stops.** If the tour has more than ~30 stops, the narrative is probably wrong — split the PR or pick a tighter spine.

## Don'ts

- Don't post the whole tour without showing drafts to the user first.
- Don't skip Pass 2 — a chain with broken Prev/Next is worse than no chain.
- Don't add tour comments to a PR you don't own without explicit permission.
- Don't add @ds300 attribution to the PR body or tour comments themselves — keep attribution in the skill's own docs (this file and the README), not in PRs the skill operates on.
