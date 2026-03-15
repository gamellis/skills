# Authorization & Access Control

Read this when implementing access control, defining user roles, or gating features by membership.

**Don't use Pundit or CanCanCan** — a join model (`Membership`, `Access`) with role enums is simpler, queryable, and doesn't require learning a policy DSL.

## Table of Contents
- [Join Model Pattern](#join-model-pattern)
- [Role System](#role-system)
- [Resource-Level Access Concern](#resource-level-access-concern)
- [Controller Authorization](#controller-authorization)
- [Involvement & Watching](#involvement--watching)
- [Data Cleanup on Revocation](#data-cleanup-on-revocation)

## Join Model Pattern

Gate resource access through a join table. No Pundit, no CanCanCan — a record in the database.

```ruby
# app/models/access.rb
class Access < ApplicationRecord
  belongs_to :board, touch: true
  belongs_to :user, touch: true

  enum :involvement, %i[ access_only watching ].index_by(&:itself), default: :access_only

  after_destroy_commit :clean_inaccessible_data_later
end
```

The `involvement` enum separates "can access" from "wants notifications". For simpler apps, use a `level` enum instead:

```ruby
class Access < ApplicationRecord
  enum :level, %w[ reader editor ].index_by(&:itself)
  belongs_to :user
  belongs_to :book
end
```

## Role System

Define account-wide roles as enums on `User` in a `User::Role` concern:

```ruby
# app/models/user/role.rb
module User::Role
  extend ActiveSupport::Concern

  included do
    enum :role, %i[ owner admin member system ].index_by(&:itself), scopes: false
    scope :admin, -> { where(active: true, role: %i[ owner admin ]) }

    def admin?
      super || owner?
    end
  end

  def can_administer_board?(board)
    admin? || board.creator == self
  end

  def can_administer_card?(card)
    admin? || card.creator == self
  end
end
```

Key patterns:
- Higher roles are supersets — override `admin?` to include `owner?`
- **Creator fallback** — the person who made a resource can always manage it
- Use `scopes: false` to define custom role scopes

For simpler apps, a single method suffices:

```ruby
def can_administer?(record = nil)
  administrator? || self == record&.creator || record&.new_record?
end
```

## Resource-Level Access Concern

Add an `Accessible` concern to the primary resource with `grant_to`/`revoke_from` as association extensions:

```ruby
# app/models/board/accessible.rb
module Board::Accessible
  extend ActiveSupport::Concern

  included do
    has_many :accesses, dependent: :delete_all do
      def revise(granted: [], revoked: [])
        transaction do
          grant_to granted
          revoke_from revoked
        end
      end

      def grant_to(users)
        Access.insert_all Array(users).collect { |user|
          { board_id: proxy_association.owner.id, user_id: user.id,
            account_id: proxy_association.owner.account.id }
        }
      end

      def revoke_from(users)
        destroy_by user: users unless proxy_association.owner.all_access?
      end
    end

    has_many :users, through: :accesses
    scope :all_access, -> { where(all_access: true) }

    after_create :grant_access_to_creator
    after_save_commit :grant_access_to_everyone
  end

  def accessible_to?(user)
    accesses.exists?(user: user)
  end

  def watchers
    users.active.where(accesses: { involvement: :watching })
  end

  private
    def grant_access_to_creator
      accesses.create(user: creator, involvement: :watching)
    end

    def grant_access_to_everyone
      accesses.grant_to(account.users.active) if all_access_previously_changed?(to: true)
    end
end
```

Use `insert_all` for bulk grants — no callbacks needed for idempotent access operations.

For book-style apps with reader/editor levels, use `upsert_all`:

```ruby
def update_access(editors:, readers:)
  editors = Set.new(editors)
  readers = Set.new(everyone_access? ? User.active.ids : readers)
  all = editors + readers

  accesses.upsert_all(all.collect { |user_id|
    { user_id: user_id, level: editors.include?(user_id) ? :editor : :reader }
  }, unique_by: [ :book_id, :user_id ])
  accesses.where.not(user_id: all).delete_all
end
```

## Controller Authorization

Two complementary patterns: **query scoping** (can't load what you can't access) and **explicit permission checks**.

### Query Scoping via *Scoped Concerns

The query IS the authorization. `find` raises `RecordNotFound` (404) if no access:

```ruby
# app/controllers/concerns/board_scoped.rb
module BoardScoped
  extend ActiveSupport::Concern
  included { before_action :set_board }

  private
    def set_board
      @board = Current.user.boards.find(params[:board_id])
    end

    def ensure_permission_to_admin_board
      head :forbidden unless Current.user.can_administer_board?(@board)
    end
end

# app/controllers/concerns/card_scoped.rb
def set_card
  @card = Current.user.accessible_cards.find_by!(number: params[:card_id])
end
```

For public + authenticated access, scope to accessible-or-published:

```ruby
def set_book
  @book = Book.accessable_or_published.find(params[:book_id])
end

def ensure_editable
  head :forbidden unless @book.editable?
end
```

### Explicit Permission Checks

Gate specific actions with `ensure_*` methods returning `head :forbidden`:

```ruby
before_action :ensure_can_administer, only: %i[ update destroy ]

def ensure_can_administer
  head :forbidden unless Current.user.can_administer?(@room)
end
```

## Involvement & Watching

Layer notification preferences on the access join model:

- **Board-level**: `involvement` enum on `Access` (`access_only` vs `watching`)
- **Card-level**: Separate `Watch` model for per-card opt-in/out

```ruby
class Watch < ApplicationRecord
  belongs_to :user
  belongs_to :card, touch: true
  scope :watching, -> { where(watching: true) }
end
```

For chat apps, use finer-grained involvement with room-type defaults:

```ruby
enum :involvement, %w[ invisible nothing mentions everything ].index_by(&:itself), prefix: :involved_in

# Direct messages default to "everything", group rooms to "mentions"
def default_involvement
  direct? ? "everything" : "mentions"
end
```

## Data Cleanup on Revocation

When revoking access, clean up related data asynchronously:

```ruby
# app/models/access.rb
after_destroy_commit :clean_inaccessible_data_later

# app/models/board/accessible.rb
def clean_inaccessible_data_for(user)
  return if accessible_to?(user)  # handles re-grant race condition

  mentions_for_user(user).destroy_all
  notifications_for_user(user).destroy_all
  watches_for(user).destroy_all
  pins_for(user).destroy_all
end
```

The `return if accessible_to?` guard is critical — if access was revoked then re-granted before the job runs, skip cleanup.
