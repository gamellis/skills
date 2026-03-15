# Project Setup

Read this when starting a new Rails application or reviewing the baseline gem stack and configuration.

## Table of Contents
- [Gemfile](#gemfile)
- [Application Config](#application-config)

## Gemfile

Start every Rails app with this core stack. No bundler (importmap-rails loads ES modules directly from the browser), no CSS preprocessor (Tailwind v4 handles everything in CSS), and no auth gem (hand-rolled is ~60 lines):

```ruby
gem "rails", github: "rails/rails"  # or latest stable
gem "propshaft"
gem "importmap-rails"
gem "turbo-rails"
gem "stimulus-rails"
gem "puma"
gem "thruster", require: false
gem "solid_queue"
gem "solid_cache"
gem "solid_cable"
gem "sqlite3"
gem "bcrypt"
gem "image_processing"
gem "rubocop-rails-omakase", require: false
gem "bootsnap", require: false
```

No webpack, esbuild, or JS bundler. No Devise. No Sass/PostCSS/Tailwind.

Add only when needed:
- `kamal` — deployment orchestration
- `geared_pagination` — flexible pagination
- `aws-sdk-s3` — cloud storage (SaaS only)
- `web-push` + `net-http-persistent` — push notifications
- `mission_control-jobs` — job dashboard
- `trilogy` — MySQL adapter (SaaS with networked DB)

## Application Config

```ruby
# config/application.rb
config.load_defaults 8.0  # use latest stable
config.autoload_lib(ignore: %w[assets tasks rails_ext])

# Use UUID PKs only for multi-tenant SaaS:
# config.generators do |g|
#   g.orm :active_record, primary_key_type: :uuid
# end
```

For Docker, Puma tuning, Thruster, and process management, use the **rails-deployment** skill.
