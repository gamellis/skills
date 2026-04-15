# Example Pitches

Two real pitches at different scales. Study these for tone, level of detail, and how each ingredient works.

---

## Example 1: Small Pitch — `hew doctor` (Preflight Validation)

### Problem

After running `hew init`, the user has no way to verify everything is wired up correctly. Tokens could be expired, the relay could be misconfigured, the webhook could be pointing at the wrong URL. Without a diagnostic tool, the user won't know until the foreman fails at runtime.

### Appetite

Small. A single command that runs a checklist of validation steps. No interactivity, no fixing — just clear pass/fail with actionable suggestions.

### Solution

A `hew doctor` command that validates the full stack from prerequisites to service connections. Runs standalone — does not require the foreman to be running.

**Check list:**

```
$ hew doctor

Prerequisites
  ✓ node v22.1.0
  ✓ pnpm 9.1.0
  ✓ wrangler 3.60.0

Config
  ✓ ~/.config/hew/config.json exists and valid
  ✓ At least one repo registered

Relay
  ✓ Relay URL configured: hew-relay.user.workers.dev
  ✓ Relay reachable (HTTP 200)

Linear
  ✓ Linear access token valid (from keychain)
  ✓ Webhook active and pointing at relay URL

GitHub
  ✓ GitHub access token valid (from keychain)
  ✓ Token has required scopes (contents ✓, pull-requests ✓)

Ready! Run "hew start" to launch the foreman.
```

**Failure output** — each failure includes a one-line fix suggestion:

```
  ✗ Linear access token expired
    → Run "hew init" to re-authorize Linear

  ✗ Relay not reachable at hew-relay.user.workers.dev
    → Run "cd relay && wrangler deploy" to redeploy
```

**How each check works:**
- Prerequisites: shell out to `which` + `--version`
- Config: read and validate JSON, check repos array non-empty
- Relay reachable: HTTP GET to relay URL health endpoint
- Linear/GitHub tokens: API call with token from keychain

**Exit code:** `0` if all pass, `1` if any fail. Useful for scripting (`hew doctor && hew start`).

### Rabbit Holes

- **Partial success** — don't bail on first failure. Run all checks and report everything at once so the user can fix multiple issues in one pass.
- **Network timeouts** — keep HTTP checks fast (2-second timeout). Don't let a slow relay hang the whole doctor run.
- **Token introspection** — checking scopes on GitHub is easy (response header). Linear doesn't expose scopes the same way. Just verify the token works by making an API call.

### No-Gos

- Don't fix problems automatically — report and suggest
- Don't require the foreman to be running
- Don't check repo-level state (that's beyond doctor's scope)
- Don't cache results — always check live state

---

## Example 2: Small Pitch — Thin Cloudflare Relay (Webhook Receiver)

### Problem

The current worker is ~1,500 lines handling Linear webhooks, GitHub API calls, Linear GraphQL, KV command queuing, and daemon polling. It needs 4 secrets and a KV namespace. As the project moves to the Linear Agent API, all of that orchestration shifts to the foreman — the relay's only job is receiving webhooks and getting them to the foreman.

The current KV poll pattern also won't scale: at a 5-second polling interval, KV list operations hit ~17,280/day — well over the 1,000/day free tier limit.

### Appetite

Small. The relay is intentionally minimal — the less code, the better. Build it alongside the existing worker so the current pipeline keeps running. Decommissioning the old worker is a separate effort.

### Solution

A new Cloudflare Worker backed by a single Durable Object. Two auto-deployed secrets. Three responsibilities plus a health endpoint:

1. **Receive** webhooks from Linear (HMAC-SHA256 signature verification)
2. **Store** events in Durable Object transactional storage (SQLite, free tier)
3. **Push** events to the foreman over a persistent WebSocket connection

**How it works:**
- When the foreman is connected: webhook arrives → DO pushes it over WebSocket immediately
- When the foreman is offline: events accumulate in DO storage. On reconnect, the DO replays missed events, the foreman acks, and the DO cleans them up
- `GET /` returns 200 — a health endpoint for diagnostics

**Why Durable Objects over KV:**
- Events survive foreman downtime, replayed on reconnect
- WebSocket support — instant delivery, no polling
- Free tier headroom — 100K requests/day vs KV's 1,000/day limit
- SQLite storage — consistent reads/writes, not eventually consistent

**What it doesn't do:** Everything else. No Linear API calls, no GitHub API calls, no command parsing. That all lives in the foreman now.

### Rabbit Holes

- **Webhook verification** — use the standard HMAC-SHA256 signing with the `Linear-Signature` header and the auto-provisioned signing secret. Don't invent a custom scheme.
- **Message ordering** — webhooks aren't guaranteed in order. Store timestamps and let the foreman sort if needed, don't try to enforce ordering in the relay.
- **Multi-foreman support** — don't design for it. One foreman per installation. Keep it simple.

### No-Gos

- No Linear API calls from the relay
- No GitHub API calls from the relay
- No command parsing or interpretation of webhook payloads
- No user-managed secrets — both secrets are auto-deployed during init

---

## What to notice

Both pitches share patterns worth emulating:

1. **Problem tells a specific story** — not "we need X" but "here's what breaks and why it's unacceptable"
2. **Appetite is a design constraint** — "Small" isn't just a label, it's followed by *why* small is the right scope
3. **Solution is breadboard-level** — describes topology (what connects to what) without wireframes or code
4. **"What it doesn't do" sections** — the solution explicitly names what it excludes, reinforcing boundaries
5. **Rabbit holes are specific and resolved** — each one names the trap AND the compromise
6. **No-gos are firm exclusions** — not "nice to haves we'll skip" but "we are deliberately not doing this"
