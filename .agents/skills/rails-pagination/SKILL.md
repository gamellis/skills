---
name: rails-pagination
description: "Implement pagination in Rails applications using either Kaminari (traditional page-number navigation) or Basecamp-style patterns (infinite scroll with geared_pagination, cursor-based pagination for real-time feeds). Use when adding pagination to index pages, implementing infinite scroll, building cursor-based feeds, paginating chat messages, adding 'load more' buttons, or choosing between offset-based and cursor-based pagination strategies."
---

# Rails Pagination

Read this when adding pagination to a list view, choosing between page numbers and infinite scroll, or building a real-time feed with stable ordering.

Two proven approaches to pagination in Rails — choose based on your UX needs. Default to `geared_pagination` when unsure — its accelerating page sizes (15 → 30 → 50 → 100) give fast initial loads and it works out of the box with Turbo Frames.

## Decision Guide

| UX Need | Approach | Reference |
|---------|----------|-----------|
| Admin panel, numbered page links | Kaminari | [references/kaminari.md](references/kaminari.md) |
| Infinite scroll / "Load more" | Geared Pagination + Turbo | [references/geared-pagination.md](references/geared-pagination.md) |
| Real-time feed (chat, activity) | Hand-rolled cursor pagination | [references/cursor-pagination.md](references/cursor-pagination.md) |
| Very simple (< 100 records) | `.limit(N)` — no gem needed | N/A |

## Don't

**Don't use offset pagination for real-time feeds** — new records shift all offsets, causing duplicates and skipped items. Use cursor-based pagination (`WHERE created_at < ?`) for stable ordering.

**Don't run `SELECT COUNT(*)` on large tables** — it's a full table scan on most databases. Use `geared_pagination` (no count needed) or Kaminari's `without_count` mode.

**Don't add a pagination gem for <100 records** — a simple `.limit(N)` or `.last(N)` is clearer and avoids the dependency.

**Don't mix pagination approaches in one view** — pick one strategy per endpoint. Mixing offset and cursor pagination in the same list creates confusing edge cases.

## Approach Details

### Kaminari — Numbered Pages
Traditional offset-based gem with `page(N).per(25)` and view helpers for numbered links. Best for admin panels and SEO-friendly CRUD. See [references/kaminari.md](references/kaminari.md).

### Geared Pagination — Infinite Scroll
Basecamp's gem with accelerating page sizes (15 → 30 → 50 → 100) for fast initial loads. Built-in Turbo Frame helpers. No COUNT queries. See [references/geared-pagination.md](references/geared-pagination.md).

### Cursor-Based — Real-Time Feeds
Hand-rolled model concern using `created_at` cursors for stable pagination during live inserts. Bidirectional loading with `page_before`/`page_after` scopes. See [references/cursor-pagination.md](references/cursor-pagination.md).
