---
name: session-handoff
description: Hand the current conversation off to a Claude Code session card the user can click to open as a fresh session.
argument-hint: "What will the next session be used for?"
disable-model-invocation: true
---

Write a handoff summary of the current conversation so a fresh agent can continue the work. Instead of saving it, propose a session card seeded with the summary, using `mcp__ccd_session__start_session` (load it with `ToolSearch` `select:mcp__ccd_session__start_session` if it is deferred). The user clicks the card to start it as a new session.

Call it with:

- `initiation: "own_initiative"`: a card that waits for the user's click.
- `context: "fresh"`: the new session sees only the brief, not this conversation.
- `title`: an imperative action phrase under 60 characters, e.g. "Fix login bug". It is shown on the card.
- `prompt`: the handoff summary itself.
- `background`: 1–2 plain-English sentences of the overall goal and how this piece fits. At least 80 characters.

A card opens in the current project and cannot take `cwd`, `host`, or `preset`. When the work lives in another folder or repo, name its absolute paths in the summary.

Include a "suggested skills" section in the summary, which suggests skills that the agent should invoke.

Do not duplicate content already captured in other artifacts (PRDs, plans, ADRs, issues, commits, diffs). Reference them by path or URL instead.

Redact any sensitive information, such as API keys, passwords, or personally identifiable information — the summary becomes the agent's prompt.

If the user passed arguments, treat them as a description of what the next session will focus on and tailor the summary accordingly.
