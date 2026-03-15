---
name: rails-webhooks-events
description: "Build polymorphic event systems and webhook delivery pipelines in Rails. Use when creating activity timelines, event-driven notification dispatch, webhook endpoints with HMAC signing, SSRF-safe HTTP delivery, format-aware payloads (JSON/Slack/Campfire), delinquency tracking with auto-deactivation, or fan-out job patterns with cursor-based resumption."
---

# Rails Webhooks & Events

Read this when adding activity tracking, building webhook integrations, implementing event-driven notifications, or designing fan-out delivery pipelines.

Build event-sourced activity systems and secure webhook delivery following Basecamp's Fizzy patterns.

## Quick Reference

| Component | Purpose |
|-----------|---------|
| Event model | Polymorphic record driving timelines, notifications, and webhooks |
| Eventable concern | `track_event("closed")` on any model — auto-prefixes action |
| Webhook model | Per-board subscription with `has_secure_token :signing_secret` |
| Delivery model | SSRF-safe HTTP POST with HMAC signing, size-limited response streaming |
| DelinquencyTracker | Auto-deactivate after 10+ failures over 1+ hour |
| WebhookDispatchJob | `ActiveJob::Continuable` fan-out with cursor resumption |

## Decision Guide

| Need | Component | Reference |
|------|-----------|-----------|
| Record who did what and when | Event model + Eventable concern | [references/events.md](references/events.md) |
| Show activity timeline to users | Event model with `description` | [references/events.md](references/events.md) |
| Notify users on actions | Event `after_create_commit` triggers notifications | [references/events.md](references/events.md) |
| Push data to external services | Webhook model + dispatch chain | [references/webhooks.md](references/webhooks.md) |
| Format payloads for Slack/Campfire | Format-aware payloads with regex matching | [references/webhooks.md](references/webhooks.md) |
| Auto-disable broken webhooks | DelinquencyTracker (10+ failures, 1+ hour) | [references/webhooks.md](references/webhooks.md) |
| Feed event history to LLM | Events as structured context | [references/events.md](references/events.md) |

## When To

**When adding a trackable action**: Include `Eventable` in the model and call `track_event("action_name")` in the relevant method. The concern auto-prefixes the action with the model name.

**When an event triggers notifications AND webhooks**: Let the Event model's `after_create_commit` handle both — one event record fans out to notifications and webhook dispatch. Don't wire them separately.

**When adding a new webhook format**: Add a regex pattern and payload branch to the existing format-detection logic, not a new subclass. Formats are data, not types.

**When delivery fails**: Let `DelinquencyTracker` handle it automatically — it counts failures over time and deactivates webhooks after 10+ failures spanning 1+ hour. Don't build custom retry logic.

## Don't

**Don't create events in `after_save`** — `after_save` fires inside the transaction, so dependent records (notifications, webhooks) may query stale data. Use `after_create_commit`.

**Don't let system-generated records create events** — if a job creates records that trigger events that enqueue more jobs, you get infinite loops. Guard with `should_track_event?`.

**Don't skip SSRF protection on webhook delivery** — always reject private/internal IP addresses before making HTTP requests. External URLs can resolve to internal IPs.

**Don't use a join table for action subscriptions** — a JSON array column with `LIKE` queries is simpler and avoids N+1 joins for checking subscription status.

## Reference Guide

### Event System
Read [references/events.md](references/events.md) for the Event model, Eventable concern, event descriptions, preventing infinite loops, and using events as LLM context.

### Webhook Delivery Pipeline
Read [references/webhooks.md](references/webhooks.md) for the Webhook model, dispatch chain, SSRF-safe delivery, HMAC signing, format-aware payloads, and delinquency tracking.
