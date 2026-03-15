# Cursor-Based Pagination — Real-Time Feeds

## Table of Contents
- [When to Use Cursors](#when-to-use-cursors)
- [The Pagination Concern](#the-pagination-concern)
- [Controller Patterns](#controller-patterns)
- [Bidirectional Loading](#bidirectional-loading)
- [Time-Scoped Queries](#time-scoped-queries)
- [View Integration](#view-integration)
- [Adapting to Other Models](#adapting-to-other-models)

## When to Use Cursors

Offset-based pagination (`LIMIT 40 OFFSET 80`) breaks when records are inserted in real time — page boundaries shift, causing duplicates or skipped records. Cursor-based pagination uses a stable reference point (a record's ID or timestamp) to define "before this" and "after this".

Use cursor pagination for:
- Chat messages
- Activity feeds
- Notification lists
- Any list where records arrive in real time

## The Pagination Concern

Extract pagination into a model concern with a fixed page size:

```ruby
# app/models/message/pagination.rb
module Message::Pagination
  extend ActiveSupport::Concern

  PAGE_SIZE = 40

  included do
    scope :last_page, -> { ordered.last(PAGE_SIZE) }
    scope :first_page, -> { ordered.first(PAGE_SIZE) }

    scope :before, ->(message) { where("created_at < ?", message.created_at) }
    scope :after, ->(message) { where("created_at > ?", message.created_at) }

    scope :page_before, ->(message) { before(message).last_page }
    scope :page_after, ->(message) { after(message).first_page }

    scope :page_created_since, ->(time) { where("created_at > ?", time).first_page }
    scope :page_updated_since, ->(time) { where("updated_at > ?", time).last_page }
  end

  class_methods do
    def page_around(message)
      page_before(message) + [ message ] + page_after(message)
    end

    def paged?
      count > PAGE_SIZE
    end
  end
end
```

Include it on the model:

```ruby
class Message < ApplicationRecord
  include Pagination

  scope :ordered, -> { order(:created_at) }

  belongs_to :room
  belongs_to :creator, class_name: "User", default: -> { Current.user }
end
```

## Controller Patterns

### Basic: Load Last Page or Cursor-Navigate

```ruby
class MessagesController < ApplicationController
  def index
    @messages = find_paged_messages
  end

  private
    def find_paged_messages
      case
      when params[:before].present?
        @room.messages.with_creator.page_before(@room.messages.find(params[:before]))
      when params[:after].present?
        @room.messages.with_creator.page_after(@room.messages.find(params[:after]))
      else
        @room.messages.with_creator.last_page
      end
    end
end
```

URL patterns:
- `/rooms/123/messages` — latest 40 messages
- `/rooms/123/messages?before=msg_456` — 40 messages older than msg_456
- `/rooms/123/messages?after=msg_456` — 40 messages newer than msg_456

### Page Around: Show Context for a Specific Record

When deep-linking to a specific message (e.g., from a search result):

```ruby
class RoomsController < ApplicationController
  def show
    messages = @room.messages.with_creator.with_attachment_details

    if show_first_message = messages.find_by(id: params[:message_id])
      @messages = messages.page_around(show_first_message)
    else
      @messages = messages.last_page
    end
  end
end
```

`page_around` returns up to `PAGE_SIZE` messages before, the target message, and up to `PAGE_SIZE` messages after — providing full context.

## Bidirectional Loading

The cursor approach naturally supports loading in both directions. Views provide "load older" and "load newer" links:

```erb
<% if @room.messages.paged? %>
  <% if @messages.first && @room.messages.before(@messages.first).exists? %>
    <%= link_to "Load older messages",
          room_messages_path(@room, before: @messages.first.id),
          data: { turbo_frame: "messages" } %>
  <% end %>
<% end %>

<%= render partial: "messages/message", collection: @messages %>

<% if @messages.last && @room.messages.after(@messages.last).exists? %>
  <%= link_to "Load newer messages",
        room_messages_path(@room, after: @messages.last.id),
        data: { turbo_frame: "messages" } %>
<% end %>
```

## Time-Scoped Queries

For real-time updates, fetch records created or updated since a timestamp:

```ruby
class Rooms::RefreshesController < ApplicationController
  def show
    @new_messages = @room.messages.with_creator.page_created_since(@last_updated_at)
    @updated_messages = @room.messages
      .without(@new_messages)
      .with_creator
      .page_updated_since(@last_updated_at)
  end
end
```

Respond with Turbo Streams to append new and replace updated:

```erb
<%# rooms/refreshes/show.turbo_stream.erb %>
<% @new_messages.each do |message| %>
  <%= turbo_stream.append "messages", partial: "messages/message", locals: { message: message } %>
<% end %>

<% @updated_messages.each do |message| %>
  <%= turbo_stream.replace message, partial: "messages/message", locals: { message: message } %>
<% end %>
```

For true real-time, combine with ActionCable broadcasting:

```erb
<%= turbo_stream_from @room, :messages %>
```

## Adapting to Other Models

The same pattern works for any ordered collection. Adapt the concern:

### Activity Feed

```ruby
module Event::Pagination
  extend ActiveSupport::Concern

  PAGE_SIZE = 30

  included do
    scope :last_page, -> { latest.last(PAGE_SIZE) }
    scope :page_before, ->(event) { where("created_at < ?", event.created_at).last_page }
    scope :page_since, ->(time) { where("created_at > ?", time).latest.first(PAGE_SIZE) }
  end
end
```

### Notification List

```ruby
module Notification::Pagination
  extend ActiveSupport::Concern

  PAGE_SIZE = 25

  included do
    scope :last_page, -> { ordered.last(PAGE_SIZE) }
    scope :page_before, ->(notification) { where("created_at < ?", notification.created_at).last_page }
    scope :unread_since, ->(time) { unread.where("created_at > ?", time).ordered.first(PAGE_SIZE) }
  end
end
```

### Key Principles

1. **Fixed PAGE_SIZE** — keep it simple, one constant per model
2. **Scope composition** — `before`/`after` scopes compose with `last_page`/`first_page`
3. **Timestamp cursors** — use `created_at` for insertion-order stability
4. **ID cursors** — use `id` when timestamps may collide (sub-second inserts)
5. **No COUNT queries** — check `paged?` only when you need to show/hide pagination UI
6. **Turbo Streams for updates** — append new, replace updated, remove deleted
