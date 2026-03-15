---
name: setup-repo
description: Set up a new repository for the Hew pipeline. Use when the user wants to add a repo to the Hew daemon, connect a project to the factory pipeline, or says "/setup-repo". Handles cloning, daemon config, prerequisite checks, and Linear project reminders.
user_invocable: true
---

# Setup Repo

Connects a new repository to the Hew pipeline so the daemon can run spec generation and build loops against it.

## Steps

### 1. Get repo details

Ask the user for:
- **GitHub repo** (e.g. `gamellis/tally`) — required
- **Local path** — optional, defaults to `~/hew-repos/<repo-name>`

If the user provided these as arguments (e.g. `/setup-repo gamellis/tally`), parse them and skip asking.

### 2. Clone the repo

Check if the local path already exists. If not, clone it:

```bash
git clone git@github.com:<owner>/<repo>.git <local-path>
```

If the directory exists, confirm with the user that it's the right repo rather than silently proceeding.

### 3. Update daemon/.env

The daemon config lives at `daemon/.env` in this repo.

- If the file doesn't exist, copy from `.env.example` and warn the user they need to fill in `POLL_URL`, `POLL_SECRET`, and `CURSOR_API_KEY`.
- Find the `REPOS=` line. Append the new entry (`owner/repo:local-path`) with a comma separator if entries already exist.
- If the repo is already in the REPOS line, skip and tell the user.

Format: `REPOS=owner/repo:/absolute/path,owner/repo2:/absolute/path2`

Make sure to expand `~` to the full home directory path when writing to the file.

### 4. Check prerequisites

Check the cloned repo for these files and report what's present and what's missing:

| Prerequisite | Path | Why |
|---|---|---|
| Pitches directory | `pitches/` | Shape phase creates pitch files here |
| Agent config | `agents/AGENT.md` and `agents/skills/` | Codebase context for AI agents |
| Docker Compose | `docker-compose.yml` or `compose.yml` | Required for running Claude Code in Docker |

Missing items aren't blockers — just tell the user what they'll need to create before running the pipeline.

### 5. Linear project reminder

End with a reminder:

> Make sure your Linear project has a GitHub external link pointing to `https://github.com/<owner>/<repo>`. This is how Hew resolves which repo a command targets. Add it under Project Settings > External Links in Linear.
