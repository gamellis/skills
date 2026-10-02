---
name: ticket-handoff
description: Hand the current conversation off to a ticket in the user's project management tool (Linear, GitHub Issues, Jira, etc.) so a fresh agent or person can pick it up later.
argument-hint: "What will the ticket be used for?"
disable-model-invocation: true
---

Write a handoff summary of the current conversation so a fresh agent can continue the work. Instead of saving it, file it as a ticket in the user's project management tool, with the summary as the ticket body.

Work out which tool from the project's own docs first — `CLAUDE.md`, `AGENTS.md`, or something like `docs/agents/issue-tracker.md` — and follow the team, project, labels, and CLI they name. If the tool or where in it the ticket belongs is still unclear, ask the user before creating anything; never guess.

Prefer the tool's CLI (`linear`, `gh`, `jira`) over an MCP connector when both are available.

Always give the ticket a descriptive title (imperative action phrase, e.g. "Fix login bug").

Include a "suggested skills" section in the summary, which suggests skills that the agent should invoke.

Do not duplicate content already captured in other artifacts (PRDs, plans, ADRs, issues, commits, diffs). Reference them by path or URL instead — and link related tickets with the tool's own relations where it has them.

Redact any sensitive information, such as API keys, passwords, or personally identifiable information — the ticket may be visible to the whole team.

If the user passed arguments, treat them as a description of what the ticket will be used for and tailor the summary accordingly.

Finish by replying with the ticket's identifier and URL.
