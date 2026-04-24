# Code Patterns for `NextRails.next?`

**Based on "The Complete Guide to Upgrade Rails" by FastRuby.io (OmbuLabs)**

Use `NextRails.next?` anywhere your application code must behave differently between the current and next dependency sets. This is the **only** acceptable way to branch on version — whether you are upgrading Rails, Ruby, or another core dependency.

**Only branch when the old code actually breaks on the next version.** A deprecation warning is not a reason for a conditional — if the new API works on both, replace the call site directly and let the deprecation disappear on its own. Reserve `NextRails.next?` for removed constants, removed methods, or incompatible signature/return-type changes that would raise an error otherwise.

The version numbers in the example below are from a Rails 4.2 → 5.0 upgrade, but the pattern applies any time a method's signature or availability changes incompatibly across versions — the same shape works for any adjacent-version boundary where each side raises on the other's call.

---

## Rails API Changes Requiring a Conditional

### `ActionController::TestRequest.new` → `.create` (Rails 4.2 → 5.0)

On Rails 4.2, [`ActionController::TestRequest.new`](https://github.com/rails/rails/blob/11f2bdf75a888682b34df0f9be03b94f54fc6796/actionpack/lib/action_controller/test_case.rb#L201) takes an optional `env` argument. On Rails 5.0, `new` [requires two non-optional arguments](https://github.com/rails/rails/blob/c4d3e202e10ae627b3b9c34498afb45450652421/actionpack/lib/action_controller/test_case.rb#L48), so the 4.2-style call raises `ArgumentError`. Rails 5.0 introduced `TestRequest.create` to hide that complexity — but `.create` does not exist on 4.2 (`NoMethodError`). Each side raises on the other's call.

```ruby
test_request =
  if NextRails.next?
    ActionController::TestRequest.create
  else
    ActionController::TestRequest.new
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

Same rule applies to `serialize :preferences, coder: JSON` vs. the older positional form, `update` vs. `update_attributes` (prior to the 6.1 removal), and most Rails API evolution: if both versions accept the new form, migrate call sites directly and skip the conditional.

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
