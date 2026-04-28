# Code Patterns for `NextRails.next?`

**Based on "The Complete Guide to Upgrade Rails" by FastRuby.io (OmbuLabs)**

Use `NextRails.next?` anywhere your application code must behave differently between the current and next dependency sets. This is the **only** acceptable way to branch on version — whether you are upgrading Rails, Ruby, or another core dependency.

**Only branch when the old code actually breaks on the next version.** A deprecation warning is not a reason for a conditional — if the new API works on both, replace the call site directly and let the deprecation disappear on its own. Reserve `NextRails.next?` for removed constants, removed methods, or incompatible signature/return-type changes that would raise an error otherwise.

The version numbers in the example below are from a Rails 4.2 → 5.0 upgrade, but the pattern applies any time a method's signature or availability changes incompatibly across versions — the same shape works for any adjacent-version boundary where each side raises on the other's call.

---

## The Foundational Case: Gemfile Version Pin

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

This is genuinely two-sided — the `7.1.0` pin fails resolution on the 7.0 side and vice versa — so the conditional is required for bundler to resolve at all. Every other use of `NextRails.next?` is secondary (Tier 2 or 3).

---

## Three-Tier Approach

When a version gap affects application code (not the Gemfile), pick the **lowest tier that works**. Lower tiers are cheaper to read, cheaper to clean up, and do not leave dead branches behind. Climb tiers only when the lower option actually breaks.

| Tier | Technique | When |
|------|-----------|------|
| 1 | Unconditional migration — use the new API on both sides. | Both Rails versions accept the new form (the typical Rails deprecation case). |
| 2 | `NextRails.next?` conditional at the call site. | New API doesn't exist on current (or old API raises on next). Smaller gaps (up to ~20 call sites). |
| 3 | Backport / forwardport shim — monkey-patch in one file. | Same gap spans an application layer (all controllers, all mailers) or affects ~20+ call sites. |

**Cleanup at the end of the dual-boot is mechanical in all three tiers:**
- Tier 1 — nothing to clean; the code is already unconditional.
- Tier 2 — drop `else` branches, keep the `if NextRails.next?` body.
- Tier 3 — delete the shim file.

See `workflows/cleanup-workflow.md` for the full cleanup procedure.

---

### Tier 1: Unconditional Migration (preferred)

If the **new** API exists on the current version too, there is no breaking change — just replace the call site. Wrapping a deprecation in `NextRails.next?` adds dead branches you'll have to clean up for no benefit. This is the case for almost every Rails-core deprecation at an adjacent-minor boundary because Rails ships the replacement one version before it removes the old form.

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

### Tier 2: `NextRails.next?` Conditional (for a few call sites)

Reach for a conditional when the gap is genuinely two-sided — the new API doesn't exist on current, or the old one raises on next — and the affected call sites are few (up to ~20). Common triggers:

- **A third-party gem tied to the Rails version has a two-sided API break.** Rails versions often pull in different majors of gems like Rack, minitest, or rspec-rails, or a Rails major replaces a gem-provided feature with a native API using different syntax.
- **You treat deprecations as errors** via `ActiveSupport::Deprecation.behavior = :raise`. A deprecated-but-still-working Rails call now raises on the next side, so the branch lets you use the new API on next and keep the old form on current.
- **The Rails upgrade also forces a Ruby upgrade** with stdlib extractions or syntax differences.

**Example — `ignorable` gem → native `ignored_columns=` (Rails 4.2 → 5.0):**

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

Put the next-version branch on top so cleanup is mechanical: after the upgrade, keep the `if` body and drop the `else`.

---

### Tier 3: Backport / Forwardport Shim (for many call sites)

When a version gap spans an application layer (all controllers, all mailers, all models with the same concern) or affects ~20+ call sites, a single monkey-patch in an initializer (for runtime gaps) or `test_helper.rb` / `spec_helper.rb` (for test-only gaps) lets all call sites stay identical on both sides. The shim makes the lagging version behave like the other one — usually by redefining a method to delegate to an existing equivalent. This is cleaner than scattering dozens of `NextRails.next?` branches and much simpler to remove at cleanup: delete the shim file.

**Two directions:**
- **Backport** — teach the *current* (older) version to behave like the next one, so all call sites are written using the new shape. More common because the new shape is usually the one you want to keep after cleanup.
- **Forwardport** — teach the *next* (newer) version to behave like the current one, so existing call sites keep working unchanged while you migrate them gradually.

Pick whichever direction lets the shared call-site code read best.

**Example — `ActionController::Parameters#values` backport (Rails 7.0 → 7.1):**

Rails 7.1 changed [`ActionController::Parameters#values`](https://github.com/rails/rails/pull/44816) to return an array of `ActionController::Parameters` objects instead of an array of plain hashes. Any controller or service that calls `params.values` and treats the result as hashes will behave differently between the two sides. With many such call sites, a shim is cleaner than scattering conditionals:

```ruby
# config/initializers/ac_parameters_values_backport.rb

# Backport of Rails 7.1's ActionController::Parameters#values behavior so that
# call sites on the Rails 7.0 side also receive an array of
# ActionController::Parameters (not plain hashes).
# Delete this file once the upgrade to Rails 7.1 is complete.
# Reference: https://github.com/rails/rails/pull/44816

if !NextRails.next?
  module ActionController
    class Parameters
      def values
        to_enum(:each_value).to_a
      end
    end
  end
end
```

With the shim loaded, every `params.values` call across the app returns the 7.1 shape on both sides — no per-call-site branching. After the upgrade, delete `ac_parameters_values_backport.rb` and nothing else needs to change.

**Caveats for Tier 3:**
- Monkey-patches affect the whole process. Keep the shim narrow (one or two methods), well-commented with a removal marker, and guarded with `if !NextRails.next?` (for a backport) or `if NextRails.next?` (for a forwardport) so it only loads on the side that needs it.
- If the signature difference between versions is severe enough that a faithful shim isn't possible, fall back to Tier 2 — a conditional at each call site is clearer than a lossy shim.

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
