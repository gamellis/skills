---
name: fix-pr-comments
description: "Fetch, triage, and fix PR review comments using a TDD-based process. Use this skill whenever the user wants to address PR feedback, fix review comments, resolve PR conversations, work through code review suggestions, or mentions PR comments they need to handle. Also trigger when the user says things like 'fix the PR comments', 'address the review', 'go through the feedback', 'what did the reviewer say', or references a PR number with comments to resolve."
user_invocable: true
argument-hint: "[PR number or 'all' or comment numbers to fix]"
---

# Fix PR Comments

Fetch review comments from a GitHub PR, triage them, and apply fixes using a test-driven process. Each fix follows a red-green-refactor cycle: write a failing test that captures the reviewer's concern, implement the minimal fix, then clean up.

## Step 1: Identify the PR

Determine which PR to work on:

1. If the user provided a PR number (via argument or prompt), use it
2. Otherwise, detect from the current branch:
   ```bash
   gh pr view --json number,title,url,headRefName 2>/dev/null
   ```
3. If no PR is found, ask the user

## Step 2: Fetch Comments and Establish Baseline

Pull all review comments and general PR comments:

```bash
# Review comments (on specific lines of code)
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments --paginate

# General PR comments (conversation-level)
gh api repos/{owner}/{repo}/issues/{pr_number}/comments --paginate

# Also grab the review summaries for context
gh api repos/{owner}/{repo}/pulls/{pr_number}/reviews --paginate
```

For each comment, capture: id, author, body, file path + line (if review comment), whether it's part of a resolved conversation, and any reply thread.

### Find the pre-review state

To properly evaluate whether comments are still relevant — and to write meaningful failing tests — you need the code as it was when the reviewer commented. This is the commit the review was written against, not the current HEAD (which may already contain fixes).

```bash
# Get the first commit on the PR and the current HEAD
gh api repos/{owner}/{repo}/pulls/{pr_number} --jq '{base_sha: .base.sha, head_sha: .head.sha}'

# Look at the commit history to find the state before any fix-up commits
git log --oneline base_sha..head_sha
```

If later commits explicitly address review comments (commit messages like "fix review feedback", "address comments", or the PR author replied "fixed in abc123"), identify the **original commit** — the one the reviewer was looking at. You'll check out this commit when writing failing tests in Step 5 to confirm the bug actually exists before fixing it.

Even when the PR is already merged, always check out the pre-fix commit to validate the reviewer's concern. This matters because:
- It proves the test catches a real bug (not a hypothetical one)
- It confirms the fix actually resolves the issue
- It produces a meaningful regression test that guards against reintroduction

## Step 3: Triage

Categorize every comment into one of these buckets:

| Category | Description | Action |
|----------|-------------|--------|
| **fix** | Actionable code change needed — bug, logic error, missing validation, wrong behavior | Fix via TDD |
| **nit** | Style, naming, formatting, minor preference | Apply directly (no test needed) |
| **question** | Reviewer asking for clarification, not requesting a change | Flag for user to respond |
| **resolved** | Already addressed in a subsequent commit or conversation marked resolved | Verify fix, write regression test |
| **out-of-scope** | Valid concern but belongs in a separate PR or future work | Skip, note for user |

Present a summary table to the user:

```
## PR #42 — Review Comments (7 total)

### Fixes (3)
  1. [fix] auth.ts:45 — @reviewer: "This doesn't handle expired tokens"
  2. [fix] api.ts:112 — @reviewer: "Race condition when concurrent requests hit this"
  3. [fix] utils.ts:30 — @reviewer: "Off-by-one in pagination offset"

### Nits (2)
  4. [nit] auth.ts:12 — @reviewer: "Rename to isTokenValid for clarity"
  5. [nit] api.ts:5 — @reviewer: "Unused import"

### Questions (1)
  6. [question] — @reviewer: "Why not use the existing rate limiter?"

### Skipped (1)
  7. [resolved] api.ts:80 — already fixed in commit abc123
```

Ask: **"Which comments should I address? (numbers, 'all', 'fixes', 'nits', or specific like '1,3,4')"**

If the user provided a filter as an argument (e.g., `/fix-pr-comments all` or `/fix-pr-comments 1,3`), use that instead of asking.

## Step 4: Detect Test Framework

Before writing any tests, figure out what testing tools the project uses. Check for:

- `jest.config.*`, `vitest.config.*` — JS/TS
- `pytest.ini`, `pyproject.toml` [tool.pytest], `conftest.py` — Python
- `Gemfile` with rspec/minitest, `test/` or `spec/` dirs — Ruby
- `*_test.go`, `go test` — Go
- `Cargo.toml` with `[dev-dependencies]` — Rust

Look at existing test files to understand patterns: file naming, import style, how fixtures work, where tests live. Match these conventions exactly — the new tests should look like they belong.

## Step 5: Fix via TDD

For each selected **fix** comment (and each **resolved** comment worth verifying), work through a vertical slice.

### Check out the pre-fix state

Before writing a test, switch to the commit the reviewer was looking at:

```bash
git stash  # save any in-progress work
git checkout <original_commit>
```

This ensures your test actually fails against the buggy code. If you write a test against the already-fixed HEAD, you can't confirm the test catches anything real.

### RED — Write a failing test

Read the code the reviewer flagged. Understand the concern. Write one test that captures the broken or missing behavior the reviewer identified.

The test should:
- Exercise the code through its public interface, not internal details
- Describe the behavior in its name: `test_expired_token_returns_401`, not `test_auth_fix`
- Fail for the right reason — the behavior the reviewer flagged is genuinely missing or broken
- Be an integration-style test where possible (real code paths, minimal mocking)

