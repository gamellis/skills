---
name: apply-suggestions
description: Apply suggestions and fixes from a code review, handling optional changes and skipped items
argument-hint: "[optional change numbers to apply]"
---

# Apply Review Suggestions

Update the code with the suggestions and fixes from the review.

## Rules

- **Apply fixes and suggestions** from the review
- **Include optional changes** listed after this command. If $ARGUMENTS contains numbers, apply only those optional changes. If no numbers are provided, apply all optional changes.
- **Do NOT include work belonging to future tasks** — if a change is covered by a future task, leave it for that task to complete
- **Ask for clarification** if the review presents a choice between two or more solutions to a problem. Give each choice a short name or number so the user can select quickly.

## Output

After applying changes, include:

1. **Changed items** — list of items applied from the review
2. **Skipped items** — list of items intentionally not applied (with reason: future task, out of scope, etc.)
