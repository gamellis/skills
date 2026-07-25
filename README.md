# skills

Agent skills I've written and use day to day. They're written to be repo-agnostic — nothing here assumes a particular project, language, or framework.

A skill is a folder with a `SKILL.md` describing a task in enough detail that an agent can do it well, plus any reference material it needs. They work with [Claude Code](https://claude.com/claude-code) and any agent that reads the same format.

## The skills

| Skill | What it does |
|---|---|
| [`fix-pr-comments`](skills/fix-pr-comments) | Fetches review comments from a GitHub PR, triages them, and fixes each one test-first — write a failing test capturing the reviewer's concern, make it pass, then resolve the thread. Handles skip-reason replies for feedback you're not taking. |
| [`write-a-pitch`](skills/write-a-pitch) | Collaborates with you on a [Shape Up](https://basecamp.com/shapeup) pitch — problem, appetite, solution, rabbit holes, no-gos. Explores your codebase so the pitch is grounded in real code, and aims for a pitch complete enough that an agent could generate a spec from it. |
| [`chip-handoff`](skills/chip-handoff) | Summarizes the current conversation and hands it to a fresh session as a clickable task chip, rather than writing a handoff file you have to shuttle around yourself. |
| [`writing-tighten`](skills/writing-tighten) | Checks a draft against nine prose rules — kill the warm-up, cut 10%, no adverbs, active voice, concrete over abstract — one pass per rule. Reports what each pass found and applies only the fixes you pick, so the edit stays yours. Command-only; it won't fire on its own. |

`fix-pr-comments` needs the [`gh`](https://cli.github.com) CLI. `chip-handoff` is the one skill here that isn't portable — it depends on the `spawn_task` tool, which only exists in the Claude Code desktop app.

`writing-tighten`'s rules are adapted from Kaguura Gichuru's [20,585 New Subscribers in 90 Days](https://kaguura.substack.com/p/90-days-20585-new-subscribers-heres). The essay-structure and audience-growth advice in that piece was left out — this is only the repeatable prose rules.

`chip-handoff` is adapted from [`claude-handoff`](https://github.com/mattpocock/skills/blob/main/skills/in-progress/claude-handoff/SKILL.md) by [Matt Pocock](https://github.com/mattpocock). His version hands off to a background agent; this one delivers the same summary as a clickable task chip.

## License

[MIT](LICENSE)
