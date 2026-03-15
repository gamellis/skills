---
name: basecamp-rails
description: "Build Rails applications following Basecamp/37signals conventions: importmap-rails, Propshaft, Hotwire (Turbo + Stimulus), hand-rolled authentication, concerns over service objects, SQLite default, Solid Queue, Tailwind CSS v4. Use when building new Rails applications, adding features to Rails apps, setting up Rails project structure, implementing authentication, implementing authorization and access control, creating models with concerns, adding real-time features, writing tests, or any Rails development task where the user wants to follow modern Rails conventions."
---

# Basecamp Rails

Read this when building a new Rails app, adding features to an existing one, or choosing between libraries and patterns for a Rails project.

Build Rails applications the Basecamp way — server-rendered HTML with Hotwire for interactivity, concerns over service objects, SQLite by default, and no unnecessary dependencies.

## Core Philosophy

1. **No JavaScript framework** — Turbo + Stimulus + ActionCable handle everything. SPA frameworks add build complexity, bundle size, and state sync bugs that server-rendered HTML avoids entirely.
2. **No JS bundler** — importmap-rails loads ES modules directly. This eliminates build steps, lock files, and `node_modules`, so deploying JS is as simple as adding a pin.
3. **Tailwind CSS v4** — CSS-first config via `@theme` for design tokens, utility classes in templates, `@utility` for component abstractions, `dark:` variant for dark mode. No Sass, no PostCSS config files, no `tailwind.config.js` — all configuration lives in CSS.
4. **No auth gem** — hand-rolled Authentication concern (~60 lines). Devise is 5,000+ lines of opaque middleware; hand-rolled auth is fully understood and trivially customized.
5. **No service objects** — domain logic lives in model concerns. Concerns keep logic discoverable on the model itself — scanning `include` lines reveals capabilities without hunting through `app/services/`.
6. **No factory gem** — Minitest with YAML fixtures. Fixtures are deterministic, bulk-inserted once per suite, and load in milliseconds — no per-test object creation overhead.
7. **SQLite first** — switch to Postgres/MySQL only for multi-tenant SaaS. SQLite needs no server process, no connection pooling, and no port conflicts — one less thing to configure in development and production.
8. **Solid Queue** — database-backed jobs, no Redis dependency. Jobs survive restarts as table rows, and the queue is queryable with standard SQL.
9. **Thruster** — HTTP/2, auto SSL, gzip; no nginx needed. One binary replaces nginx config, SSL certs, and gzip middleware — fewer moving parts in production.
10. **Concerns are chapters** — the model file is a table of contents. Scanning `include` lines reveals capabilities at a glance; each concern file reads top-to-bottom as a complete feature.

## Decision Guide

| Situation | Approach | Reference |
|-----------|----------|-----------|
| Adding a model feature | Namespaced concern (e.g., `Card::Closeable`) | [references/models.md](references/models.md) |
| State change (close, pin, archive) | Singleton resource controller (`resource :closure`) | [references/controllers.md](references/controllers.md) |
| Real-time updates | `broadcasts_refreshes` or Turbo Stream | [references/realtime.md](references/realtime.md) |
| Interactive UI component | Stimulus controller or Tailwind Plus Element | [references/frontend.md](references/frontend.md) |
| Background work | Solid Queue job with `_later`/`_now` naming | [references/background-jobs.md](references/background-jobs.md) |
| User asks for React/Vue/Angular | Hotwire equivalent (Turbo Frames, Streams, Stimulus) | [references/views-hotwire.md](references/views-hotwire.md) |
| Dialog, dropdown, tabs | Tailwind Plus Elements web component | [references/tailwind-elements.md](references/tailwind-elements.md) |
| Database choice for new app | SQLite unless multi-tenant SaaS | [references/database.md](references/database.md) |

## When To

**When adding a model feature**: Create a namespaced concern (e.g., `Card::Closeable`) and `include` it in the model. The model file stays a readable table of contents.

**When a controller needs complex logic**: Move it to a model method or concern, not a service object. Controllers call model methods; models own the domain.

**When choosing delegated types vs STI**: If subtypes have different columns, use delegated types. If all subtypes share the same schema, use STI.

**When adding JS behavior**: Write a Stimulus controller, not an inline `<script>`. Stimulus controllers are reusable, testable, and auto-connected via `data-controller`.

