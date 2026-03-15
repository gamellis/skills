# Background Jobs

Read this when adding background processing, scheduling recurring work, or deciding whether something should be a job at all.

## Table of Contents
- [Solid Queue](#solid-queue)
- [Job Design](#job-design)
- [Recurring Jobs](#recurring-jobs)
- [When Not to Use Jobs](#when-not-to-use-jobs)

## Solid Queue

Use Solid Queue (Rails 8) for background jobs. Database-backed, no Redis dependency. Jobs are rows in your database — they survive restarts, are queryable with SQL, and need no extra infrastructure.

**When Not to Use Jobs**: Don't background work that can run inline in under 100ms. The job overhead (serialize, enqueue, deserialize, execute) can exceed the work itself. Inline callbacks are simpler and more debuggable.

```ruby
gem "solid_queue"
```

Run as a Puma plugin for self-hosted (single process):

```ruby
# config/puma.rb
plugin :solid_queue
```

Add Mission Control for monitoring:

```ruby
gem "mission_control-jobs"

# config/routes.rb
namespace :admin do
  mount MissionControl::Jobs::Engine, at: "/jobs"
end
```

## Job Design

### The `_later` / `_now` Convention

Keep job-enqueuing and synchronous logic together in the model concern:

```ruby
# app/models/event/relaying.rb
module Event::Relaying
  extend ActiveSupport::Concern

  included do
    after_create_commit :relay_later
  end

  def relay_later
    Event::RelayJob.perform_later(self)
  end

  def relay_now
    # actual work
  end
end

# app/jobs/event/relay_job.rb
class Event::RelayJob < ApplicationJob
  def perform(event)
    event.relay_now
  end
end
```

The convention:
1. `_later` enqueues the job
2. `_now` contains the actual logic
3. The job class is a thin wrapper delegating to the model

### SMTP Error Handling

For mailer jobs, handle errors by category:

```ruby
# app/jobs/concerns/smtp_delivery_error_handling.rb
module SmtpDeliveryErrorHandling
  extend ActiveSupport::Concern

  included do
    retry_on Net::OpenTimeout, Net::ReadTimeout, wait: :polynomially_longer
    retry_on Net::SMTPServerBusy, wait: :polynomially_longer

    rescue_from Net::SMTPSyntaxError do |error|
      # Invalid address — log, don't retry
      Rails.logger.warn "SMTP syntax error: #{error.message}"
    end

    rescue_from Net::SMTPFatalError do |error|
      # Unknown user / oversized — log, don't retry
      Rails.logger.warn "SMTP fatal error: #{error.message}"
    end
  end
end
```

For multi-tenant job context propagation (serializing `Current.account` via GlobalID), use the **rails-multi-tenancy** skill.

## Recurring Jobs

Use `config/recurring.yml` for scheduled tasks:

```yaml
# config/recurring.yml
production:
  deliver_bundled_notifications:
    command: "Notification::Bundle.deliver_all_later"
    schedule: every 30 minutes

  cleanup_expired_sessions:
    command: "Session.where('last_active_at < ?', 30.days.ago).delete_all"
    schedule: every day at 04:00

  auto_postpone_stale:
    command: "Card.auto_postpone_all_due"
    schedule: every hour at minute 50
```

Two styles:
- `command:` — arbitrary Ruby expression (no job class needed)
- `class:` — enqueues a specific job class

## When Not to Use Jobs

Don't add jobs you don't need. If the app is simple and single-user, run everything synchronously. Only move work to background jobs when:

- The operation involves external I/O (email, webhooks, push notifications)
- The operation is slow and the user shouldn't wait
- The operation can be batched (notification digests)
