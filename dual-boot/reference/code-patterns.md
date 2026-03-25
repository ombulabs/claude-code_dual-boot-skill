# Code Patterns for `NextRails.next?`

**Based on "The Complete Guide to Upgrade Rails" by FastRuby.io (OmbuLabs)**

Use `NextRails.next?` anywhere your application code must behave differently between the current and next dependency sets. This is the **only** acceptable way to branch on version — whether you are upgrading Rails, Ruby, or another core dependency.

---

## Rails Configuration Changes

### spec/rails_helper.rb — `fixture_path` → `fixture_paths`

```ruby
if NextRails.next?
  config.fixture_paths = ["#{::Rails.root}/spec/fixtures"]
else
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
end
```

### config/initializers/session_store.rb

```ruby
if NextRails.next?
  Rails.application.config.session_store :cookie_store, key: '_myapp_session'
else
  Rails.application.config.session_store :cookie_store, key: '_myapp_session', secure: Rails.env.production?
end
```

---

## Rails Model Changes

### Serialization API change

```ruby
# app/models/user.rb
if NextRails.next?
  serialize :preferences, coder: JSON
else
  serialize :preferences, JSON
end
```

---

## Ruby Version Differences

### Handling stdlib removals (e.g., `net-smtp` extracted in Ruby 3.1)

```ruby
# Gemfile
if next?
  gem 'net-smtp', require: false  # extracted from stdlib in Ruby 3.1
end
```

### Handling syntax or behavior changes

```ruby
# app/services/parser.rb
if NextRails.next?
  # Ruby 3.2+ uses a different default for Regexp timeout
  Regexp.timeout = 5
end
```

---

## Core Dependency Changes

### Sidekiq API change (6.x → 7.x)

```ruby
# config/initializers/sidekiq.rb
if NextRails.next?
  Sidekiq.default_job_options = { 'retry' => 3 }
else
  Sidekiq.default_worker_options = { 'retry' => 3 }
end
```

---

## Gem Version Differences in the Gemfile

### Inline ternary for gem versions in test groups

```ruby
# Gemfile
group :development, :test do
  gem 'rspec-rails', next? ? '~> 6.0' : '~> 5.1'
  gem 'factory_bot_rails'
end
```

---

## General Pattern

```ruby
if NextRails.next?
  # Code for the TARGET version (new behavior)
else
  # Code for the CURRENT version (old behavior)
end
```

Always put the **new version** code in the `if` branch and the **old version** code in the `else` branch. This makes cleanup easier — after the upgrade, you keep the `if` branch and remove the `else`.
