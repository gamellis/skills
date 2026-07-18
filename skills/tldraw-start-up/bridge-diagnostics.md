# Bridge diagnostics

This file is reference for the startup companion to the official
[tldraw-offline skill](../tldraw-offline/SKILL.md). It does not replace that
skill's canvas or editor instructions.

The tldraw Offline app writes its agent metadata to:

```text
~/Library/Application Support/tldraw/server.json
```

The app bundle starts an HTTP server on `127.0.0.1:7236` and writes the active token and PID to that file. The app-managed helper is installed at:

```text
`$HOME/skills/tldraw-offline/tq`
```

The helper path inside the Codex skill directory may not exist; prefer the app-installed path under `$HOME`.

## Evidence-backed recovery

The app can be visibly open while the bridge metadata is stale or while a sandbox cannot reach the host loopback interface. Diagnose in this order:

```bash
cat "$HOME/Library/Application Support/tldraw/server.json"
lsof -nP -iTCP:7236 -sTCP:LISTEN
```

If `lsof` shows `tldraw` listening on `127.0.0.1:7236`, the bridge is up. Run curl from the host context; a sandboxed curl can fail even though the listener is healthy.

The authoritative end-to-end check is:

```bash
PORT=$(jq -r .port "$HOME/Library/Application Support/tldraw/server.json")
TOKEN=$(jq -r .token "$HOME/Library/Application Support/tldraw/server.json")
curl -sS -X POST "http://127.0.0.1:$PORT/api/search" \
  -H "authorization: Bearer $TOKEN" \
  -H 'content-type: application/json' \
  --data-binary '{"code":"const doc = await api.getFocusedDoc(); return {connected:true, doc}"}'
```

If the host-context check fails, quit and relaunch the application itself, not only the document, then reread `server.json`. Do not edit the working SQLite files or delete metadata as a first response.

The public tldraw Offline README describes the app as an agent-capable local desktop application with no remote account or sync server. The direct editor API is the intended control surface; the local bridge is managed by the desktop app.
