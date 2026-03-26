# TDD for PR Fixes

Applying TDD to PR review comments is different from greenfield TDD. You're not designing a new feature — you're proving a reviewer's concern is valid and then fixing it. This changes the emphasis of each phase.

## RED: The Reviewer's Concern as a Test

The reviewer already told you what's wrong. Your job is to translate their concern into a test that demonstrates the problem.

**Common translations:**

| Reviewer says | Test captures |
|---------------|---------------|
| "This doesn't handle X" | Input X → expected behavior not observed |
| "Race condition when..." | Concurrent calls → state corruption or wrong result |
| "Off-by-one" | Boundary input → wrong output |
| "Missing validation" | Invalid input → should reject but doesn't |
| "This will break if..." | The specific scenario → unexpected failure |
| "Null pointer / undefined" | Null/missing input → crash instead of graceful handling |

**Writing the test:**

```typescript
// Reviewer: "This doesn't handle expired tokens — users get a 500 instead of 401"
test("expired token returns 401, not 500", async () => {
  const expiredToken = createToken({ expiresAt: Date.now() - 1000 });
  const response = await request(app)
    .get("/api/protected")
    .set("Authorization", `Bearer ${expiredToken}`);

  expect(response.status).toBe(401);
});
```

The test name should read like a restatement of the reviewer's concern. Someone reading the test file later should understand what bug this prevents.

**Always test against the pre-fix commit.** Don't write a test against HEAD and wonder why it passes — check out the commit the reviewer was looking at. The test should fail there, proving the bug was real. Then switch back to HEAD and confirm the test passes, proving the fix works.

```bash
# Write the test, then:
git stash
git checkout <pre-fix-commit>
<run-tests>  # should FAIL — use whatever the project uses (vitest, pytest, rspec, go test, etc.)
git checkout <branch>
git stash pop
<run-tests>  # should PASS
```

This applies even when all comments are already resolved. The output in that case is a regression test suite — you're not changing code, you're proving the fixes were correct and guarding against reintroduction.

**If the test passes against the pre-fix commit**, the reviewer's concern may not have been valid, or you're testing the wrong code path. Investigate before moving on.

## GREEN: Minimal Fix

PR fixes should be surgical. The reviewer pointed at a specific problem — fix that problem and nothing else. Resist the urge to refactor the surrounding code, even if it's ugly.

For already-resolved comments, the GREEN phase is just confirming the existing fix makes the test pass. Cherry-pick or check out HEAD and run the test. If it passes, you're done — the fix works and you now have a regression test for it.

For unresolved comments, write the smallest change that makes the test pass.

Why minimal? Because:
- The PR is already in review. Large changes restart the review cycle.
- Each fix is one logical change. Reviewers can verify each one independently.
- You reduce the risk of introducing new issues while fixing old ones.

## REFACTOR: Be Conservative

In normal TDD, the refactor phase is where you improve design. In PR fixes, be more conservative:

- **Do** extract duplication if your fix introduced it
- **Do** rename things if clarity is the point of the review comment
- **Don't** restructure modules or change abstractions
- **Don't** fix things the reviewer didn't mention

The refactor scope should stay within the blast radius of the fix. If the reviewer's comment was about `validateToken()`, your refactor stays in `validateToken()` and its immediate callers.

## One Comment, One Cycle

Process comments sequentially, not in batch. Each red-green-refactor cycle is independent:

```
Comment 1: expired tokens → test → fix → refactor → done
Comment 2: race condition → test → fix → refactor → done
Comment 3: off-by-one → test → fix → refactor → done
```

This matters because:
- Fixes can interact. The race condition fix might change how you approach the off-by-one.
- Running the full suite between fixes catches interactions early.
- If a fix turns out to be more complex than expected, you can stop and ask the user without leaving other fixes half-done.

## When TDD Doesn't Fit

Not every review comment needs a test. Skip the red-green cycle for:

- **Style/naming changes**: Rename `isValid` to `isTokenValid` — no behavior change, no test needed
- **Documentation**: "Add a comment explaining why this timeout is 30s" — just add the comment
- **Dead code removal**: "This branch is unreachable" — remove it, run existing tests
- **Import cleanup**: Unused imports, reordering — no behavior change
- **Log message fixes**: "This error message is misleading" — just fix the message

The rule: if the change can't break behavior, it doesn't need a new test. But always run the existing suite to confirm.

## Test Placement

Put new tests near related existing tests. If `auth.test.ts` already has token validation tests, add your expired-token test there. If no relevant test file exists, create one following the project's naming convention.

Group the PR-fix tests naturally with the feature they test — don't create a separate "pr-fixes" test file. These tests are permanent regression tests, not temporary patches.
