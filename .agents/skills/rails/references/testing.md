# Testing

Read this when writing tests, setting up fixtures, configuring parallel test execution, or writing system tests with multiple users.

## Table of Contents
- [Minitest](#minitest)
- [Fixtures](#fixtures)
- [Test Helper Setup](#test-helper-setup)
- [Parallel Execution](#parallel-execution)
- [System Tests](#system-tests)
- [Test Helpers](#test-helpers)

## Minitest

Use Minitest. No RSpec. Minitest is part of Ruby's standard library — no extra dependencies, no DSL to learn, and test files read as plain Ruby methods.

Use YAML fixtures, not FactoryBot. Fixtures are bulk-inserted once per test suite and shared across all tests, so they run in milliseconds. FactoryBot creates objects per-test, which gets slower as the suite grows.

```
test/
  models/          # Unit tests
  controllers/     # Integration tests (ActionDispatch::IntegrationTest)
  system/          # Browser tests (Capybara + Selenium)
  fixtures/        # YAML fixture data
  test_helper.rb
```

## Fixtures

Use YAML fixtures, not FactoryBot. Fixtures are deterministic and bulk-inserted once per test run.

```ruby
# test/test_helper.rb
fixtures :all
```

```yaml
# test/fixtures/users.yml
alice:
  name: Alice
  email_address: alice@example.com
  role: administrator

bob:
  name: Bob
  email_address: bob@example.com
  role: member
```

Reference in tests:

```ruby
test "admin can delete" do
  assert users(:alice).can_administer?
  refute users(:bob).can_administer?
end
```

## Test Helper Setup

### Single-Tenant

```ruby
# test/test_helper.rb
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
  fixtures :all

  include ActiveJob::TestHelper
  include SessionTestHelper
end
```

### Multi-Tenant

```ruby
class ActiveSupport::TestCase
  parallelize(workers: :number_of_processors)
  fixtures :all

  include ActiveJob::TestHelper
  include SessionTestHelper

  setup { Current.account = accounts(:default) }
  teardown { Current.clear_all }
end

class ActionDispatch::IntegrationTest
  setup do
    integration_session.default_url_options[:script_name] = "/#{accounts(:default).slug}"
  end
end
```

## Parallel Execution

Always parallelize:

```ruby
parallelize(workers: :number_of_processors)
```

## System Tests

### Multi-User Browser Tests

Use `using_session` to simulate multiple users:

```ruby
test "real-time message delivery" do
  using_session("Alice") do
    sign_in users(:alice)
    visit room_path(@room)
  end

  using_session("Bob") do
    sign_in users(:bob)
    visit room_path(@room)
    send_message "Hello Alice!"
  end

  using_session("Alice") do
    assert_text "Hello Alice!"
  end
end
```

### Turbo Broadcast Assertions

```ruby
include Turbo::Broadcastable::TestHelper

test "broadcasts on create" do
  assert_broadcasts(@room, 1) do
    @room.messages.create!(body: "test", creator: users(:alice))
  end
end
```

## Test Helpers

Extract shared test behavior into helpers. `SessionTestHelper` is a must:

```ruby
# test/helpers/session_test_helper.rb
module SessionTestHelper
  def sign_in(user)
    session = user.sessions.create!(user_agent: "Test", ip_address: "127.0.0.1")
    cookies[:session_token] = session.token
    Current.user = user
  end

  def sign_out
    Current.session&.destroy
    cookies.delete(:session_token)
  end
end
```

### HTTP Mocking

Use WebMock to block external requests in tests:

```ruby
setup { WebMock.disable_net_connect! }
teardown { WebMock.reset! }
```

Use VCR for recording/replaying external API interactions when needed.
