---
name: rails-multi-tenancy
description: "Implement URL-path multi-tenancy in Rails applications. Use when adding tenant isolation via SCRIPT_NAME middleware, propagating CurrentAttributes through background jobs with GlobalID, building dual-mode (SaaS/self-hosted) apps, configuring read replicas with automatic failover, implementing two-phase account deletion, sharded full-text search, or layering SaaS features via a Rails engine."
---

# Rails Multi-Tenancy

Read this when adding multi-tenant isolation, propagating tenant context through jobs, building dual-mode (SaaS + self-hosted) apps, or implementing account lifecycle management.

URL-path multi-tenancy using SCRIPT_NAME rewriting — tenant context propagates through routes, jobs, and search with zero route changes.

## Quick Reference

| Component | Purpose |
|-----------|---------|
| AccountSlug::Extractor | Rack middleware moves tenant slug to SCRIPT_NAME |
| CurrentAttributes | `Current.account` cascades to resolve per-tenant user |
| Job propagation | Serialize `Current.account` as GlobalID, restore before execution |
| MultiTenantable | `cattr_accessor :multi_tenant` gates single vs multi-tenant mode |
| Cancellable + Incineratable | Two-phase deletion with 30-day grace period |
| Search sharding | CRC32 hashing maps accounts to 16 MySQL FULLTEXT tables |
| SaaS engine | `config.to_prepare` injects billing/limits into base classes |

## Decision Guide

| Question | Answer | Reference Section |
|----------|--------|-------------------|
| Subdomain or path tenancy? | Path — SCRIPT_NAME works behind any proxy and needs no DNS | SCRIPT_NAME Middleware |
| How to scope queries? | Always go through `Current.account` associations | CurrentAttributes |
| How to pass tenant to jobs? | Serialize as GlobalID, restore in `before_perform` | Job Propagation |
| Single-tenant or multi-tenant? | `cattr_accessor :multi_tenant` — one flag gates everything | Multi-Tenant Toggle |
| How to delete an account? | Cancel (30-day grace) → Incinerate (recurring job) | Account Lifecycle |
| How to search across tenants? | CRC32 shard mapping for MySQL; FTS5 for SQLite | Search Sharding |
| How to add billing/limits? | SaaS engine — `config.to_prepare` injects into base classes | SaaS Engine |

## When To

**When adding a new scoped resource**: Always access it through `Current.account` associations (e.g., `Current.account.boards.find(params[:id])`). Never query the model directly.

**When serializing tenant in a job**: Use `GlobalID` to serialize `Current.account`, not a plain integer ID. GlobalID handles lookup and restores the full object.

**When choosing single vs multi-tenant**: Use `MultiTenantable` with `cattr_accessor :multi_tenant`. The SaaS engine flips the flag; self-hosted stays single-tenant.

**When deleting an account**: Two phases — cancel sets a 30-day grace period (user can undo), then a recurring job incinerates expired accounts permanently.

## Don't

**Don't use subdomain-based tenancy** — SCRIPT_NAME path rewriting works behind any reverse proxy, needs no wildcard DNS, and requires zero route changes.

**Don't query models without tenant scoping** — `Board.find(id)` leaks data across tenants. Always scope through `Current.account`.

**Don't serialize tenant as a plain ID in jobs** — use GlobalID so the object is restored correctly. Plain IDs require manual lookup and miss type checking.

**Don't allow account cancellation in single-tenant mode** — there's only one account. Guard cancellation with `multi_tenant?`.

For all patterns and code examples, read [references/multi-tenancy.md](references/multi-tenancy.md).

## Architecture Overview

- **SCRIPT_NAME middleware** — moves tenant slug from PATH_INFO to SCRIPT_NAME so all route helpers auto-include the tenant prefix. Zero route changes needed. `/12345/boards/1` → SCRIPT_NAME=`/12345`, PATH_INFO=`/boards/1`. This approach works behind any reverse proxy (nginx, Thruster, load balancers) because SCRIPT_NAME is a standard Rack convention.
- **Job context propagation** — prepend `TenantJobExtensions` on ActiveJob to serialize `Current.account` as GlobalID at enqueue, restore before execution. This ensures jobs always run in the correct tenant context without manual passing.
- **Dual-mode** — `cattr_accessor :multi_tenant` gates single vs SaaS mode. Self-hosted allows one account; SaaS engine flips the flag. Feature code checks `multi_tenant?` instead of environment variables.
- **Two-phase deletion** — Cancel (30-day grace period) → Incinerate (recurring job). SaaS engine hooks: cancel pauses Stripe, incinerate cancels subscription.
- **Search sharding** — CRC32 hashing maps accounts to 16 MySQL FULLTEXT tables; SQLite mode uses FTS5 directly
- **Read replicas** — automatic failover with GTID-based transaction pinning for read-after-write consistency
