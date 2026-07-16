# Example Pitches

Two pitches at different scales, written against fictional codebases. Study these for tone, level of detail, and how each ingredient works — the projects are made up, but the shape is what a real pitch should look like.

---

## Example 1: Small Pitch — `stagehand doctor` (Preflight Validation)

*Context: `stagehand` is a CLI that deploys a static site to a hosting provider. It has an `init` command that writes a config file and stores API tokens in the system keychain.*

### Problem

After running `stagehand init`, the user has no way to verify everything is wired up correctly. Tokens could be expired, the config could point at a deleted project, the build command could be missing. Without a diagnostic tool, the user won't know until a deploy fails halfway through — usually while they're trying to ship something.

### Appetite

Small. A single command that runs a checklist of validation steps. No interactivity, no fixing — just clear pass/fail with actionable suggestions.

### Solution

A `stagehand doctor` command that validates the full stack from prerequisites to service connections. Runs standalone — does not require a deploy in progress.

**Check list:**

```
$ stagehand doctor

Prerequisites
  ✓ node v22.1.0
  ✓ git 2.45.0

Config
  ✓ ./stagehand.json exists and valid
  ✓ Build command resolves: "npm run build"
  ✓ Output directory exists: ./dist

Host
  ✓ API token valid (from keychain)
  ✓ Project "marketing-site" exists and is writable
  ✓ Custom domain DNS points at host

Ready! Run "stagehand deploy" to ship.
```

**Failure output** — each failure includes a one-line fix suggestion:

```
  ✗ API token expired
    → Run "stagehand init" to re-authorize

  ✗ Output directory ./dist not found
    → Run your build first, or fix "outputDir" in stagehand.json
```

**How each check works:**
- Prerequisites: shell out to `which` + `--version`
- Config: read and validate JSON against the existing `ConfigSchema` in `src/config.ts`
- Build command: resolve the binary on `PATH`, don't execute it
- Token and project: single API call each, reusing the `HostClient` in `src/host/client.ts`

**Exit code:** `0` if all pass, `1` if any fail. Useful for scripting (`stagehand doctor && stagehand deploy`).

### Rabbit Holes

- **Partial success** — don't bail on first failure. Run all checks and report everything at once so the user can fix multiple issues in one pass.
- **Network timeouts** — keep HTTP checks fast (2-second timeout). Don't let a slow host hang the whole doctor run.
- **DNS checking** — resolving a custom domain can be slow and flaky on some networks. Do one lookup with a short timeout and report "couldn't verify" rather than failing outright.
- **Token introspection** — the host API doesn't expose token scopes. Don't try to infer them; just verify the token works by making a real call.

### No-Gos

- Don't fix problems automatically — report and suggest
- Don't run the build to check it (that's the user's time, and doctor should be fast)
- Don't check per-deploy state like build cache freshness — that's beyond doctor's scope
- Don't cache results — always check live state

---

## Example 2: Medium Pitch — Weekly Digest Email

*Context: "Beacon" is a team issue tracker. It has `User`, `Project`, and `Issue` models, an existing `Notification` mailer, and a background job runner already used for `IssueReindexJob`.*

### Problem

Beacon only emails you when something is addressed to you directly — an assignment or an @-mention. Anything else, you find out by opening the app. Project leads have told us they open Beacon on Monday morning purely to reconstruct what moved last week, clicking through five or six projects to piece it together. Two teams have built their own scraper scripts to do this. When a lead is on vacation, they come back to no idea what happened and no way to catch up except scrolling.

### Appetite

Medium. This touches a new job, a new mailer template, a preferences surface, and a digest-window query across issues — but it's all built on parts that already exist. If it starts requiring a new aggregation table or per-user scheduling infrastructure, we've overshot and should cut scope.

### Solution

A weekly email, per project, summarizing what changed.

**Breadboard:**

- **Project settings page** gets a "Weekly digest" section: a toggle (default off), a day-of-week picker, and a "Send me a preview" affordance. Writes to a new `digest_enabled` / `digest_day` pair on the existing `ProjectMembership` model — a preference per member, not per project, so two leads on the same project can differ.
- **`DigestJob`** runs daily, finds memberships whose `digest_day` is today, and enqueues one `DigestMailer` per membership. Follows the same enqueue pattern as `IssueReindexJob`.
- **The email** contains, scoped to the last 7 days: issues opened, issues closed, issues that changed status, and comment count. Each line links back into the app. If nothing changed, no email goes out — silence is better than "0 issues this week."
- **Preview** renders the same mailer against the last 7 days and sends only to the requester, so a lead can see the format before committing their team to it.

**What it doesn't do:** It doesn't summarize *content* — no AI-written prose, no "here's what mattered." It's a list of what moved, and the reader decides what matters. It also doesn't batch across projects: a lead on four digest-enabled projects gets four emails. Cross-project rollup is a different problem with different design questions.

### Rabbit Holes

- **Timezone for "day"** — users span timezones and `digest_day` is ambiguous without one. Use the timezone already on `User#timezone`; if it's null, fall back to the account's. Don't add a new timezone field or a per-digest send-time picker.
- **The digest-window query** — naively this is a full scan of issues per project per member. Scope it with the existing `issues.updated_at` index and reuse `Project#recent_activity`, which already does most of this for the activity feed. If that query can't be reused, cut the comment count before you add a new index.
- **Unsubscribe compliance** — this is bulk-ish mail and needs a working one-click unsubscribe. The `Notification` mailer already has an unsubscribe token flow. Reuse it rather than inventing digest-specific tokens.
- **Duplicate sends on retry** — the job runner retries on failure, and a retry after a partial batch would double-send. Record a `last_sent_on` date per membership and skip anything already sent today.

### No-Gos

- No digest for personal activity ("your week") — this is project-scoped only
- No Slack, webhook, or in-app delivery — email only
- No custom digest windows (daily, biweekly, monthly). One cadence: weekly
- No admin surface to force-enable digests across a team. It's opt-in, per member

---

## What to notice

Both pitches share patterns worth emulating:

1. **Problem tells a specific story** — not "we need X" but "here's what breaks and why it's unacceptable"
2. **Appetite is a design constraint** — the tier isn't just a label, it's followed by *why* that scope is right, and by what would signal an overshoot
3. **Solution is breadboard-level** — describes topology (what connects to what) without wireframes or code
4. **Solution names real code** — existing models, jobs, and patterns the work builds on, so the builder starts from what's there
5. **"What it doesn't do" sections** — the solution explicitly names what it excludes, reinforcing boundaries from the inside
6. **Rabbit holes are specific and resolved** — each one names the trap AND the compromise
7. **No-gos are firm exclusions** — not "nice to haves we'll skip" but "we are deliberately not doing this"
