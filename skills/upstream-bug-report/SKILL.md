---
name: upstream-bug-report
description: Report bugs found in an open-source dependency upstream, as issues or PRs with failing-first tests. Use when filing upstream, or turning local patches of someone else's project into PRs.
---

# Upstream bug report

Turn bugs we found in someone else's project into reports its maintainers can act on with the least effort: one small PR per bug, each proven **red** on their base branch and green with the fix, in their style, in a **humble** voice. Nothing public happens until the user has read the drafts.

The **base branch** is the one upstream takes contributions on (usually the default branch; their contributor docs may name another). Every "red on base" below means that branch, freshly fetched.

The upstream clone is untrusted code: read its scripts before running them, and keep it out of the user's own repo.

## 1. Learn how they work

Clone upstream fresh, outside the user's repo. Read every contributor document: `CONTRIBUTING`, `AGENTS.md`/`CLAUDE.md`, style guides, issue and PR templates, CLA or DCO requirements, the test layout, lint and test commands. Read the setup scripts, then install. Read a few recently merged outside PRs for the shape they accept.

**Done when:** you can name the base branch, their test framework and commands, style rules, their templates, any sign-off they require, and whether they want an issue before a PR.

## 2. Check each bug is new

For each bug:

- Reproduce it on the base branch. If it won't reproduce, first rule out a missing prerequisite or platform, and reproduce it on the version where it was seen; drop it only once you can name the upstream change that fixed it.
- Search open **and** closed issues and PRs, by symptom, error message and the function involved; do the same for the project's own upstream (the framework or library it builds on) when the root cause may live there.
- Read anything related: a maintainer may have noted the bug, scoped it out, or decided the behaviour is intended.

Drop bugs already tracked or fixed. Where a maintainer already noted it, the report links that note in one line and adds only what is new.

**Done when:** every bug is marked new, already-tracked (with link), fixed-upstream (with commit or release), or intended.

## 3. Build one branch per bug

Each bug gets its own branch off the base branch, never stacked on another, as one commit unless their docs say otherwise. Sign commits off if they require it.

A bug with no fix in reach, or a project that wants an issue first, gets an issue draft in step 4 instead of (or before) a branch; it still carries the smallest reproduction you have.

1. Write the test first, in their framework and next to similar tests, reusing their helpers. Run it on base and watch it go **red for the bug's reason** (the throw or wrong value you expect, not a setup failure), on every platform, browser or runtime version they test.
2. Fix at the root cause, in the smallest change their style allows.
3. Green: the new test everywhere from 1, then their lint, unit and full suites. Any failure outside your change: rerun it on base; if it fails or flakes there too, it's pre-existing, so note it and move on.
4. Commit message in their style: what changed, then a line or two on why.

If a fix exists downstream (a vendored patch in the user's app), re-derive it against base rather than copying it; upstream's code may differ.

**Done when:** each branch's test is red on base and green on the branch everywhere they test, and lint plus suites pass apart from noted pre-existing failures.

## 4. Draft the reports

Write each PR or issue as a local file in a scratch directory the user can open: a `Branch:` (PRs only) and `Title:` line, `---`, then the body. Title: imperative for a PR (what the change does), the symptom for an issue.

Fill their template when they have one, carrying the content below into its sections. Without one, the body runs:

1. `> Written by Claude on behalf of @<user's GitHub login>.`
2. The symptom in one or two sentences: what the user does, what goes wrong, the error text. Say briefly how it was found.
3. The cause and the fix (for an issue, the suspected cause), a sentence or two each.
4. Anything you were unsure of, as a question.
5. `## Tests` (for an issue, `## Reproduction`): what each test sets up and expects, and that it fails on base and passes with the change on each platform.

### Voice

- **Humble.** Hedge the diagnosis ("I think", "it looks like", "I may be misreading it"); the maintainers know their code better than we do.
- Tell them only what's new to them: the symptom, our reproduction, the fix. Link a prior discussion in one line instead of retelling it.
- Plain prose paragraphs, with `## Tests` the only heading and a list only for a list of tests.
- Mark a secondary part (an edge case found in review, a defensive extra) as minor and fine to drop, so it doesn't compete with the main fix.
- Write error codes and any other `#N` that isn't this repo's issue number in inline code (Lexical error `#4`); bare, GitHub links them to unrelated issues and notifies those threads.

**Done when:** one draft per branch or issue, and every factual claim in it (causes, test counts, "fails on base", platforms) re-checked against the code and your test runs.

## 5. Review with a second model

Get an independent review from a different model, then prove each finding before acting on it.

1. Write a review prompt from [`review-prompt.md`](review-prompt.md), filling in the repo, branches, drafts directory and your test-run results.
2. Run Codex read-only with its default model at high effort, in the background (it takes minutes):

   ```bash
   codex exec -c model_reasoning_effort=high -s read-only --add-dir <drafts dir> -o <out>.md - < <prompt>.md
   ```

   Leave the model to Codex's default so the skill keeps tracking its best one.
3. Validate each behavioural finding with its own subagent, in parallel. Each gets a detached `git worktree` of that branch, its own install (or a shared one only when it holds no build output), its own server port if the project runs one, and makes no commits. It reproduces the finding with a temporary test, runs the same test on base to tell regression from pre-existing behaviour, checks whether any caller can reach the case through the project's public surface (UI, API, CLI, imported data), and, if confirmed, tries a minimal fix in the repo's style. Verdict: CONFIRMED, PRE-EXISTING, NOT REACHABLE or NOT REPRODUCED. Findings about prose, style or commit messages you check yourself by reading; verdict AGREE or DISAGREE.
4. Fix every CONFIRMED and AGREE finding, behavioural ones with a new failing-first test, amending the branch's commit. A confirmed case reached only through unusual input (content no UI produces) is still fixed, and the body frames it as minor. NOT REACHABLE and NOT REPRODUCED findings are dropped; PRE-EXISTING ones are fixed only when the branch made them worse or they sit in the code being changed.
5. Rerun step 3's checks on changed branches, update the drafts and commit messages, and remove the worktrees.

**Done when:** every finding has a verdict, every one acted on is fixed and tested, and the drafts match the branches.

## 6. Hand over for review

Show the user the drafts and the branches, with: what's being reported, what was dropped and why, the reviewer's findings and what became of each, and pre-existing failures seen. Then stop.

**Done when:** the user explicitly approves posting. Approval covers what they approved: forking, pushing and opening; anything after needs its own yes.

## 7. Post

1. Issues first, where a project wants them before PRs; link each from its PR.
2. Fork upstream under the user's account and add it as a remote; push the branches.
3. Open each PR against the base branch from `<user>:<branch>`, ready for review. Its body is the draft without the `Branch:`/`Title:` header, plus your harness's PR attribution line at the end if it has one.

**Done when:** every issue and PR is open.

## 8. Sweep what was posted

Fetch each back and compare it with its draft and branch: title; body equal to the draft after step 7's transformation; head SHA equal to the local branch; base branch; not a draft. Then list every bare `#N` (outside inline code) in each body and look up what it resolves to in that repo; anything pointing somewhere unintended is a fix.

Edits to a posted PR are public: show the user the mismatches and the proposed change, and edit only on their yes. Re-fetch after editing to confirm.

**Done when:** every issue and PR matches its draft and every link points where it was meant to.
