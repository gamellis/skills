# Real-Time & ActionCable

Read this when adding live updates, presence indicators, or real-time collaboration features.

**Don't build custom ActionCable channels for simple notifications** — `broadcasts_refreshes` or a single Turbo Stream broadcast covers most cases without any channel code.

## Table of Contents
- [Choosing Your Level](#choosing-your-level)
- [Turbo Broadcasts (Simple)](#turbo-broadcasts-simple)
- [Custom ActionCable Channels](#custom-actioncable-channels)
- [Presence Tracking](#presence-tracking)
- [Connection Authentication](#connection-authentication)
- [Solid Cable](#solid-cable)

## Choosing Your Level

Match real-time complexity to your domain:

1. **Minimal** — Single Turbo broadcast (e.g., "being edited" indicator). No custom channels.
2. **Standard** — `broadcasts_refreshes` for model changes, explicit broadcasts for notifications. No custom channels.
3. **Full** — Custom ActionCable channels for presence, typing indicators, unread sync. Only for chat-like apps.

Most apps need level 1 or 2.

## Turbo Broadcasts (Simple)

### Auto-Refresh on Model Changes

```ruby
# app/models/card/broadcastable.rb
module Card::Broadcastable
  extend ActiveSupport::Concern
  included do
    broadcasts_refreshes  # Turbo Morph on any update
  end
end
```

### Explicit Broadcasts

```ruby
# Notifications — prepend on create, remove on read
after_create_commit -> { broadcast_prepend_later_to user, :notifications, target: "notifications" }
def broadcast_read = broadcast_remove_to(user, :notifications)

# Messages — append to room
def broadcast_create
  broadcast_append_to room, :messages, target: [room, :messages]
end
```

Use `_later` suffix to enqueue broadcast via background job.

### Subscribe in Views

```erb
<%= turbo_stream_from Current.user, :notifications %>
<%= turbo_stream_from @board %>
```

## Custom ActionCable Channels

Only build custom channels when Turbo broadcasts aren't enough (presence, typing, etc.).

### Channel with Authorization

```ruby
class RoomChannel < ApplicationCable::Channel
  def subscribed
    if @room = current_user.rooms.find_by(id: params[:room_id])
      stream_for @room
    else
      reject
    end
  end
end
```

Authorization happens at subscription time — scope through `current_user`.

### Typing Notifications

Server side is trivial — just relay:

```ruby
class TypingNotificationsChannel < RoomChannel
  def start(data)
    broadcast_to @room, action: :start, user: current_user.slice(:id, :name)
  end

  def stop(data)
    broadcast_to @room, action: :stop, user: current_user.slice(:id, :name)
  end
end
```

Client side does the real work: throttle outbound (1/sec), auto-purge inbound (5-sec timeout).

### Empty Heartbeat Channel

An empty channel gives you connected/disconnected callbacks for free:

```ruby
class HeartbeatChannel < ApplicationCable::Channel
end
```

Client subscribes and uses WebSocket state to detect online/offline, fetch missed messages on reconnect.

## Presence Tracking

Use an integer connection counter, not a boolean:

```ruby
# app/models/membership/connectable.rb
module Membership::Connectable
  CONNECTION_TTL = 60.seconds

  included do
    scope :connected, -> { where(connected_at: CONNECTION_TTL.ago..) }
    scope :disconnected, -> { where(connected_at: [nil, ...CONNECTION_TTL.ago]) }
  end

  def connected? = connected_at? && connected_at >= CONNECTION_TTL.ago

  def present
    self.class.connect(self, connected? ? connections + 1 : 1)
  end

  def disconnected
    decrement_connections
    update!(connected_at: nil) if connections < 1
  end

  def refresh_connection
    increment_connections unless connected?
    touch :connected_at
  end
end
```

Each browser tab increments `connections` on subscribe, decrements on unsubscribe. User is offline only when all tabs close. Client refreshes every 50 seconds (TTL is 60 seconds).

Debounce visibility changes by 5 seconds to ignore brief tab switches.

## Connection Authentication

Reuse HTTP session cookies — no separate token system:

```ruby
# app/channels/application_cable/connection.rb
module ApplicationCable
  class Connection < ActionCable::Connection::Base
    identified_by :current_user

    def connect
      self.current_user = find_verified_user
    end

    private
      def find_verified_user
        if session = Session.find_by(token: cookies.signed[:session_token])
          session.user
        else
          reject_unauthorized_connection
        end
      end
  end
end
```

## Solid Cable

Use Solid Cable (database-backed) instead of Redis for ActionCable:

```yaml
# config/cable.yml
production:
  adapter: solid_cable
  connects_to:
    database:
      writing: cable
  polling_interval: 0.1.seconds
  message_retention: 1.day
```

Eliminates Redis dependency. Fine for most apps that don't need sub-millisecond message delivery.
