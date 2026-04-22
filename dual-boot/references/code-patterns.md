# Code Patterns for `NextRails.next?`

**Based on "The Complete Guide to Upgrade Rails" by FastRuby.io (OmbuLabs)**

Use `NextRails.next?` anywhere your application code must behave differently between the current and next dependency sets. This is the **only** acceptable way to branch on version — whether you are upgrading Rails, Ruby, or another core dependency.

**Only branch when the old code actually breaks on the next version.** A deprecation warning is not a reason for a conditional — if the new API works on both, replace the call site directly and let the deprecation disappear on its own. Reserve `NextRails.next?` for removed constants, removed methods, or incompatible signature/return-type changes that would raise an error otherwise.

---

## Rails API Removals

### `ActionDispatch::Http::ParameterFilter` removed in Rails 6.1

Under Rails 6.1 the old constant raises `NameError`; under Rails 6.0 the new one does not yet exist. A conditional is required during dual-boot.

```ruby
# app/services/log_sanitizer.rb
if NextRails.next?
  filter = ActiveSupport::ParameterFilter.new([:password, :token])
else
  filter = ActionDispatch::Http::ParameterFilter.new([:password, :token])
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

## When NOT to Branch: Deprecations

If the **new** API exists on the current version too, there is no breaking change — just replace the call site. Wrapping a deprecation in `NextRails.next?` adds dead branches you'll have to clean up for no benefit.

❌ **WRONG — `fixture_path` → `fixture_paths` (deprecation only):**

```ruby
# Both config.fixture_path= and config.fixture_paths= work on Rails 7.0 and 7.1.
# 7.1 only emits a deprecation warning for the old setter.
if NextRails.next?
  config.fixture_paths = ["#{::Rails.root}/spec/fixtures"]
else
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
end
```

✅ **CORRECT — just use the new API everywhere:**

```ruby
config.fixture_paths = ["#{::Rails.root}/spec/fixtures"]
```

Same rule applies to `serialize :preferences, coder: JSON` vs. the older positional form, `update` vs. `update_attributes` (prior to 6.1 removal), and most Rails API evolution: if both versions accept the new form, migrate call sites directly and skip the conditional.

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
