---
name: run-daemon
description: Start the Hew daemon that polls for queued spec/build commands. Use when the user says "/run-daemon", "start the daemon", "start polling", or wants to run the local pipeline dispatcher.
user_invocable: true
---

# Run Daemon

Starts the Hew daemon, which polls the worker for queued `/spec` and `/build` commands and runs them against local repos.

## Steps

1. Check that `daemon/.env` exists. If missing, tell the user to run `/setup-repo` first or copy `.env.example` to `.env` and fill in the values.

2. Validate required vars are set (non-empty) in the `.env` file:
   - `POLL_URL`
   - `POLL_SECRET`
   - `REPOS`

   If any are missing or empty, tell the user which ones need filling in and stop.

3. Start the daemon:
   ```bash
   cd daemon && pnpm daemon
   ```
