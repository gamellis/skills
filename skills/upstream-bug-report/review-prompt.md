# Review prompt

Fill the `<…>` slots and send as Codex's prompt.

```text
You are reviewing <N> draft pull requests to <owner/repo> (<one line on what the project is>) before they are posted by an outside contributor. Do not modify any files; this is a read-only review.

Repo: the current directory (a clone of <owner/repo>). Base branch: `<base>`. Each PR is one commit on its own branch:
1. `<branch>`
…

Read each with `git diff <base>...<branch>`. The draft PR bodies (title + body) are in <drafts dir> (files numbered in the order above).

Test runs so far (treat as claims to check, not facts): <for each branch, the new tests' results on base and branch per platform, suite results, and pre-existing failures seen on base>.

Read the repo's contributor and style documents (<list them>) and the code around each change.

For each PR, review:
- Correctness of the fix: does it address the root cause? Edge cases it misses or breaks (nested structures, other callers of changed helpers, state the change reads before or after a mutation, <project-specific concerns>)?
- Whether the tests genuinely reproduce the bug (fail on the base branch for the right reason) and assert meaningful behaviour; brittle or flaky patterns; whether they follow the conventions of neighbouring tests and helpers.
- Style: conformance to the repo's style documents and to the surrounding code.
- PR body: accuracy of every factual claim against the code and tests (causes, what tests cover, "fails on base"); tone (should be humble, concise, and tell maintainers only what they don't already know); anything maintainers would push back on.
- Commit message accuracy.

Output: a numbered list of findings. For each: PR number, file:line (or "PR body"), severity (blocker / should-fix / nit), the concrete problem, and the evidence (quote code or describe the scenario step by step). Only report things you are reasonably confident about; say "no findings" for a PR if none. Do not pad.
```