**When a user asks for React/Vue**: Explain the Hotwire equivalent — Turbo Frames for lazy-loading, Turbo Streams for live updates, Stimulus for sprinkles of interactivity.

**When setting up a new model**: Write the model file as a table of contents (`include` lines, associations, validations), then create each concern as a separate file.

## Don't

**Don't create `app/services/`** — service objects scatter domain logic across files. Put it in a model concern where it's discoverable.

**Don't use inline `current_user.admin?` checks** — scope queries to the current user's permissions (e.g., `Current.account.boards`) so authorization is impossible to forget.

**Don't add Devise or Warden** — hand-rolled auth is ~60 lines, fully understood, and avoids 5,000+ lines of opaque middleware.

**Don't use `after_save` for broadcasting** — `after_save` fires inside the transaction, so subscribers may read stale data. Use `after_create_commit` / `after_update_commit`.

**Don't use Sass or PostCSS** — Tailwind v4's CSS-first config (`@theme`, `@utility`) handles design tokens, component classes, and dark mode without a preprocessor.

## Reference Guide

Load the relevant reference file based on what you're working on:

### Setting Up a New App
Read [references/project-setup.md](references/project-setup.md) for Gemfile and application config. For deployment (Docker, Puma, Kamal), use the **rails-deployment** skill.

### Models & Domain Logic
Read [references/models.md](references/models.md) for concern organization, naming conventions, the `_later`/`_now` pattern, template methods, CurrentAttributes, delegated types vs STI.

### Controllers & Routing
Read [references/controllers.md](references/controllers.md) for thin controllers, the Authentication concern, CRUD-oriented routing, scoping concerns, `params.expect`, rate limiting.

### Views & Hotwire
Read [references/views-hotwire.md](references/views-hotwire.md) for Turbo Streams, Stimulus controller patterns, custom stream actions, fragment caching, JS organization.

### Database
Read [references/database.md](references/database.md) for SQLite, FTS5 full-text search, float position scores, schema conventions, primary key strategy.

### Background Jobs
Read [references/background-jobs.md](references/background-jobs.md) for Solid Queue setup, the `_later`/`_now` convention, recurring jobs, SMTP error handling.

### Testing
Read [references/testing.md](references/testing.md) for Minitest, fixtures, parallel execution, multi-user system tests, test helpers.

### Real-Time Features
Read [references/realtime.md](references/realtime.md) for Turbo broadcasts, ActionCable channels, presence tracking, Solid Cable, connection authentication.

### Tailwind CSS & Frontend
Read [references/frontend.md](references/frontend.md) for Tailwind v4 setup, `@theme` design tokens, `@utility` component classes, dark mode, `cn()` helper, asset pipeline integration.

### Tailwind Plus Elements
Read [references/tailwind-elements.md](references/tailwind-elements.md) for interactive UI components (dialogs, dropdowns, tabs, disclosures, selects) using web components with importmap — no JavaScript framework needed.

### Mailers & Notifications
Read [references/mailers.md](references/mailers.md) for notification bundling, web push, in-app notifications via Turbo Streams, join codes.

### Authorization & Access Control
Read [references/authorization.md](references/authorization.md) for join-model access, role enums with creator fallback, query-scoped authorization, involvement levels, revocation cleanup, notification routing, and filter query objects.

### Rich Text & Mentions
Read [references/rich-content.md](references/rich-content.md) for ActionText, @mention system, Markdown alternative with `has_markdown`.

## Quick Patterns

### Model with Concerns

```ruby
# app/models/card.rb
class Card < ApplicationRecord
  include Assignable, Closeable, Searchable, Taggable

  belongs_to :board
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end

# app/models/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy
    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  def close(user: Current.user)
    create_closure!(user:) unless closed?
  end
end
```

### CRUD Resource Controller

```ruby
# config/routes.rb
resources :cards do
  scope module: :cards do
    resource :closure    # create = close, destroy = reopen
    resource :pin
    resources :comments
  end
end

# app/controllers/cards/closures_controller.rb
class Cards::ClosuresController < ApplicationController
  include BoardScoped, CardScoped

  def create = @card.close
  def destroy = @card.reopen
end
```

### Turbo Stream Broadcast

```ruby
module Card::Broadcastable
  extend ActiveSupport::Concern
  included { broadcasts_refreshes }
end
```

### Current Attributes

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user
  def account = Account.first  # single-tenant
end
```
