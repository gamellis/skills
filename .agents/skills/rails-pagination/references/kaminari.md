# Kaminari — Traditional Page-Number Pagination

Read this when building admin panels, SEO-friendly index pages, or any view that needs numbered page links. Use `without_count` mode for large tables.

## Table of Contents
- [Setup](#setup)
- [Basic Usage](#basic-usage)
- [Configuration](#configuration)
- [View Helpers](#view-helpers)
- [Performance: without_count](#performance-without_count)
- [With Turbo](#with-turbo)

## Setup

```ruby
# Gemfile
gem "kaminari"
```

Generate config:
```bash
rails generate kaminari:config
```

## Basic Usage

### Controller

```ruby
class UsersController < ApplicationController
  def index
    @users = User.order(:name).page(params[:page])
  end
end
```

Chain `.per(N)` to override default page size:

```ruby
@users = User.order(:name).page(params[:page]).per(50)
```

### View

```erb
<%= paginate @users %>
```

Renders an HTML5 `<nav>` with `?page=N` links: `<< First < Prev 1 2 3 ... 50 Next > Last >>`.

### Metadata

```ruby
@users.total_pages    #=> 50
@users.current_page   #=> 1
@users.next_page      #=> 2
@users.prev_page      #=> nil (on first page)
@users.first_page?    #=> true
@users.last_page?     #=> false
@users.out_of_range?  #=> false
@users.limit_value    #=> 25 (records per page)
```

## Configuration

### Per-Model Defaults

```ruby
class User < ApplicationRecord
  paginates_per 50       # default page size for this model
  max_paginates_per 100  # cap even if .per(999) is called
end
```

### Global Config

```ruby
# config/initializers/kaminari_config.rb
Kaminari.configure do |config|
  config.default_per_page = 25
  config.max_per_page = 100
  config.window = 4        # pages shown around current
  config.outer_window = 0  # pages shown at start/end
  config.left = 0
  config.right = 0
  config.page_method_name = :page
  config.param_name = :page
end
```

## View Helpers

### Customize Window Size

```erb
<%= paginate @users, window: 2 %>
<%= paginate @users, outer_window: 3 %>
```

### Pass Extra Params

```erb
<%= paginate @users, params: { controller: "admin/users", format: :turbo_stream } %>
```

### Custom Theme

Generate theme templates:
```bash
rails generate kaminari:views default
rails generate kaminari:views bootstrap5
```

Apply a theme:
```erb
<%= paginate @users, theme: "bootstrap5" %>
```

### Page Entries Info

```erb
<%= page_entries_info @users %>
<%# Renders: "Displaying users 1–25 of 1000 in total" %>
```

### I18n

```yaml
# config/locales/en.yml
en:
  views:
    pagination:
      first: "&laquo; First"
      last: "Last &raquo;"
      previous: "&lsaquo; Prev"
      next: "Next &rsaquo;"
      truncate: "&hellip;"
```

## Performance: without_count

Skip the `SELECT COUNT(*)` query on large tables:

```ruby
@users = User.page(params[:page]).without_count
```

Only provides prev/next links — no total pages, no numbered page links. Use when total count is expensive or unnecessary.

## With Turbo

### Turbo Frames

Wrap pagination in a Turbo Frame to update only the list:

```erb
<%= turbo_frame_tag "users_list" do %>
  <%= render @users %>
  <%= paginate @users %>
<% end %>
```

Clicking page links replaces only the frame content.

### Turbo Stream Format

For partial page updates:

```erb
<%= paginate @users, params: { format: :turbo_stream } %>
```

Respond with a Turbo Stream template:

```ruby
# users_controller.rb
def index
  @users = User.order(:name).page(params[:page])

  respond_to do |format|
    format.html
    format.turbo_stream
  end
end
```

```erb
<%# index.turbo_stream.erb %>
<%= turbo_stream.replace "users_list" do %>
  <%= render partial: "users_list", locals: { users: @users } %>
<% end %>
```
