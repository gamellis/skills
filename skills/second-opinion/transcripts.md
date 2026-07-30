# Exporting the current session's transcript

Each harness persists its own sessions locally. Use the recipe for the one you're running in.

Keep assistant and user text; reduce tool calls to a one-line marker so the reader can see where legwork happened without drowning in tool output.

## Claude Code

Sessions live in `~/.claude/projects/<slug>/<uuid>.jsonl`, where `<slug>` is the working directory with every `/` and `.` replaced by `-`. The current session is the most recently modified file there.

```bash
SLUG=$(pwd | sed 's/[/.]/-/g')
SESSION=$(ls -t ~/.claude/projects/"$SLUG"/*.jsonl | head -1)
jq -r '
  select(.type=="user" or .type=="assistant") | select(.isSidechain|not)
  | (.message.role // .type) as $role
  | ((.message.content // []) | if type=="string" then [{type:"text",text:.}] else . end)
  | map(if .type=="text" then .text
        elif .type=="tool_use" then "[tool: " + .name + "]"
        else empty end) | join("\n")
  | select(length>0) | "\n## " + $role + "\n\n" + .
' "$SESSION" > .scratch/second-opinion/<slug>.transcript.md
```

## Codex

Sessions live under `~/.codex/sessions/<year>/<month>/<day>/`, most recent last. Same JSONL shape in outline — inspect one line first and adjust the filter to the keys you find.

## Cursor

Chats live under `~/.cursor/chats/`. Inspect the store before writing a filter; the layout differs from the JSONL harnesses.

## If extraction fails

Don't reconstruct the conversation from memory — a remembered transcript is a summary, and a summary is the briefing this skill exists to avoid. Tell the user extraction failed, and ask them to paste the conversation into a file instead.
