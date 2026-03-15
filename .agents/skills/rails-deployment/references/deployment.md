# Deployment Patterns — Detailed Reference

Read this when writing a Dockerfile, tuning Puma, configuring Kamal, or deciding between single-process and multi-process deployment.

**When choosing single vs multi-process**: SQLite requires single-process (Puma + Solid Queue plugin) because it locks on writes. Postgres/MySQL support multi-process (separate Solid Queue or Resque workers via Procfile).

**When setting Puma threads**: Match your database — 1 thread for SQLite (serialized writes), 3-5 for Postgres/MySQL. More threads than the DB pool just increases lock contention.

## Table of Contents
- [Multi-Stage Docker Build](#multi-stage-docker-build)
- [jemalloc](#jemalloc)
- [Thruster](#thruster)
- [Puma Configuration](#puma-configuration)
- [Process Management](#process-management)
- [Kamal](#kamal)
- [Bootsnap Precompilation](#bootsnap-precompilation)
- [Container Security](#container-security)
- [Version Tracking](#version-tracking)

## Multi-Stage Docker Build

Three stages: `base` (runtime deps), `build` (compile), `final` (production image).

### Base Stage

Install only runtime dependencies — no compilers:

```dockerfile
ARG RUBY_VERSION=3.4.7
FROM docker.io/library/ruby:$RUBY_VERSION-slim AS base
WORKDIR /rails

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y curl libjemalloc2 libvips sqlite3 libssl-dev && \
    ln -s /usr/lib/$(uname -m)-linux-gnu/libjemalloc.so.2 /usr/local/lib/libjemalloc.so

ENV RAILS_ENV="production" \
    BUNDLE_DEPLOYMENT="1" \
    BUNDLE_PATH="/usr/local/bundle" \
    BUNDLE_WITHOUT="development:test" \
    LD_PRELOAD="/usr/local/lib/libjemalloc.so"
```

Common runtime packages:
- `libjemalloc2` — memory allocator
- `libvips` — image processing (Active Storage variants)
- `sqlite3` — database (self-hosted)
- `ffmpeg` — video processing (if needed)
- `libssl-dev` — TLS support

### Build Stage

Install compilers, build gems, precompile assets:

```dockerfile
FROM base AS build
RUN apt-get install --no-install-recommends -y build-essential git libyaml-dev pkg-config
COPY Gemfile Gemfile.lock vendor ./
RUN bundle install && \
    bundle exec bootsnap precompile -j 1 --gemfile
COPY . .
RUN bundle exec bootsnap precompile -j 1 app/ lib/
RUN SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile
```

`SECRET_KEY_BASE_DUMMY=1` generates a throwaway key for asset precompilation — the real key is set at runtime.

The `-j 1` flag on bootsnap disables parallel compilation, working around a QEMU bug in cross-platform Docker builds.

### Final Stage

Copy only what's needed from the build stage:

```dockerfile
FROM base
RUN groupadd --system --gid 1000 rails && \
    useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash
USER 1000:1000
COPY --chown=rails:rails --from=build "${BUNDLE_PATH}" "${BUNDLE_PATH}"
COPY --chown=rails:rails --from=build /rails /rails
EXPOSE 80 443
CMD ["./bin/thrust", "./bin/rails", "server"]
```

Build tools (`gcc`, `make`, `git`) are not in the final image.

## jemalloc

jemalloc is a drop-in malloc replacement that reduces memory fragmentation in long-running Ruby processes (typically 20-30% reduction).

```dockerfile
# Install
RUN apt-get install --no-install-recommends -y libjemalloc2

# Create symlink
RUN ln -s /usr/lib/$(uname -m)-linux-gnu/libjemalloc.so.2 /usr/local/lib/libjemalloc.so

# Enable via LD_PRELOAD
ENV LD_PRELOAD="/usr/local/lib/libjemalloc.so"
```

The `$(uname -m)` in the symlink handles both `x86_64` and `aarch64` architectures.

## Thruster

Thruster (`thrust`) is an HTTP proxy that sits in front of Puma, replacing nginx:

```dockerfile
CMD ["./bin/thrust", "./bin/rails", "server"]
```

Or in a Procfile:
```
web: bundle exec thrust bin/start-app
```

Provides:
- **HTTP/2** — multiplexed connections
- **Auto SSL** — Let's Encrypt certificates, automatic renewal
- **gzip compression** — no need for `Rack::Deflater`
- **Static file serving** — serves `public/` directly
- **X-Sendfile** — efficient file downloads

Expose both ports for direct TLS termination:
```dockerfile
EXPOSE 80 443
```

If TLS is handled upstream (load balancer), expose only port 80.

## Puma Configuration

### SQLite — Fewer Threads, More Workers

SQLite has no network I/O wait, so threads provide less benefit. Use 1 thread per worker with more workers:

```ruby
# config/puma.rb
if !Rails.env.local?
  workers Integer(ENV.fetch("WEB_CONCURRENCY") { Concurrent.physical_processor_count })
  threads 1, 1

  before_fork do
    Process.warmup  # GC, compact, free empty pages
  end

  before_worker_boot do
    GC.config(rgengc_allow_full_mark: false)  # Defer major GC
  end

  out_of_band do
    GC.start if GC.latest_gc_info(:need_major_by)  # GC between requests
  end
end
```

GC strategy: defer major GC during request handling, run it out-of-band between requests. `Process.warmup` before fork compacts memory for better copy-on-write sharing.

### Network Database — More Threads

PostgreSQL/MySQL have network I/O waits where threads can do useful work:

```ruby
# config/puma.rb
worker_count = (Concurrent.processor_count * 0.666).ceil
workers ENV.fetch("WEB_CONCURRENCY") { worker_count }
threads 5, 5
```

### Debug Signal Handler

Add a signal handler for production debugging:

```ruby
Signal.trap :SIGPROF do
  Thread.list.each { |t| puts t; puts t.backtrace; puts }
end
```

Send `kill -PROF <pid>` to dump all thread backtraces.

### preload_app!

Safe when the app doesn't spawn background threads before fork:

```ruby
preload_app!  # Workers share memory via copy-on-write
```

## Process Management

### Single-Process: Puma + Solid Queue

For SQLite-based apps, run everything in one process:

```ruby
# config/puma.rb
plugin :solid_queue
```

No Procfile, no Redis, no separate worker process. Puma handles HTTP and background jobs.

Guard for SaaS vs self-hosted:
```ruby
unless MyApp.saas? || ENV["SOLID_QUEUE_IN_PUMA"] == "false"
  plugin :solid_queue
end
```

### Multi-Process: Procfile

For apps needing Redis (Action Cable, Resque):

```
# Procfile
web: bundle exec thrust bin/start-app
redis: redis-server config/redis.conf
workers: FORK_PER_JOB=false INTERVAL=0.1 bundle exec resque-pool
```

Three processes: HTTP server, Redis, and background workers.

## Kamal

Kamal handles building, pushing, and deploying Docker images:

```ruby
# Gemfile
gem "kamal", require: false
```

Provides:
- Building and pushing Docker images
- Rolling deployments across servers
- Zero-downtime deploys with Thruster health checks
- Secret management via environment variables

## Bootsnap Precompilation

Pre-generate ISeq and YAML caches in the Docker build:

```dockerfile
RUN bundle exec bootsnap precompile -j 1 --gemfile
RUN bundle exec bootsnap precompile -j 1 app/ lib/
```

First command caches gems, second caches application code. This speeds up boot time significantly in production containers.

## Container Security

### Non-Root User

Always run as a non-root user:

```dockerfile
RUN groupadd --system --gid 1000 rails && \
    useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash
USER 1000:1000
COPY --chown=rails:rails --from=build "${BUNDLE_PATH}" "${BUNDLE_PATH}"
COPY --chown=rails:rails --from=build /rails /rails
```

UID/GID 1000 is conventional. All copied files must be `--chown`ed to the rails user.

## Version Tracking

Embed version info at build time:

```dockerfile
ARG APP_VERSION
ENV APP_VERSION=$APP_VERSION
ARG GIT_REVISION
ENV GIT_REVISION=$GIT_REVISION
```

Build with:
```bash
docker build --build-arg APP_VERSION=1.2.3 --build-arg GIT_REVISION=$(git rev-parse HEAD) .
```

Available at runtime for health checks and logging.
