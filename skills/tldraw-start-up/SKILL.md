---
name: tldraw-start-up
description: Use automatically when starting a tldraw Offline task, connecting to an open tldraw canvas, launching or restarting tldraw, checking server.json, diagnosing a localhost bridge failure, fixing a stale tldraw server, or when the tldraw editor API cannot connect.
---

# tldraw Offline startup

This is a startup and connection companion to the official [tldraw-offline skill](../tldraw-offline/SKILL.md). Read that skill first for canvas operations, editor API usage, document scripts, shape safety, and verification. This skill owns only startup and bridge diagnosis.

## Startup

1. Read `~/Library/Application Support/tldraw/server.json` on every run. Load its current `port` and `token`; both are per-launch values.
2. Prefer the app-installed helper at `"$HOME/skills/tldraw-offline/tq"`. It reads the current metadata for each request.
3. Verify `GET /readme`, then make an authenticated `POST /api/search` request returning `await api.getDocs()`.
4. If a sandboxed shell reports that localhost is unreachable, repeat the same read-only request in the host context with the required escalation. Treat sandbox loopback visibility and bridge health as separate states.
5. If the host-context request fails, inspect the current `server.json`, check whether its PID is alive, and ask the user to launch tldraw Offline if the app is closed. After launch, reread `server.json` before retrying.

Startup is complete only when an authenticated `/api/search` call returns the open-document list. Hand control back to the official skill at that point.

The bridge is an app-managed service; there is no separate tldraw Offline server command to start. A fresh port or token means the app was relaunched, so always reread metadata.

## Bridge checks

```bash
sh "$HOME/skills/tldraw-offline/tq" POST /api/search \
  '{"code":"return await api.getDocs()"}'
```

For the detailed startup diagnosis and recovery evidence, read [bridge-diagnostics.md](bridge-diagnostics.md).
