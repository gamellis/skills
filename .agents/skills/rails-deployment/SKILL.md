---
name: rails-deployment
description: "Deploy Rails applications with Docker, Kamal, Thruster, and Puma following Basecamp conventions. Use when writing Dockerfiles, configuring Puma, setting up Kamal deployment, adding jemalloc, configuring Thruster for HTTP/2 and auto-SSL, tuning production performance, or setting up process management with Solid Queue, Resque, or Procfiles."
---

# Rails Deployment

Read this when writing a Dockerfile, configuring Puma threads, setting up Kamal, choosing between single-process and multi-process, or optimizing production memory usage.

Deploy Rails apps with multi-stage Docker, Thruster (no nginx), and Kamal — optimized for SQLite or MySQL. The goal is a single-binary production stack: Thruster wraps Puma, which runs Solid Queue via plugin, so one process serves HTTP, handles TLS, and processes jobs.

## Quick Reference

| Component | Purpose |
|-----------|---------|
| Multi-stage Dockerfile | Small production images (~base → build → final~) |
| Thruster | HTTP/2, auto SSL, gzip — replaces nginx |
| jemalloc | 20-30% memory reduction via `LD_PRELOAD` |
| Puma | App server — tune threads to your database |
| Kamal | Zero-downtime deployment orchestration |
| Solid Queue | Database-backed jobs — runs inside Puma |
| Bootsnap | Precompile caches in Docker for fast boot |

## When To

**When starting a new app**: Default to SQLite with a single Puma process. Add Postgres/MySQL only when you need concurrent writes from multiple servers.

**When setting Puma thread count**: Use 1 thread for SQLite (it serializes writes), 3-5 threads for Postgres/MySQL. More threads than your DB can handle just increases contention.

**When TLS terminates at a load balancer**: Configure Thruster to expose only port 80 (`HTTP_PORT=80`, `TLS_DOMAIN=`). Let the load balancer handle certificates.

**When your Docker image is too large**: Check that build tools (gcc, make, node) are only in the build stage, not the final stage. Multi-stage builds should produce images under 200MB.

## Don't

**Don't add nginx** — Thruster handles HTTP/2, auto SSL, gzip, and X-Sendfile in a single binary. Adding nginx doubles your config surface for zero benefit.

**Don't skip jemalloc** — Ruby's default allocator fragments memory over time. jemalloc gives 20-30% memory savings with zero code changes via `LD_PRELOAD`.

**Don't run Solid Queue as a separate process for SQLite** — SQLite locks on writes, so a separate process would contend with Puma. Use the Puma plugin (`solid_queue`) to run it in-process.

**Don't use `after_save` to enqueue jobs** — the job may execute before the transaction commits, reading stale data. Use `after_create_commit` or `after_update_commit`.

For all configuration patterns, Dockerfile templates, Puma tuning, and process management, read [references/deployment.md](references/deployment.md).

## Key Decisions

- **SQLite**: 1 Puma thread, many workers, Solid Queue as Puma plugin (single process)
- **PostgreSQL/MySQL**: 3-5 Puma threads, separate Solid Queue or Resque process via Procfile
- **Thruster** wraps Puma for HTTP/2, auto SSL, gzip — no nginx needed
- **jemalloc** via `LD_PRELOAD` gives 20-30% memory reduction
- **Bootsnap** precompile in Docker build stage for fast production boot
