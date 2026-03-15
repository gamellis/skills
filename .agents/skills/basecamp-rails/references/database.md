# Database Patterns

Read this when choosing a database, setting up full-text search, deciding on primary key strategy, or defining schema conventions.

## Table of Contents
- [SQLite First](#sqlite-first)
- [FTS5 Full-Text Search](#fts5-full-text-search)
- [Primary Key Strategy](#primary-key-strategy)
- [Schema Conventions](#schema-conventions)
- [Float Position Scores](#float-position-scores)

## SQLite First

Use SQLite as the default database. It requires no separate server and handles more than most people expect, even in production. SQLite eliminates connection pooling, port conflicts, and the "works on my machine" problems that come with running a separate database server.

Only switch to a networked database (Postgres/MySQL) for multi-tenant SaaS or when you need concurrent writes from multiple processes.

**When choosing UUID vs integer primary keys**: Use integer PKs by default (faster, smaller). Use UUIDs only when IDs are exposed in URLs and you need to prevent enumeration (e.g., multi-tenant SaaS).

```ruby
gem "sqlite3"
```

## FTS5 Full-Text Search

Use SQLite's FTS5 for full-text search. No Elasticsearch needed.

### Migration

```ruby
# Use execute for virtual tables (not supported by schema DSL)
execute <<~SQL
  CREATE VIRTUAL TABLE leaf_search_index USING fts5(title, content, tokenize='porter')
SQL
```

### Searchable Concern

```ruby
# app/models/leaf/searchable.rb
module Leaf::Searchable
  extend ActiveSupport::Concern

  included do
    after_create_commit  :create_in_search_index
    after_update_commit  :update_in_search_index
    after_destroy_commit :remove_from_search_index
  end

  class_methods do
    def search(terms)
      joins("JOIN leaf_search_index ON leaves.id = leaf_search_index.rowid")
        .where("leaf_search_index MATCH ?", terms)
        .select("leaves.*",
          "highlight(leaf_search_index, 0, '<mark>', '</mark>') AS title_match",
          "snippet(leaf_search_index, 1, '<mark>', '</mark>', '...', 20) AS content_match")
    end
  end

  private
    def create_in_search_index
      execute_sql_with_binds(
        "INSERT INTO leaf_search_index(rowid, title, content) VALUES (?, ?, ?)",
        id, title, searchable_content)
    end

    def update_in_search_index
      execute_sql_with_binds(
        "UPDATE leaf_search_index SET title = ?, content = ? WHERE rowid = ?",
        title, searchable_content, id)
    end

    def remove_from_search_index
      execute_sql_with_binds(
        "DELETE FROM leaf_search_index WHERE rowid = ?", id)
    end

    def execute_sql_with_binds(sql, *binds)
      self.class.connection.exec_query(sql, "SQL", binds.map { |b| [nil, b] })
    end
end
```

For simpler models (like messages with just a body):

```ruby
scope :search, ->(query) {
  joins("JOIN message_search_index idx ON messages.id = idx.rowid")
    .where("idx.body MATCH ?", query)
}
```

## Primary Key Strategy

- **Single-tenant / self-hosted**: Use integer PKs (default). Simpler, faster.
- **Multi-tenant SaaS**: Use UUIDs. Globally unique, no collision across tenants.

```ruby
# config/application.rb — only for multi-tenant
config.generators do |g|
  g.orm :active_record, primary_key_type: :uuid
end
```

## Schema Conventions

### Enforce Uniqueness at the Database Level

```ruby
add_index :accesses, [:user_id, :board_id], unique: true
add_index :memberships, [:room_id, :user_id], unique: true
```

### Foreign Keys

Always add explicit foreign keys:

```ruby
add_foreign_key :sessions, :users
add_foreign_key :messages, :rooms
add_foreign_key :messages, :users, column: "creator_id"
```

### Singleton Tables

For single-tenant apps, enforce one account at the database level:

```ruby
create_table :accounts do |t|
  t.integer :singleton_guard, default: 0, null: false
  t.index :singleton_guard, unique: true
end
```

### Multi-Tenant: Scope All Data

In multi-tenant apps, add `account_id` to every table:

```ruby
create_table :cards, id: :uuid do |t|
  t.uuid :account_id, null: false
  # ...
end
```

## Float Position Scores

Use float columns for ordering items that can be reordered:

```ruby
create_table :leaves do |t|
  t.float :position_score, null: false
end
```

Insert between existing items by computing the midpoint. Rebalance when gaps get too small:

```ruby
module Positionable
  REBALANCE_THRESHOLD = 1e-10

  def move_to_position(offset, followed_by: [])
    with_positioning_lock do
      all_to_move = [self, *followed_by]
      before, after = before_and_after_for(offset: offset, moving: all_to_move)
      gap = (after - before) / (all_to_move.count + 1)

      all_to_move.each.with_index(1) do |item, index|
        item.update!(position_score: before + (index * gap))
      end

      remember_to_rebalance_positions if gap < REBALANCE_THRESHOLD
    end
  end
end
```
