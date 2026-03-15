# Models & Concerns

Read this when creating a new model, adding behavior to an existing model, choosing between concerns and service objects, or deciding between STI and delegated types.

## Table of Contents
- [Core Philosophy](#core-philosophy)
- [Concern Organization](#concern-organization)
- [Anatomy of a Concern](#anatomy-of-a-concern)
- [Concern Patterns](#concern-patterns)
- [CurrentAttributes](#currentattributes)
- [Polymorphism Patterns](#polymorphism-patterns)
- [Filter Query Object](#filter-query-object)

## Core Philosophy

All domain logic lives in models via concerns. No service objects, no interactors. Controllers call model methods directly. Service objects scatter logic across `app/services/` where it's invisible from the model — concerns keep everything discoverable via `include` lines.

The model file is a **table of contents**. Concerns are the **chapters**.

**Don't create grab-bag concerns** (e.g., `Utilities`, `Helpers`) — each concern should represent a single, named capability (`Closeable`, `Searchable`, `Taggable`). If you can't name it after what it does, it's not a concern.

```ruby
# app/models/card.rb — keep this short
class Card < ApplicationRecord
  include Assignable, Closeable, Commentable, Searchable, Taggable, Watchable

  belongs_to :board
  belongs_to :creator, class_name: "User", default: -> { Current.user }

  scope :ordered, -> { order(created_at: :desc) }
end
```

What stays in the model file:
1. Concern includes (the feature list)
2. Core associations (`belongs_to`, `has_many`)
3. Essential scopes
4. Orchestration methods that span multiple concerns

## Concern Organization

Place concerns in a **model-namespaced subdirectory**, not `app/models/concerns/`:

```
app/models/
├── card.rb
├── card/
│   ├── assignable.rb      # Card::Assignable
│   ├── closeable.rb       # Card::Closeable
│   ├── searchable.rb      # Card::Searchable
│   └── taggable.rb        # Card::Taggable
├── user.rb
├── user/
│   ├── avatar.rb           # User::Avatar
│   └── role.rb             # User::Role
```

Reserve `app/models/concerns/` for the rare concerns shared across multiple models.

## Anatomy of a Concern

Every concern follows: **associations/scopes in `included`, public methods, private methods**. One concept per concern.

```ruby
# app/models/card/closeable.rb
module Card::Closeable
  extend ActiveSupport::Concern

  included do
    has_one :closure, dependent: :destroy

    scope :closed, -> { joins(:closure) }
    scope :open, -> { where.missing(:closure) }
  end

  def closed? = closure.present?
  def open? = !closed?

  def close(user: Current.user)
    unless closed?
      transaction do
        create_closure!(user: user)
        track_event :closed, creator: user  # calls Eventable concern
      end
    end
  end

  def reopen(user: Current.user)
    if closed?
      transaction do
        closure&.destroy
        track_event :reopened, creator: user
      end
    end
  end
end
```

Guidelines:
- **One concern = one concept** — if you can't name it with a single adjective, split it
- **Concerns own their associations** — `Closeable` defines `has_one :closure`
- **Concerns call each other freely** — `close` calls `track_event` from `Eventable`; no dependency injection needed
- **Most concerns are 20-50 lines** — even complex ones rarely exceed 100 lines
- **Name with adjectives** — `-able` suffix: `Searchable`, `Closeable`, `Taggable`

## Concern Patterns

### The `_later` / `_now` Convention

Keep job-enqueuing and synchronous methods together:

```ruby
module Card::Searchable
  extend ActiveSupport::Concern

  def reindex_later
    ReindexJob.perform_later(self)
  end

  def reindex_now
    # actual indexing logic
  end
end
```

### Template Method Pattern

Shared concern defines lifecycle + hooks. Model-specific concern implements details:

```ruby
# app/models/concerns/eventable.rb — shared
module Eventable
  extend ActiveSupport::Concern

  included do
    has_many :events, as: :eventable, dependent: :destroy
  end

  def track_event(action, creator: Current.user, **particulars)
    events.create!(action: "#{eventable_prefix}_#{action}", creator:, particulars:) if should_track_event?
  end

  private
    def should_track_event? = true
    def eventable_prefix = self.class.name.demodulize.underscore
end

# app/models/card/eventable.rb — Card-specific
module Card::Eventable
  extend ActiveSupport::Concern
  include ::Eventable

  private
    def should_track_event? = published?  # override: only track published cards
end
```

### DSL Concerns

Define class methods that generate instance methods:

```ruby
# app/models/concerns/positionable.rb
module Positionable
  extend ActiveSupport::Concern

  included do
    scope :positioned, -> { order(:position_score, :id) }
  end

  class_methods do
    def positioned_within(parent, association:, filter:)
      define_method(:all_positioned_siblings) { send(parent).send(association).send(filter).positioned }
    end
  end
end

# Usage:
class Leaf < ApplicationRecord
  include Positionable
  positioned_within :book, association: :leaves, filter: :active
end
```

## CurrentAttributes

Use `ActiveSupport::CurrentAttributes` for request-scoped context:

```ruby
# app/models/current.rb — single-tenant
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user
  def account = Account.first
end

# app/models/current.rb — multi-tenant
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user, :account
end
```

Set in the Authentication concern, clear automatically at end of request.

## Polymorphism Patterns

### Delegated Types — when types need different columns

```ruby
class Leaf < ApplicationRecord
  delegated_type :leafable, types: %w[Page Section Picture], dependent: :destroy
end

module Leafable
  TYPES = %w[Page Section Picture]
  included do
    has_one :leaf, as: :leafable, touch: true
  end
end

class Page < ApplicationRecord
  include Leafable
  has_markdown :body  # Page-specific field
end
```

### STI — when types share the same schema

```ruby
class Room < ApplicationRecord
  scope :opens, -> { where(type: "Rooms::Open") }
  scope :directs, -> { where(type: "Rooms::Direct") }
end

class Rooms::Open < Room    # auto-joins new users
class Rooms::Direct < Room  # 1:1 messaging
```

Use delegated types when each type has unique columns. Use STI when they share the same table structure.

### has_secure_password

Always use `validations: false` — handle validation in forms, not the model:

```ruby
class User < ApplicationRecord
  has_secure_password validations: false
end
```

This supports passwordless users (bots, magic-link auth).

## Filter Query Object

For complex search, use a database-backed filter model instead of Ransack:

```ruby
class Filter < ApplicationRecord
  include Fields, Params, Resources, Summarized

  belongs_to :creator, class_name: "User", default: -> { Current.user }

  def self.from_params(params)
    find_by_params(params) || build(params)
  end

  def cards
    result = creator.accessible_cards.preloaded.published  # auth baked in
    result = result.where(board: boards.ids) if boards.present?
    result = result.assigned_to(assignees.ids) if assignees.present?
    result = result.tagged_with(tags.ids) if tags.present?
    # ... chain more scopes conditionally
    result.distinct
  end
end
```

Key sub-modules:
- **Fields** — `store_accessor` on JSON column with `.inquiry` for expressive checks (`indexed_by.closed?`)
- **Params** — serialize to/from URL params, digest-based deduplication
- **Resources** — HABTM associations for tags, boards, assignees; re-scope through user access
- **Summarized** — human-readable filter descriptions

Controller integration:

```ruby
module FilterScoped
  included { before_action :set_filter }

  private
    def set_filter
      if params[:filter_id].present?
        @filter = Current.user.filters.find(params[:filter_id])
      else
        @filter = Current.user.filters.from_params(filter_params)
      end
    end
end
```
