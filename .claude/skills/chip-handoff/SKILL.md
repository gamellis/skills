---
name: chip-handoff
description: Hand the current conversation off to a new session via a spawn_task chip — one click opens the fresh session.
argument-hint: "What will the next session be used for?"
disable-model-invocation: true
---

Write a handoff summary of the current conversation so a fresh agent can continue the work. Instead of saving it to a file or launching a background agent, create a task chip with the `mcp__ccd_session__spawn_task` tool, passing the summary as the `prompt`. The user clicks the chip to open it as a new session.

- `title`: imperative action phrase describing the continuation (e.g. "Continue relay auth refactor").
- `tldr`: 1-2 plain-English sentences on what the next session will do.
- `prompt`: the full handoff summary — it must stand alone, with absolute file paths and enough context to act without this conversation.
- `cwd`: set to the current working directory so the new session starts in the right place.

Include a "suggested skills" section in the summary, which suggests skills that the agent should invoke.

Do not duplicate content already captured in other artifacts (PRDs, plans, ADRs, issues, commits, diffs). Reference them by path or URL instead.

Redact any sensitive information, such as API keys, passwords, or personally identifiable information — the summary becomes the new session's prompt.

If the user passed arguments, treat them as a description of what the next session will focus on and tailor the summary accordingly.

If the spawn_task tool is unavailable (e.g. running in a plain terminal rather than the desktop app), fall back to the `handoff` skill's behavior: save the summary to the OS temp directory and tell the user where it is.
