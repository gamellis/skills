---
name: hew
description: "Use the Hew CLI for all pipeline and factory questions. Trigger this skill whenever the user mentions pipeline status, feature progress, spec/build/shape state, repo info, config, grading, queue, or refers to feature IDs like ES-42/ES-112. Also trigger for casual questions like 'where is ES-XX at', 'has the spec landed', 'what's building', 'kick off spec', 'run the build', 'is config valid', 'what repos are set up', 'check ES-XX', or any question about the Hew pipeline even if they don't say 'hew' explicitly. This skill supersedes the old pipeline-status skill — always prefer this one."
user_invocable: true
---

# Hew CLI

Use the `hew` CLI for all pipeline operations. The CLI inspects `.hew/` state on disk, polls the worker queue, and triggers factory scripts.

**Do not** manually grep git logs, read `.hew/` files directly, parse Linear/GitHub APIs, or use the old `pipeline-status` skill for pipeline questions. Use `hew` instead — it's faster and more reliable.

## Running hew

Every `hew` command needs the venv active and HEW_ROOT set. Run this setup once per shell session:

```bash
source ~/Documents/GitHub/hew/agent-harness/.venv/bin/activate && export HEW_ROOT=~/Documents/GitHub/hew
```

Then run commands normally:

```bash
hew pipeline status ES-42 --repo gamellis/skip
```

If you get "command not found: hew", the venv isn't activated — rerun the source line above. If you get "Could not find Hew project root", HEW_ROOT isn't set — rerun the export.

## Determining the repo

The `--repo` flag takes `owner/repo` format. If the user doesn't specify one:

1. Run `hew --json repo list` to see configured repos
2. Match the current working directory against the repo paths
3. If there's only one repo configured, use it
4. If ambiguous, ask

## Determining the feature ID

If the user doesn't specify a feature ID:

1. Check the current git branch — feature IDs look like `ES-42`, `ES-112` (two capital letters, dash, number)
2. Run `hew --json repo info <OWNER/REPO>` to see pitches and specs on disk
3. If still ambiguous, ask

## Commands

### Pipeline status (most common)

```bash
hew pipeline status <FEATURE_ID> --repo <OWNER/REPO>
```

Shows Shape/Spec/Build phase state by inspecting `.hew/` on disk. Use `--json` for structured output.

### Repo info

```bash
hew repo info <OWNER/REPO>
```

Lists pitches, specs, and active builds for a repo.

### Repo list

```bash
hew repo list
```

### Project health

```bash
hew info
```

Overview of all config, repos, and validation checks.

### Config validation

```bash
hew config validate
```

### Trigger spec generation

```bash
hew pipeline spec <FEATURE_ID> --repo <OWNER/REPO>
```

Runs `specgen.sh` — requires Docker.

### Trigger build loop

```bash
hew pipeline build <FEATURE_ID> --repo <OWNER/REPO>
```

Runs `ralph.sh` — requires Docker.

### Grade a spec

```bash
hew pipeline grade <SPEC_DIR> --passes 3
```

### Queue monitoring

```bash
hew queue list
```

Polls the worker for queued commands.

## Output

Use human-readable output by default. Use `hew --json <command>` when you need to process the result further or chain commands.
