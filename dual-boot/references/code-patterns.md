# Code Patterns for `NextRails.next?`

**Based on "The Complete Guide to Upgrade Rails" by FastRuby.io (OmbuLabs)**

Use `NextRails.next?` anywhere your application code must behave differently between the current and next dependency sets. This is the **only** acceptable way to branch on version — whether you are upgrading Rails, Ruby, or another core dependency.

**Only branch when the old code actually breaks on the next version.** A deprecation warning is not a reason for a conditional — if the new API works on both, replace the call site directly and let the deprecation disappear on its own. Reserve `NextRails.next?` for removed constants, removed methods, or incompatible signature/return-type changes that would raise an error otherwise.

The version numbers in the example below are from a Rails 4.2 → 5.0 upgrade, but the pattern applies any time a gem-provided API is replaced by a native Rails API with different syntax (or vice versa) — the same shape works for any adjacent-version boundary where dropping the gem on one side makes the call unavailable.

---

## Rails API Changes Requiring a Conditional

### `ignorable` gem → native `ignored_columns=` (Rails 4.2 → 5.0)

An app using the [`ignorable`](https://github.com/nthj/ignorable) gem to ignore columns on Rails 4.2 hits a genuine two-sided case when upgrading to 5.0, which introduced a native `ignored_columns=` with different syntax. If the gem is dropped on the 5.0 side (since Rails now has the feature), `ignore_columns :category` raises `NoMethodError` there; `self.ignored_columns += [:category]` raises `NoMethodError` on 4.2 where the setter doesn't exist yet.

```ruby
# app/models/project.rb
class Project < ActiveRecord::Base
  if NextRails.next?
    self.ignored_columns += [:category]
  else
    ignore_columns :category
  end
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
