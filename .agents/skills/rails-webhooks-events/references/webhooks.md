# Webhook Delivery Pipeline

Read this when building webhook subscriptions, implementing secure HTTP delivery, handling delivery failures, or adding format-aware payloads for integrations like Slack.

`ActiveJob::Continuable` powers the fan-out dispatch — if the job is interrupted mid-delivery, it resumes from the last cursor position instead of re-delivering to all webhooks. This is critical for reliability at scale.

**When making outbound HTTP requests**: Always validate the resolved IP address against private ranges (SSRF protection). External URLs can resolve to internal IPs via DNS rebinding.

## Table of Contents
- [Webhook Model](#webhook-model)
- [Dispatch Chain](#dispatch-chain)
- [Webhook Delivery](#webhook-delivery)
- [SSRF Protection](#ssrf-protection)
- [HMAC Signing](#hmac-signing)
- [Format-Aware Payloads](#format-aware-payloads)
- [Delinquency Tracking](#delinquency-tracking)

## Webhook Model

```ruby
class Webhook < ApplicationRecord
  include Triggerable

  SLACK_WEBHOOK_URL_REGEX = %r{//hooks\.slack\.com/services/T[^\/]+/B[^\/]+/[^\/]+\Z}i
  CAMPFIRE_WEBHOOK_URL_REGEX = %r{/rooms/\d+/\d+-[^\/]+/messages\Z}i
  BASECAMP_CAMPFIRE_WEBHOOK_URL_REGEX = %r{/\d+/integrations/[^\/]+/buckets/\d+/chats/\d+/lines\Z}i

  PERMITTED_ACTIONS = %w[
    card_assigned card_closed card_postponed card_auto_postponed
    card_board_changed card_published card_reopened
    card_sent_back_to_triage card_triaged card_unassigned
    comment_created
  ].freeze

  has_secure_token :signing_secret
  has_many :deliveries, dependent: :delete_all
  has_one :delinquency_tracker, dependent: :delete

  belongs_to :board
  serialize :subscribed_actions, type: Array, coder: JSON
  normalizes :subscribed_actions, with: ->(value) { Array.wrap(value).map(&:to_s).uniq & PERMITTED_ACTIONS }
end
```

Key decisions:
- **Per-board scope** — webhooks belong to a board, not an account
- **Action filtering** — `normalizes` intersects input with `PERMITTED_ACTIONS`
- **Auto-generated signing secret** — `has_secure_token :signing_secret`
- **Destination detection** — regex patterns identify Slack, Campfire, Basecamp URLs

### Triggering

```ruby
module Webhook::Triggerable
  extend ActiveSupport::Concern

  included do
    scope :triggered_by, ->(event) { where(board: event.board).triggered_by_action(event.action) }
    scope :triggered_by_action, ->(action) { where("subscribed_actions LIKE ?", "%\"#{action}\"%") }
  end

  def trigger(event)
    deliveries.create!(event: event) unless account.cancelled?
  end
end
```

Uses LIKE query on JSON array — simpler than a join table for membership checks.

## Dispatch Chain

```
Action on card/comment
  -> track_event("closed")
    -> Event created
      -> after_create_commit :dispatch_webhooks
        -> Event::WebhookDispatchJob
          -> Webhook.active.triggered_by(event).find_each
            -> webhook.trigger(event)
              -> deliveries.create!(event:)
                -> after_create_commit :deliver_later
                  -> Webhook::DeliveryJob
                    -> delivery.deliver
```

The dispatch job uses `ActiveJob::Continuable` for cursor-based resumption:

```ruby
class Event::WebhookDispatchJob < ApplicationJob
  include ActiveJob::Continuable

  queue_as :webhooks
  discard_on ActiveJob::DeserializationError

  def perform(event)
    step :dispatch do |step|
      Webhook.active.triggered_by(event).find_each(start: step.cursor) do |webhook|
        webhook.trigger(event)
        step.advance! from: webhook.id
      end
    end
  end
end
```

If interrupted, resumes from last webhook ID. Each delivery retries independently.

## Webhook Delivery

```ruby
class Webhook::Delivery < ApplicationRecord
  STALE_TRESHOLD = 7.days
  USER_AGENT = "fizzy/1.0.0 Webhook"
  ENDPOINT_TIMEOUT = 7.seconds
  MAX_RESPONSE_SIZE = 100.kilobytes

  enum :state, %w[ pending in_progress completed errored ].index_by(&:itself)

  def deliver
    in_progress!
    self.request[:headers] = headers
    self.response = perform_request
    self.state = :completed
    save!
    webhook.delinquency_tracker.record_delivery_of(self)
  rescue
    errored!
    raise
  end
end
```

## SSRF Protection

Resolve hostname, reject private IPs, pin connection to resolved IP:

```ruby
def perform_request
  if resolved_ip.nil?
    { error: :private_uri }
  else
    request = Net::HTTP::Post.new(uri, headers).tap { |r| r.body = payload }
    response = http.request(request) { |r| stream_body_with_limit(r) }
    { code: response.code.to_i }
  end
end

def resolved_ip
  @resolved_ip ||= SsrfProtection.resolve_public_ip(uri.host)
end

def http
  Net::HTTP.new(uri.host, uri.port).tap do |http|
    http.ipaddr = resolved_ip  # Prevents DNS rebinding
    http.use_ssl = (uri.scheme == "https")
    http.open_timeout = ENDPOINT_TIMEOUT
    http.read_timeout = ENDPOINT_TIMEOUT
  end
end
```

### Response Streaming with Size Limit

```ruby
def stream_body_with_limit(response)
  bytes_read = 0
  response.read_body do |chunk|
    bytes_read += chunk.bytesize
    raise ResponseTooLarge if bytes_read > MAX_RESPONSE_SIZE
  end
end
```

### Categorized Error Handling

Every failure mode is a typed symbol — `completed` with error, not `errored` (reserved for unexpected exceptions):

```ruby
def perform_request
  # ...
rescue ResponseTooLarge
  { error: :response_too_large }
rescue Resolv::ResolvTimeout, Resolv::ResolvError, SocketError
  { error: :dns_lookup_failed }
rescue Net::OpenTimeout, Net::ReadTimeout, Errno::ETIMEDOUT
  { error: :connection_timeout }
rescue Errno::ECONNREFUSED, Errno::EHOSTUNREACH, Errno::ECONNRESET
  { error: :destination_unreachable }
rescue OpenSSL::SSL::SSLError
  { error: :failed_tls }
end
```

## HMAC Signing

Every delivery includes an HMAC-SHA256 signature and timestamp:

```ruby
def headers
  {
    "User-Agent" => USER_AGENT,
    "Content-Type" => content_type,
    "X-Webhook-Signature" => signature,
    "X-Webhook-Timestamp" => event.created_at.utc.iso8601
  }
end

def signature
  OpenSSL::HMAC.hexdigest("SHA256", webhook.signing_secret, payload)
end
```

Receivers verify by computing the same HMAC with their signing secret.

## Format-Aware Payloads

Detect destination by URL regex, render the same event in each service's format:

```ruby
def content_type
  if webhook.for_campfire?    then "text/html"
  elsif webhook.for_basecamp? then "application/x-www-form-urlencoded"
  else "application/json"
  end
end

def payload
  @payload ||= if webhook.for_basecamp?
    { content: render_payload(formats: :html) }.to_query
  elsif webhook.for_campfire?
    render_payload(formats: :html)
  elsif webhook.for_slack?
    slack_payload
  else
    render_payload(formats: :json)
  end
end

def slack_payload
  text = event.description_for(nil).to_plain_text
  url = polymorphic_url(event.eventable, base_url_options.merge(script_name: account.slug))
  { text: "#{text} <#{url}|Open in Fizzy>" }.to_json
end
```

## Delinquency Tracking

Auto-deactivate webhooks after sustained failure (both thresholds must be met):

```ruby
class Webhook::DelinquencyTracker < ApplicationRecord
  DELINQUENCY_THRESHOLD = 10
  DELINQUENCY_DURATION = 1.hour

  def record_delivery_of(delivery)
    if delivery.succeeded?
      reset
    else
      mark_first_failure_time if consecutive_failures_count.zero?
      increment!(:consecutive_failures_count, touch: true)
      webhook.deactivate if delinquent?
    end
  end

  private
    def reset
      update_columns consecutive_failures_count: 0, first_failure_at: nil
    end

    def delinquent?
      failing_for_too_long? && too_many_consecutive_failures?
    end

    def failing_for_too_long?
      first_failure_at&.before?(DELINQUENCY_DURATION.ago)
    end

    def too_many_consecutive_failures?
      consecutive_failures_count >= DELINQUENCY_THRESHOLD
    end
end
```

10+ consecutive failures AND first failure over 1 hour ago. A single success resets.
