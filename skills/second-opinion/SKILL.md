---
name: second-opinion
description: Ask a different AI and read its take.
disable-model-invocation: true
argument-hint: "[harness] [model] [effort] — e.g. codex gpt-5.6-sol high"
---

# Second opinion

Put a question to a **different harness** — a different model — and bring back its take. Copy-paste your session into it as is, no changes.

## 1. Export the transcript

Dump this session's conversation to `.scratch/second-opinion/<slug>.transcript.md`, `<slug>` naming the question (outside a repo, use a temp file). See [`transcripts.md`](transcripts.md) for the extraction recipe for the harness you're running in.

Close with a rule and the live question in full. It's the last thing read, and it has to outrank everything above it — a bare "Answer Q2" doesn't.

The transcript ships everything it contains — command output, file contents, whatever your tool calls returned — to another vendor's CLI. If anything in this session shouldn't travel, say so and let the user decide before you send.

**Done when:** the file reads as the conversation, in order, including the recommendations you gave along the way.

## 2. Ask the other harness

The invocation takes `[harness] [model] [effort]`, each optional. For what wasn't given, ask the user which harness — naming the ones installed here other than the one you're running in — and let the CLI use its own default model and effort.

Run it read-only from the repo root, feed the transcript in as the prompt, and capture the answer to `<slug>.answer.md`. Background it; these take minutes.

## 3. Report the divergence

Give the user, in order:

1. **The answer, verbatim** — the whole of what came back, copied through unedited.
2. **Compare** — the specific claim, weighting, and assumptions.

Check any repo fact it asserts that you hadn't, and say what you found.

Hold both takes open. 

## 4. Resume

Return to the grilling question and wait for the user's answer.
