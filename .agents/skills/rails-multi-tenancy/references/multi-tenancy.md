# Multi-Tenancy — Detailed Reference

Read this when implementing tenant isolation, propagating tenant context to background jobs, managing account lifecycles, or adding SaaS features to a self-hosted app.

**When adding a new query**: Always scope through `Current.account` — never query a model directly. Unscoped queries are the #1 cause of cross-tenant data leaks.

**Don't use `apartment` or schema-per-tenant** — SCRIPT_NAME path rewriting is simpler, works with any database, and doesn't require schema migrations per tenant.

## Table of Contents
- [Dual-Mode Architecture](#dual-mode-architecture)
- [SCRIPT_NAME Middleware](#script_name-middleware)
- [CurrentAttributes and Account Context](#currentattributes-and-account-context)
- [Background Job Propagation](#background-job-propagation)
- [Multi-Tenant Toggle](#multi-tenant-toggle)
- [Account Lifecycle](#account-lifecycle)
- [Sharded Full-Text Search](#sharded-full-text-search)
- [Database Topology](#database-topology)
- [SaaS Engine Pattern](#saas-engine-pattern)

## Dual-Mode Architecture

Same codebase serves self-hosted (SQLite) and SaaS (MySQL):

```ruby
module Fizzy
  class << self
    def saas?
      return @saas if defined?(@saas)
      @saas = !!(((ENV["SAAS"] || File.exist?(File.expand_path("../tmp/saas.txt", __dir__))) && ENV["SAAS"] != "false"))
    end

    def db_adapter
      @db_adapter ||= DbAdapter.new ENV.fetch("DATABASE_ADAPTER", saas? ? "mysql" : "sqlite")
    end
  end
end
```

Database config delegates based on this flag:

```ruby
# config/database.yml
<%
  config_path = if Fizzy.saas?
    File.join(Rails.root.join("saas").to_s, "config", "database.yml")
  else
    File.join("config", "database.#{Fizzy.db_adapter}.yml")
  end
%>
<%= ERB.new(File.read(config_path)).result %>
```

Self-hosted: one SQLite file per database (primary, cable, queue, cache). SaaS: six MySQL databases on potentially different hosts.

## SCRIPT_NAME Middleware

Extract account ID from URL path, rewrite SCRIPT_NAME so all route helpers include the prefix:

```ruby
module AccountSlug
  PATTERN = /(\d+)/
  PATH_INFO_MATCH = /\A(\/#{AccountSlug::PATTERN})/

  class Extractor
    def initialize(app)
      @app = app
    end

    def call(env)
      request = ActionDispatch::Request.new(env)

      if request.script_name && request.script_name =~ PATH_INFO_MATCH
        env["fizzy.external_account_id"] = AccountSlug.decode($2)
      elsif request.path_info =~ PATH_INFO_MATCH
        request.engine_script_name = request.script_name = $1
        request.path_info   = $'.empty? ? "/" : $'
        env["fizzy.external_account_id"] = AccountSlug.decode($2)
      end

      if env["fizzy.external_account_id"]
        account = Account.find_by(external_account_id: env["fizzy.external_account_id"])
        Current.with_account(account) { @app.call env }
      else
        Current.without_account { @app.call env }
      end
    end
  end
end

Rails.application.config.middleware.insert_after Rack::TempfileReaper, AccountSlug::Extractor
```

Request to `/12345/boards/1`: middleware moves `/12345` from PATH_INFO to SCRIPT_NAME. Rails thinks the app is "mounted" at `/12345`, so `boards_path` produces `/12345/boards`. No nested routes needed.

For ActionCable reconnections, the slug may already be in SCRIPT_NAME from a previous connection — both paths are handled.

## CurrentAttributes and Account Context

```ruby
class Current < ActiveSupport::CurrentAttributes
  attribute :session, :user, :identity, :account
  attribute :http_method, :request_id, :user_agent, :ip_address, :referrer

  def session=(value)
    super(value)
    self.identity = session.identity if value.present?
  end

  def identity=(identity)
    super(identity)
    self.user = identity.users.find_by(account: account) if identity.present?
  end

  def with_account(value, &) = with(account: value, &)
  def without_account(&) = with(account: nil, &)
end
```

Setting `Current.account` cascades: when `identity` is set afterward, it resolves the per-account User from the global Identity.

## Background Job Propagation

Jobs run outside request context. Serialize tenant as GlobalID:

```ruby
module FizzyActiveJobExtensions
  extend ActiveSupport::Concern

  prepended do
    attr_reader :account
    self.enqueue_after_transaction_commit = true
  end

  def initialize(...)
    super
    @account = Current.account
  end

  def serialize
    super.merge({ "account" => @account&.to_gid })
  end

  def deserialize(job_data)
    super
    if _account = job_data.fetch("account", nil)
      @account = GlobalID::Locator.locate(_account)
    end
  end

  def perform_now
    if account.present?
      Current.with_account(account) { super }
    else
      super
    end
  end
end

ActiveSupport.on_load(:active_job) do
  prepend FizzyActiveJobExtensions
end
```

Every job captures `Current.account` at enqueue, serializes as GlobalID, restores before execution. `enqueue_after_transaction_commit = true` prevents jobs from running before the enqueuing transaction commits.

## Multi-Tenant Toggle

```ruby
module Account::MultiTenantable
  extend ActiveSupport::Concern

  included do
    cattr_accessor :multi_tenant, default: false
  end

  class_methods do
    def accepting_signups?
      multi_tenant || Account.none?
    end
  end
end
```

Self-hosted: one account (signups only when none exist). SaaS engine flips `Account.multi_tenant = true`. Same flag gates cancellation — prevents self-hosted users from deleting their only account.

## Account Lifecycle

### Creation

```ruby
class Account < ApplicationRecord
  include Storage, Cancellable, Entropic, Incineratable, MultiTenantable, Seedeable

  before_create :assign_external_account_id
  after_create :create_join_code

  class << self
    def create_with_owner(account:, owner:)
      create!(**account).tap do |account|
        account.users.create!(role: :system, name: "System")
        account.users.create!(**owner.with_defaults(role: :owner, verified_at: Time.current))
      end
    end
  end

  def slug
    "/#{AccountSlug.encode(external_account_id)}"
  end
end
```

Every account gets: sequential `external_account_id` (URL slug), system user, owner user, join code.

Sequential IDs via row-level locking:

```ruby
class Account::ExternalIdSequence < ApplicationRecord
  class << self
    def next
      with_lock do |sequence|
        sequence.increment!(:value).value
      end
    end

    private
      def with_lock
        transaction do
          sequence = lock.first_or_create!(value: initial_value)
          yield sequence
        end
      end
  end
end
```

### Cancellation — Two-Phase Deletion

```ruby
module Account::Cancellable
  included do
    has_one :cancellation, dependent: :destroy
    define_callbacks :cancel
    define_callbacks :reactivate
  end

  def cancel(initiated_by: Current.user)
    with_lock do
      if cancellable? && active?
        run_callbacks :cancel do
          create_cancellation!(initiated_by: initiated_by)
        end
        AccountMailer.cancellation(cancellation).deliver_later
      end
    end
  end

  def reactivate
    with_lock do
      if cancelled?
        run_callbacks :reactivate do
          cancellation.destroy
        end
      end
    end
  end
end
```

### Incineration — 30-Day Grace Period

```ruby
module Account::Incineratable
  INCINERATION_GRACE_PERIOD = 30.days

  included do
    scope :due_for_incineration, -> {
      joins(:cancellation).where(account_cancellations: { created_at: ...INCINERATION_GRACE_PERIOD.ago })
    }
    define_callbacks :incinerate
  end

  def incinerate
    run_callbacks :incinerate do
      account.destroy
    end
  end
end
```

Recurring job with `ActiveJob::Continuable`:

```ruby
class Account::IncinerateDueJob < ApplicationJob
  include ActiveJob::Continuable
  queue_as :incineration

  def perform
    step :incineration do |step|
      Account.due_for_incineration.find_each do |account|
        account.incinerate
        step.checkpoint!
      end
    end
  end
end
```

SaaS engine hooks: `set_callback :cancel, :after, -> { subscription&.pause }` and `set_callback :incinerate, :before, -> { subscription&.cancel }`.

## Sharded Full-Text Search

### Auto-Detect Database Backend

```ruby
class Search::Record < ApplicationRecord
  include const_get(connection.adapter_name)  # SQLite or Trilogy

  belongs_to :searchable, polymorphic: true
  belongs_to :card

  scope :for_query, ->(query, user:) do
    query = Search::Query.wrap(query)
    if query.valid? && user.board_ids.any?
      matching(query.to_s, user.account_id).where(account_id: user.account_id, board_id: user.board_ids)
    else
      none
    end
  end
end
```

### SQLite Mode — FTS5

```ruby
module Search::Record::SQLite
  included do
    has_one :search_records_fts, -> { with_rowid },
      class_name: "Search::Record::SQLite::Fts", foreign_key: :rowid, primary_key: :id
    after_save :upsert_to_fts5_table
    scope :matching, ->(query, _) { joins(:search_records_fts).where("search_records_fts MATCH ?", query) }
  end
end
```

### MySQL Mode — 16 Shards

```ruby
module Search::Record::Trilogy
  SHARD_COUNT = 16

  included do
    self.abstract_class = true
    before_save :set_account_key, :stem_content

    scope :matching, ->(query, account_id) do
      full_query = "+account#{account_id} +(#{Search::Stemmer.stem(query)})"
      where("MATCH(#{table_name}.account_key, #{table_name}.content, #{table_name}.title) AGAINST(? IN BOOLEAN MODE)", full_query)
    end

    SHARD_CLASSES = SHARD_COUNT.times.map do |shard_id|
      Class.new(self) do
        self.table_name = "search_records_#{shard_id}"
        def self.name = "Search::Record"
      end
    end.freeze
  end

  class_methods do
    def shard_id_for_account(account_id)
      Zlib.crc32(account_id.to_s) % SHARD_COUNT
    end

    def for(account_id)
      SHARD_CLASSES[shard_id_for_account(account_id)]
    end
  end
end
```

16 anonymous ActiveRecord classes at boot, each pointing to `search_records_0` through `search_records_15`. CRC32 maps accounts to shards. Account key in the FULLTEXT index scopes results per tenant.

### Searchable Concern

```ruby
module Searchable
  SEARCH_CONTENT_LIMIT = 32.kilobytes

  included do
    after_create_commit :create_in_search_index
    after_update_commit :update_in_search_index
    after_destroy_commit :remove_from_search_index
  end

  private
    def search_record_class
      Search::Record.for(account_id)
    end
end
```

Models include `Searchable` and implement `search_title`, `search_content`, `searchable?`. Content truncated to 32KB.

## Database Topology

### SaaS Mode — Six Databases

```yaml
production:
  primary:   # Main app data
  replica:   # Read replica (different host, readonly)
  cable:     # Solid Cable
  queue:     # Solid Queue
  cache:     # Solid Cache
  saas:      # Billing data (SaasRecord base class)
```

### Read Replica Support

```ruby
module ActiveRecordReplicaSupport
  class_methods do
    def configure_replica_connections
      if replica_configured?
        connects_to database: { writing: :primary, reading: :replica }
      end
    end
  end
end

class ApplicationRecord < ActiveRecord::Base
  primary_abstract_class
  configure_replica_connections
end
```

SQLite mode: no replica config, `configure_replica_connections` is a no-op.

### Transaction Pinning (GTID)

After a write, store MySQL GTID in session. On next read, wait for replica to catch up:

```ruby
module TransactionPinning
  class Middleware
    def call(env)
      if ApplicationRecord.current_role == :reading
        wait_for_replica_catchup(request, replica_metrics)
      end

      status, headers, body = @app.call(env)

      if ApplicationRecord.current_role == :writing
        capture_transaction_id(request)  # Store GTID in session
      end

      [status, headers, body]
    end
  end
end
```

Uses `WAIT_FOR_EXECUTED_GTID_SET` with 250ms timeout. Prevents stale reads without sticky sessions.

### Multi-Datacenter Routing

```ruby
class Deployment::DatabaseResolver < ActiveRecord::Middleware::DatabaseSelector::Resolver
  def self.in_primary_datacenter?
    ENV["PRIMARY_DATACENTER"].present? || Rails.env.local?
  end

  def reading_request?(request)
    super || !DatabaseResolver.in_primary_datacenter?
  end

  private
    def read_from_primary?
      super && DatabaseResolver.in_primary_datacenter?
    end
end
```

Non-primary datacenters forced to read from local replicas. Only primary datacenter can write.

## SaaS Engine Pattern

Layer SaaS features via a Rails engine in `saas/`:

```ruby
# saas/lib/fizzy/saas/engine.rb
config.to_prepare do
  ::Account.include Account::Billing, Account::Limited
  ::User.include User::NotifiesAccountOfEmailChange
  ::Signup.prepend Fizzy::Saas::Signup
  CardsController.include(Card::LimitedCreation)
  Cards::PublishesController.include(Card::LimitedPublishing)
  ::ApplicationController.include Fizzy::Saas::Authorization::Controller
end
```

The engine injects:
- **Account::Billing** — Stripe subscriptions, plan management
- **Account::Limited** — card count and storage limits per plan
- **Card::LimitedCreation** — prevents creation when over limits
- **Fizzy::Saas::Authorization** — restricts staging access

### SaaS Signup Extension

Base app defines template methods, SaaS engine overrides:

```ruby
module Fizzy::Saas::Signup
  private
    def create_tenant
      @queenbee_account = Queenbee::Remote::Account.create!(queenbee_account_attributes)
      @queenbee_account.id.to_s
    end

    def handle_account_creation_error(error)
      @queenbee_account&.cancel
    end
end
```

Self-hosted: `create_tenant` returns nil. SaaS: creates external billing account.