Only mock at system boundaries (external APIs, databases if no test DB, time/randomness). Never mock your own modules.

Run the test against the pre-fix commit and confirm it fails. This validates the reviewer's concern was real.

### GREEN — Minimal fix

If the fix already exists in a later commit, cherry-pick or re-apply it and confirm the test passes. If not, write the smallest change that makes the test pass. Don't clean up, don't refactor, don't fix adjacent issues. Just make the red test green.

```bash
# Return to HEAD with the new test
git checkout <head_branch>
git stash pop  # restore any stashed work
```

Run the test against the current HEAD — it should pass (either because the fix was already applied, or because you just wrote it). Run the full test suite to make sure nothing broke.

### REFACTOR — Clean up

Now that tests are green, look for cleanup opportunities in the code you just touched:
- Extract duplication
- Improve naming
- Simplify conditionals

Run tests after each refactor step. Never refactor while red.

### Move to the next comment

One comment at a time. Don't batch. Each cycle is self-contained: you learn from each fix and that informs the next one.

## Step 6: Apply Nits

For selected **nit** comments, apply them directly — rename variables, remove unused imports, fix formatting. These don't need tests since they don't change behavior. Run the test suite after all nits are applied to confirm nothing broke.

## Step 7: Resolve Fixed Conversations

After fixing comments, resolve their review threads on GitHub so reviewers can see they've been addressed. Use the GraphQL API since the REST API doesn't support resolving threads.

First, fetch all review threads and their node IDs:

```bash
gh api graphql -f query='
  query($owner: String!, $repo: String!, $pr: Int!) {
    repository(owner: $owner, name: $repo) {
      pullRequest(number: $pr) {
        reviewThreads(first: 100) {
          nodes {
            id
            isResolved
            comments(first: 1) {
              nodes {
                body
                path
                line
                databaseId
              }
            }
          }
        }
      }
    }
  }
' -f owner='{owner}' -f repo='{repo}' -F pr={pr_number}
```

Match each thread to the comments you fixed by comparing the `databaseId`, file path, and line number against the comments from Step 2. For every thread whose comment you fixed or applied as a nit, resolve it:

```bash
gh api graphql -f query='
  mutation($threadId: ID!) {
    resolveReviewThread(input: {threadId: $threadId}) {
      thread {
        id
        isResolved
      }
    }
  }
' -f threadId='<thread_node_id>'
```

Only resolve threads for comments you actually addressed (categories: **fix**, **nit**, **resolved**). Do NOT resolve threads for **question** or **out-of-scope** comments — those still need human attention.

For **question** and **out-of-scope** comments, reply to the thread explaining why they weren't addressed:

```bash
# Reply to a review comment thread
gh api repos/{owner}/{repo}/pulls/{pr_number}/comments/{comment_id}/replies \
  -f body='<explanation>'
```

Keep replies concise and actionable:
- **question**: Explain that this needs a human response, and include any context you gathered that might help the PR author answer. E.g., *"This needs a response from the author — the existing rate limiter is in `src/middleware/rate-limit.ts` if that helps frame the reply."*
- **out-of-scope**: Explain why it doesn't belong in this PR and suggest where it should be tracked. E.g., *"Valid concern — this is a broader refactor beyond the scope of this PR. Recommend tracking as a follow-up issue."*

## Step 8: Report

After all selected comments are addressed, summarize:

```
## Done — PR #42

### Fixed (3)
- [1] auth.ts:45 — expired token handling
  - Test: test_expired_token_returns_401 (auth.test.ts:89)
  - Fix: Added expiry check in validateToken()

- [2] api.ts:112 — race condition
  - Test: test_concurrent_requests_dont_corrupt_state (api.test.ts:145)
  - Fix: Added mutex around shared state update

- [3] utils.ts:30 — pagination off-by-one
  - Test: test_pagination_offset_starts_at_zero (utils.test.ts:67)
  - Fix: Changed offset calc from (page * size) to ((page - 1) * size)

### Nits Applied (2)
- [4] Renamed isValid → isTokenValid in auth.ts
- [5] Removed unused import in api.ts

### Needs Your Input (1)
- [6] @reviewer asked: "Why not use the existing rate limiter?"
  → You'll want to reply to this conversation on GitHub

### Skipped (1)
- [7] Already resolved
```

## Edge Cases

**Already-merged PRs**: The most common case. The PR is closed, fixes were applied before merge. The skill is still valuable here — check out the pre-fix commit, validate the reviewer's concern with a failing test, then confirm the fix resolves it. The output is a regression test suite, not code changes. This is the skill working as intended, not a degenerate case.

**Comment threads**: When a review comment has replies, read the full thread to understand the resolution. The last reply might say "actually never mind" or "fixed in latest push."

**Suggested changes**: GitHub review comments can include code suggestions (in ```suggestion blocks). When present, evaluate whether the suggestion is correct — don't apply blindly. If it's good, use it as the target behavior for your test.

**Stale comments**: If the file or lines referenced by a comment have changed significantly since the review, note this and ask the user whether the comment still applies.

**No testable behavior**: Some fix comments point to issues that are hard to capture in a test (e.g., "this log message is misleading" or "add a comment explaining why"). Apply these directly like nits.

**Non-code files**: When comments target markdown, config, or other non-executable files, TDD doesn't apply. Apply fixes directly and note why in the report. Don't force tests where there's no behavior to verify.

## References

For deeper guidance on writing good tests for PR fixes, see [references/tdd-integration.md](references/tdd-integration.md).
