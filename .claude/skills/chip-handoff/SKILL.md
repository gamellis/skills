---
name: chip-handoff
description: Hand the current conversation off to a task chip the user can click to open as a fresh session.
argument-hint: "What will the next session be used for?"
disable-model-invocation: true
---

Write a handoff summary of the current conversation so a fresh agent can continue the work. Instead of saving it, create a task chip seeded with the summary as its prompt using the `spawn_task` tool (`mcp__ccd_session__spawn_task`), with `cwd` set to the current working directory. The user clicks the chip to open it as a new session.

Always pass a descriptive `title` (imperative action phrase, e.g. "Fix login bug") and a 1-2 sentence plain-English `tldr` — they are shown on the chip.

Include a "suggested skills" section in the summary, which suggests skills that the agent should invoke.

Do not duplicate content already captured in other artifacts (PRDs, plans, ADRs, issues, commits, diffs). Reference them by path or URL instead.

Redact any sensitive information, such as API keys, passwords, or personally identifiable information — the summary becomes the agent's prompt.

If the user passed arguments, treat them as a description of what the next session will focus on and tailor the summary accordingly.
