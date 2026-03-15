---
name: pipeline-status
description: Check the pipeline status of a Hew feature issue. Use when the user says "/pipeline-status", "where is ES-XX at", "status of ES-XX", "check pipeline", or wants to see the state of a feature's Shape/Spec/Build subtasks and linked PRs.
user_invocable: true
---

# Pipeline Status

Shows the current state of a feature issue's pipeline: its Shape, Spec, and Build subtasks, their statuses, and any linked GitHub PRs.

## Steps

### 1. Get the feature ID

If the user provided an ID as an argument (e.g. `/pipeline-status ES-42`), use it. Otherwise ask for it.

The ID format is `XX-NNN` (team prefix + number, e.g. `ES-42`).

### 2. Fetch the feature issue and its children from Linear

Use the Linear MCP tools to get the issue and its subtasks:

1. Call `get_issue` with the feature identifier to get the parent issue (title, status, project).
2. Use the parent issue's ID to look up its children — call `list_issues` filtered to the parent.

### 3. For each subtask, check for linked PRs

For Shape, Spec, and Build subtasks, check if there's a GitHub branch or PR by looking at the subtask's identifier pattern:
- Shape branch: `<subtask-id>-shape` (e.g. `es-43-shape`)
- Use `gh pr list --head <branch-name> --json number,title,state,url` to check for PRs

### 4. Present a summary

Format the output as a clear status table:

```
ES-42: My Feature Title
Status: In Progress

  Shape  [ES-43]  Done     PR #7 (merged)
  Spec   [ES-44]  Todo     —
  Build  [ES-45]  Todo     —
```

Use status indicators that make it easy to see at a glance where things stand. If a subtask doesn't exist yet, show it as "Not created".
