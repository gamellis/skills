# skills

Agent skills I've written and use day to day. They're written to be repo-agnostic — nothing here assumes a particular project, language, or framework.

A skill is a folder with a `SKILL.md` describing a task in enough detail that an agent can do it well, plus any reference material it needs. They work with [Claude Code](https://claude.com/claude-code) and any agent that reads the same format.

## The skills

| Skill | What it does |
|---|---|
| [`fix-pr-comments`](skills/fix-pr-comments) | Fetches review comments from a GitHub PR, triages them, and fixes each one test-first — write a failing test capturing the reviewer's concern, make it pass, then resolve the thread. Handles skip-reason replies for feedback you're not taking. |
| [`write-a-pitch`](skills/write-a-pitch) | Collaborates with you on a [Shape Up](https://basecamp.com/shapeup) pitch — problem, appetite, solution, rabbit holes, no-gos. Explores your codebase so the pitch is grounded in real code, and aims for a pitch complete enough that an agent could generate a spec from it. |
| [`chip-handoff`](skills/chip-handoff) | Summarizes the current conversation and hands it to a fresh session as a clickable task chip, rather than writing a handoff file you have to shuttle around yourself. |

`fix-pr-comments` needs the [`gh`](https://cli.github.com) CLI. `chip-handoff` is the one skill here that isn't portable — it depends on the `spawn_task` tool, which only exists in the Claude Code desktop app.

## Installing

Skills are discovered by folder. Copy or symlink the ones you want into a skills directory your agent reads:

```bash
git clone https://github.com/gamellis/skills.git

# For one project (Claude Code)
ln -s "$PWD/skills/skills/write-a-pitch" /path/to/your/project/.claude/skills/write-a-pitch

# Or for every project
ln -s "$PWD/skills/skills/write-a-pitch" ~/.claude/skills/write-a-pitch
```

Symlinking means `git pull` here updates the skill everywhere. Copy instead if you'd rather pin a version and diverge.

Agents following the tool-neutral convention read `.agent/skills/` instead — same idea, different destination.

## Using

All three are user-invocable. In Claude Code that means you can call them by name:

```
/write-a-pitch we need better logging
/fix-pr-comments 42
/chip-handoff picking up the auth refactor
```

`write-a-pitch` and `fix-pr-comments` also trigger on their own when a request matches the skill's description — asking to "address the PR feedback" will pull in `fix-pr-comments` without you naming it. `chip-handoff` is deliberately invoke-only, since handing off is always a decision you make rather than one the agent should reach for.

## Contributing

These are shaped around how I work, so I'm unlikely to take feature requests that pull them elsewhere — but bug reports and fixes are welcome. Fork freely; MIT means you can adapt them to your own workflow without asking.

## License

[MIT](LICENSE)
