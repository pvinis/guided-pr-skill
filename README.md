# guided-pr-skill

An [Agent Skill](https://agentskills.io) that adds a "guided tour" to a GitHub pull request: a sequence of linked inline review comments that walk the reviewer through the change in narrative order, each with Prev/Next navigation.

Works with both [Claude Code](https://claude.com/claude-code) and [pi.dev](https://pi.dev) since they share the Agent Skills format.

## Attribution

The guided-tour PR format is the work of [David Sheldrick (@ds300)](https://github.com/ds300). This skill teaches an agent to apply that format; the idea is his. Original examples worth reading in full:

- [artsy/eigen #3771, migrate routing logic to TS](https://github.com/artsy/eigen/pull/3771)
- [artsy/eigen #4055, refactor nav stack](https://github.com/artsy/eigen/pull/4055)

## Install

### Claude Code

```sh
git clone https://github.com/pvinis/guided-pr-skill.git
ln -s "$(pwd)/guided-pr-skill/guided-pr" ~/.claude/skills/guided-pr
```

Or copy `guided-pr/` into `~/.claude/skills/` directly.

### pi.dev

```sh
pi install github:pvinis/guided-pr-skill
```

Or copy `guided-pr/` into your pi.dev skills directory.

## Use

Once installed, ask your agent:

- "add a guided tour to PR #123"
- "write a walkthrough for this PR"
- "tour this PR"

The agent will read the PR, propose an ordered list of stops, draft each inline comment with you, then post them as a linked chain. You stay in control of ordering and prose at every step.

## License

MIT, see [LICENSE](./LICENSE).
