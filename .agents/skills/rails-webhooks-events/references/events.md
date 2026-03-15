# Event System

Read this when adding activity tracking to a model, building an activity timeline, feeding event history to an LLM, or understanding how events drive notifications and webhooks.

## Table of Contents
- [Event Model](#event-model)
- [Eventable Concern](#eventable-concern)
- [Event Descriptions](#event-descriptions)
- [Events as LLM Context](#events-as-llm-context)

## Event Model

Single source of truth — one Event record drives activity timelines, notifications, and webhooks.

**When adding `should_track_event?`**: Guard event creation for system-generated records (e.g., records created by background jobs) to prevent infinite loops where events trigger jobs that create more events.

```ruby
class Event < ApplicationRecord
  include Notifiable, Particulars, Promptable

  belongs_to :account, default: -> { board.account }
  belongs_to :board
  belongs_to :creator, class_name: "User"
  belongs_to :eventable, polymorphic: true

  has_many :webhook_deliveries, class_name: "Webhook::Delivery", dependent: :delete_all

  after_create -> { eventable.event_was_created(self) }
  after_create_commit :dispatch_webhooks
  after_create_commit :notify_recipients_later  # from Notifiable

  private
    def dispatch_webhooks
      Event::WebhookDispatchJob.perform_later(self)
    end
end
```

Use `after_create_commit` (not `after_create`) for fan-out — ensures the event is visible to other processes.

### Particulars — Action-Specific Data

Store extra data as JSON via `store_accessor`:

```ruby
module Event::Particulars
  extend ActiveSupport::Concern

  included do
    store_accessor :particulars, :assignee_ids
  end

  def assignees
    @assignees ||= User.where id: assignee_ids
  end
end
```

## Eventable Concern

Any model includes `Eventable` and calls `track_event`:

```ruby
module Eventable
  extend ActiveSupport::Concern

  included do
    has_many :events, as: :eventable, dependent: :destroy
  end

  def track_event(action, creator: Current.user, board: self.board, **particulars)
    if should_track_event?
      board.events.create!(action: "#{eventable_prefix}_#{action}", creator:, board:, eventable: self, particulars:)
    end
  end

  def event_was_created(event)
    # Override in model-specific concerns for side effects
  end

  private
    def should_track_event? = true
    def eventable_prefix = self.class.name.demodulize.underscore
end
```

### Card Events — Side Effects

```ruby
module Card::Eventable
  include ::Eventable

  included do
    before_create { self.last_active_at ||= created_at || Time.current }
    after_save :track_title_change, if: :saved_change_to_title?
  end

  def event_was_created(event)
    transaction do
      create_system_comment_for(event)
      touch_last_active_at unless was_just_published?
    end
  end

  private
    def should_track_event? = published?
end
```

Only published cards generate events. System comments are created for the in-card activity timeline.

### Preventing Infinite Loops

System comments don't create events — `should_track_event?` returns false for system users:

```ruby
module Comment::Eventable
  include ::Eventable

  private
    def should_track_event? = !creator.system?
end
```

Chain: action -> event -> system comment -> (no event, because system user).

## Event Descriptions

Render events as human-readable text with HTML and plain text variants:

```ruby
class Event::Description
  include ActionView::Helpers::TagHelper

  def to_html
    to_sentence(creator_tag, card_title_tag).html_safe
  end

  def to_plain_text
    to_sentence(creator_name, quoted(card.title))
  end

  private
    def action_sentence(creator, card_title)
      case event.action
      when "card_assigned"       then assigned_sentence(creator, card_title)
      when "card_published"      then "#{creator} added #{card_title}"
      when "card_closed"         then %(#{creator} moved #{card_title} to "Done")
      when "card_postponed"      then %(#{creator} moved #{card_title} to "Not Now")
      when "card_auto_postponed" then %(#{card_title} moved to "Not Now" due to inactivity)
      end
    end
end
```

HTML includes personalization — "You" for the current user via data attributes that JavaScript swaps at render time.

## Events as LLM Context

Events serialize to structured prompts for AI features:

```ruby
module Event::Promptable
  def to_prompt
    <<~PROMPT
      BEGIN OF EVENT #{id}
      ## Event #{action} (#{eventable_type} #{eventable_id})
      * Created at: #{created_at}
      * Created by: #{creator.name}
      #{eventable.to_prompt}
      END OF EVENT #{id}
    PROMPT
  end
end
```
