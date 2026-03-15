# Controllers & Routing

Read this when adding a new controller, defining routes, implementing authentication, or deciding how to structure controller logic.

## Table of Contents
- [Thin Controllers](#thin-controllers)
- [Authentication Concern](#authentication-concern)
- [CRUD-Oriented Routing](#crud-oriented-routing)
- [params.expect](#paramsexpect)
- [Rate Limiting](#rate-limiting)

## Thin Controllers

Controllers perform direct Active Record operations. No service layers — service objects add indirection without adding clarity. The controller already *is* the coordination layer.

**Don't invent custom verbs** (e.g., `POST /cards/archive`) — model state changes as CRUD on sub-resources (`resource :closure`, `resource :pin`). This gives you standard REST verbs and before_action hooks for free.

```ruby
class Cards::CommentsController < ApplicationController
  include BoardScoped, CardScoped

  def create
    @comment = @card.comments.create!(comment_params)
  end

  private
    def comment_params
      params.expect(comment: [:body])
    end
end
```

When behavior is complex, the model provides a domain-specific method:

```ruby
class Cards::ClosuresController < ApplicationController
  def create
    @card.close  # all logic lives in Card::Closeable concern
  end

  def destroy
    @card.reopen
  end
end
```

## Authentication Concern

Hand-roll authentication. No Devise, no Warden.

```ruby
# app/controllers/concerns/authentication.rb
module Authentication
  extend ActiveSupport::Concern

  included do
    before_action :require_authentication
    helper_method :signed_in?
  end

  class_methods do
    def allow_unauthenticated_access(**options)
      skip_before_action :require_authentication, **options
    end

    def require_unauthenticated_access(**options)
      allow_unauthenticated_access(**options)
      before_action :redirect_signed_in_user_to_root, **options
    end
  end

  private
    def signed_in? = Current.user.present?

    def require_authentication
      restore_authentication || request_authentication
    end

    def restore_authentication
      if session = find_session_by_cookie
        resume_session(session)
      end
    end

    def find_session_by_cookie
      Session.find_by(token: cookies.signed[:session_token])
    end

    def resume_session(session)
      session.resume(user_agent: request.user_agent, ip_address: request.remote_ip)
      Current.session = session
      Current.user = session.user
    end

    def start_new_session_for(user)
      user.sessions.start!(user_agent: request.user_agent, ip_address: request.remote_ip).tap do |session|
        authenticated_as(session)
      end
    end

    def authenticated_as(session)
      Current.session = session
      Current.user = session.user
      cookies.signed.permanent[:session_token] = { value: session.token, httponly: true, same_site: :lax }
    end

    def request_authentication
      redirect_to new_session_path
    end

    def redirect_signed_in_user_to_root
      redirect_to root_path if signed_in?
    end
end
```

### Session Model

```ruby
class Session < ApplicationRecord
  ACTIVITY_REFRESH_RATE = 1.hour

  has_secure_token
  belongs_to :user

  before_create { self.last_active_at ||= Time.now }

  def self.start!(user_agent:, ip_address:)
    create!(user_agent:, ip_address:)
  end

  def resume(user_agent:, ip_address:)
    if last_active_at.before?(ACTIVITY_REFRESH_RATE.ago)
      update!(user_agent:, ip_address:, last_active_at: Time.now)
    end
  end
end
```

### Sessions Controller

```ruby
class SessionsController < ApplicationController
  require_unauthenticated_access
  rate_limit to: 10, within: 3.minutes, only: :create,
    with: -> { render_rejection :too_many_requests }

  def new
  end

  def create
    if user = User.authenticate_by(email_address: params[:email_address], password: params[:password])
      start_new_session_for(user)
      redirect_to root_path
    else
      render_rejection :unauthorized
    end
  end

  def destroy
    Current.session&.destroy
    cookies.delete(:session_token)
    redirect_to new_session_path
  end
end
```

Use `authenticate_by` (Rails 7.1+) for constant-time comparison that prevents timing attacks.

For authorization, scoping concerns, and access control, see [authorization.md](authorization.md).

## CRUD-Oriented Routing

Model every action as a resource. Never add custom action methods.

```ruby
# Bad
resources :cards do
  post :close
  post :reopen
end

# Good
resources :cards do
  resource :closure, only: [:create, :destroy]
end
```

Use `scope module:` to namespace controllers:

```ruby
resources :cards do
  scope module: :cards do
    resource :closure
    resource :pin
    resources :comments
  end
end
```

Custom URL resolvers for polymorphic routing:

```ruby
resolve "Comment" do |comment, options|
  options[:anchor] = ActionView::RecordIdentifier.dom_id(comment)
  route_for :card, comment.card, options
end
```

## params.expect

Use `params.expect` (Rails 8) for strict parameter filtering:

```ruby
params.expect(board: [:name, :description, :all_access])
params.expect(:email_address)  # raises if missing
```

## Rate Limiting

Use Rails built-in `rate_limit` on auth endpoints:

```ruby
rate_limit to: 10, within: 3.minutes, only: :create,
  with: -> { render_rejection :too_many_requests }
```
