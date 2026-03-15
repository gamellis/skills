# Mailers & Notifications

Read this when adding email notifications, web push, in-app notifications, or deciding which notification channels to implement.

Push notifications only to disconnected users — if someone is actively viewing the page, a Turbo Stream update is faster and less intrusive than an email or push notification.

## Table of Contents
- [Choosing Your Approach](#choosing-your-approach)
- [Notification Bundling](#notification-bundling)
- [Web Push Notifications](#web-push-notifications)
- [In-App Notifications](#in-app-notifications)
- [Notification Routing](#notification-routing)
- [No Notifications](#no-notifications)

## Choosing Your Approach

Match notification complexity to your app:

1. **No notifications** — Self-hosted single-user apps. Use join codes for invitations instead of email.
2. **Web push only** — Chat apps. Immediate push to disconnected users, no email.
3. **Bundled digests + push** — Project management. Immediate push + periodic email digests.

## Notification Bundling

Bundle notifications into time-windowed digests instead of sending individual emails.

### Notification Model

```ruby
class Notification < ApplicationRecord
  belongs_to :user
  belongs_to :creator, class_name: "User"
  belongs_to :source, polymorphic: true

  scope :unread, -> { where(read_at: nil) }

  after_create_commit :broadcast_unread
  after_create :bundle

  def read
    update!(read_at: Time.current)
    broadcast_remove_to user, :notifications
  end

  private
    def broadcast_unread
      broadcast_prepend_later_to user, :notifications, target: "notifications"
    end

    def bundle
      user.bundle(self) if user.settings.bundling_emails?
    end
end
```

### Bundle Model

Use time windows, not join tables:

```ruby
class Notification::Bundle < ApplicationRecord
  belongs_to :user
  enum :status, %i[pending processing delivered]

  scope :due, -> { pending.where("ends_at <= ?", Time.current) }

  def notifications
    user.notifications.where(created_at: starts_at..ends_at).unread
  end

  def deliver
    processing!
    BundleMailer.notification(self).deliver if deliverable?
    delivered!
  end

  class << self
    def deliver_all
      due.in_batches do |batch|
        jobs = batch.collect { DeliverJob.new(it) }
        ActiveJob.perform_all_later(jobs)  # batch-enqueue
      end
    end
  end
end
```

### Recurring Delivery

```yaml
# config/recurring.yml
deliver_bundled_notifications:
  command: "Notification::Bundle.deliver_all_later"
  schedule: every 30 minutes
```

## Web Push Notifications

### Push Only to Disconnected Users

```ruby
class Room::MessagePusher
  def relevant_subscriptions
    Push::Subscription
      .joins(user: :memberships)
      .merge(Membership.visible.disconnected.where(room: room).where.not(user: message.creator))
  end
end
```

### Thread Pool for Delivery

Use a thread pool to avoid blocking job workers during HTTP calls to push services:

```ruby
class WebPush::Pool
  def initialize
    @pool = Concurrent::ThreadPoolExecutor.new(max_threads: 50, queue_size: 10000)
    @connection = Net::HTTP::Persistent.new(name: "web_push", pool_size: 150)
  end

  def queue(payload, subscriptions)
    subscriptions.find_each do |sub|
      notification = sub.notification(**payload)  # build BEFORE thread
      @pool.post { notification.deliver(connection: @connection) }
    end
  end
end
```

Key: build notification objects (ActiveRecord queries) on the calling thread, pass plain objects into the thread pool.

### Validate Push Endpoints

```ruby
class Push::Subscription < ApplicationRecord
  PERMITTED_HOSTS = %w[fcm.googleapis.com updates.push.services.mozilla.com web.push.apple.com]

  validate :validate_endpoint_url

  private
    def validate_endpoint_url
      uri = URI.parse(endpoint) rescue nil
      errors.add(:endpoint, "invalid") unless uri&.scheme == "https" && PERMITTED_HOSTS.include?(uri.host)
    end
end
```

### Self-Heal Expired Subscriptions

Automatically destroy subscriptions that fail:

```ruby
rescue WebPush::ExpiredSubscription
  subscription.destroy
```

## In-App Notifications

Use Turbo Streams — prepend on create, remove on read:

```ruby
broadcast_prepend_later_to user, :notifications, target: "notifications"
broadcast_remove_to user, :notifications
```

```erb
<%= turbo_stream_from Current.user, :notifications %>
<div id="notifications">
  <%# notifications render here %>
</div>
```

## Notification Routing

Use a polymorphic factory to determine notification recipients:

```ruby
# app/models/notifier.rb
class Notifier
  class << self
    def for(source)
      case source
      when Event
        "Notifier::#{source.eventable.class}EventNotifier".safe_constantize&.new(source)
      when Mention
        MentionNotifier.new(source)
      end
    end
  end

  def notify
    if should_notify?
      recipients.sort_by(&:id).map do |recipient|
        Notification.create! user: recipient, source: source, creator: creator
      end
    end
  end
end
```

Each subclass defines recipient logic per action:

```ruby
class Notifier::CardEventNotifier < Notifier
  private
    def recipients
      case source.action
      when "card_assigned"
        source.assignees.excluding(creator)
      when "card_published"
        board.watchers.without(creator, *card.mentionees).including(*card.assignees).uniq
      else
        board.watchers.without(creator)
      end
    end
end
```

Always exclude the creator and mentionees (who get separate mention notifications).

Wire into models with `Notifiable`:

```ruby
module Notifiable
  included do
    has_many :notifications, as: :source, dependent: :destroy
    after_create_commit :notify_recipients_later
  end

  def notify_recipients
    Notifier.for(self)&.notify
  end
end
```

## No Notifications

For simple self-hosted apps, skip notifications entirely. Use join codes for invitations:

```ruby
module Account::Joinable
  included do
    before_create { self.join_code = SecureRandom.alphanumeric(12).scan(/.{4}/).join("-") }
  end
end
```

No SMTP server required. Share URL or QR code.
