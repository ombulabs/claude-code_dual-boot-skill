# Deprecation Tracking with Dual-Boot

**Ensuring deprecation warnings are visible and tracked during upgrades**

---

## Why This Matters

Deprecation warnings are your roadmap during a Rails upgrade — they tell you exactly what will break in the next version. If deprecations are silenced, you're flying blind. Before setting up dual-boot, you must ensure deprecation warnings are visible and tracked.

---

## Step 1: Detect If Deprecations Are Silenced

Check these files for silenced deprecations:

### Check `config/environments/test.rb`

```ruby
# BAD — deprecations are invisible
config.active_support.deprecation = :silence

# BAD — all deprecation behaviors disabled (Rails 7.0+)
config.active_support.report_deprecations = false
```

### Check `config/environments/development.rb`

Same patterns as above — `:silence` or `report_deprecations = false`.

### Check for Global Silencing

Search for these patterns anywhere in the codebase:

```ruby
# Silences all deprecation warnings globally
ActiveSupport::Deprecation.silenced = true

# Silences within a block (may be acceptable in narrow cases)
ActiveSupport::Deprecation.silence { ... }
```

### Detection Commands

Search from the project root to catch configuration in any directory structure (standard Rails, Packwerk packs, engines, etc.):

```bash
# Find all deprecation configuration
grep -rn "active_support.deprecation" .
grep -rn "report_deprecations" .
grep -rn "Deprecation.silenced" .
grep -rn "Deprecation.silence" .
```

If any of these are silencing deprecations, fix them before proceeding with the upgrade.

---

## Step 2: Save Deprecation Inventory with DeprecationTracker

The `next_rails` gem includes `DeprecationTracker` which captures all deprecation warnings during test runs and saves them to a file. This gives you a complete inventory without stopping on the first failure (unlike `:raise`).

### Setup

Ensure `next_rails` is in your Gemfile at the **root level** (not inside a `group` block):

```ruby
# Gemfile
# MUST be at root level — not in :development or :test group
gem 'next_rails'
```

Then configure your test runner.

**RSpec** — in `spec/spec_helper.rb` (or `spec/rails_helper.rb`):

```ruby
RSpec.configure do |config|
  DeprecationTracker.track_rspec(
    config,
    shitlist_path: "spec/support/deprecation_warning.shitlist.json",
    mode: ENV.fetch("DEPRECATION_TRACKER", "save"),
    transform_message: -> (message) { message.gsub("#{Rails.root}/", "") }
  )
end
```

**Minitest** — in `test/test_helper.rb`:

```ruby
DeprecationTracker.track_minitest(
  shitlist_path: "test/support/deprecation_warning.shitlist.json",
  mode: ENV.fetch("DEPRECATION_TRACKER", "save"),
  transform_message: -> (message) { message.gsub("#{Rails.root}/", "") }
)
```

- **`shitlist_path`** — where to save/read the deprecation inventory (required, no default)
- **`mode`** — defaults to `save` so it always tracks
- **`transform_message`** — strips the Rails root path from messages so the shitlist is portable across environments

### Save the Deprecation Shitlist

Run the test suite with `DEPRECATION_TRACKER=save` to collect all deprecation warnings (substitute your project's test command — e.g. `bundle exec rspec`, `bin/rails test`, `bin/test`):

```bash
DEPRECATION_TRACKER=save bundle exec rspec
```

This generates `spec/support/deprecation_warning.shitlist.json` — a JSON file listing every unique deprecation warning found during the run.

> **Note:** Make sure the directory for `shitlist_path` exists before running.

### Prevent Regressions

Once you have a shitlist, run with `DEPRECATION_TRACKER=save` after fixing deprecations to update the file. As you fix deprecations, the shitlist shrinks. Any new deprecation not in the shitlist will cause a failure, preventing regressions.

### Workflow

1. `DEPRECATION_TRACKER=save bundle exec rspec` — generates initial shitlist
2. Review the shitlist to understand the scope of deprecations
3. Fix one category of deprecation (e.g., `update_attributes`)
4. `DEPRECATION_TRACKER=save bundle exec rspec` — updates the shitlist (it should shrink)
5. Repeat until the shitlist is empty
6. Commit the shitlist file so the team tracks progress together

### CI with Parallel Test Execution

`DeprecationTracker` does not natively support parallel execution. When CI splits tests across multiple nodes/containers (e.g., CircleCI parallelism, parallel_tests gem), each node only sees its subset of deprecations. You need to collect and merge the results.

**Strategy: Save per-node, merge after**

1. On each CI node, run with `DEPRECATION_TRACKER=save` as normal
2. Each node produces its own `spec/support/deprecation_warning.shitlist.json`
3. After all nodes finish, collect and merge the shitlist files:

```bash
# Example merge script (run after all parallel nodes complete)
# Collects shitlist JSON files from each node and merges them

require 'json'

shitlists = Dir.glob("node-*/spec/support/deprecation_warning.shitlist.json")
merged = {}

shitlists.each do |file|
  data = JSON.parse(File.read(file))
  data.each do |key, values|
    merged[key] ||= []
    merged[key] = (merged[key] + values).uniq
  end
end

File.write("spec/support/deprecation_warning.shitlist.json", JSON.pretty_generate(merged))
```

The exact collection mechanism depends on your CI setup:
- **CircleCI**: Use `persist_to_workspace` / `attach_workspace` to gather shitlists from parallel containers
- **GitHub Actions**: Use `upload-artifact` / `download-artifact` across matrix jobs
- **parallel_tests gem**: Each process writes to the same filesystem, so you can glob the results directly

> **Tip:** Run the initial `DEPRECATION_TRACKER=save` locally (non-parallel) to generate the first complete shitlist. Use the parallel merge strategy in CI for ongoing regression checks.

### Limitations

- **Not compatible with `minitest/parallel_fork`** — use standard minitest or RSpec
- **No native parallel support** — requires manual merge when using CI parallelism (see above)
- **Supports RSpec (`track_rspec`) and Minitest (`track_minitest`)** — other frameworks require manual integration

---

## Step 3: Custom Deprecation Behavior (Alternative Approach)

If `DeprecationTracker` doesn't fit your setup (e.g., Minitest with parallel_fork, or you want more control), you can use custom deprecation behaviors to selectively raise on fixed deprecations while logging the rest.

Based on [FastRuby.io's custom deprecation behavior approach](https://www.fastruby.io/blog/custom-deprecation-behavior.html):

### Rails 6.1+ (Built-in Support)

```ruby
# config/environments/test.rb

# Raise on specific deprecations you've already fixed (prevents regressions)
config.active_support.disallowed_deprecation = :raise
config.active_support.disallowed_deprecation_warnings = [
  /update_attributes/,          # Already migrated to .update
  /Merging .* no longer/,       # Already fixed merge conditions
  "before_filter",              # Already migrated to before_action
]

# Log everything else (so you can see remaining work)
config.active_support.deprecation = :log
```

### Rails 5.2+ (Backported via Custom Lambda)

```ruby
# config/initializers/custom_deprecation_behavior.rb
ActiveSupport::Deprecation.behavior = ->(message, callstack, deprecation_horizon, gem_name) {
  # Deprecations you've already fixed — raise if they reappear
  disallowed = [
    "update_attributes",
    /before_filter/,
  ]

  if disallowed.any? { |pattern|
    pattern.is_a?(Regexp) ? pattern === message : message.include?(pattern)
  }
    ActiveSupport::Deprecation::DEFAULT_BEHAVIORS[:raise].call(
      message, callstack, deprecation_horizon, gem_name
    )
  else
    ActiveSupport::Deprecation::DEFAULT_BEHAVIORS[:stderr].call(
      message, callstack, deprecation_horizon, gem_name
    )
  end
}
```

### Rails 4.0–5.1 (2-Argument Signature)

```ruby
# config/initializers/custom_deprecation_behavior.rb
ActiveSupport::Deprecation.behavior = ->(message, callstack) {
  disallowed = [
    "update_attributes",
    /before_filter/,
  ]

  if disallowed.any? { |pattern|
    pattern.is_a?(Regexp) ? pattern === message : message.include?(pattern)
  }
    ActiveSupport::Deprecation::DEFAULT_BEHAVIORS[:raise].call(message, callstack)
  else
    ActiveSupport::Deprecation::DEFAULT_BEHAVIORS[:stderr].call(message, callstack)
  end
}
```

### Rails 3.0–3.2 (No `:raise` Behavior)

```ruby
# config/initializers/custom_deprecation_behavior.rb
ActiveSupport::Deprecation.behavior = ->(message, callstack) {
  disallowed = [
    "update_attributes",
    /before_filter/,
  ]

  if disallowed.any? { |pattern|
    pattern.is_a?(Regexp) ? pattern === message : message.include?(pattern)
  }
    raise message
  end

  ActiveSupport::Deprecation::DEFAULT_BEHAVIORS[:stderr].call(message, callstack)
}
```

---

## Step 4: Maintenance

After completing the upgrade:

- If using DeprecationTracker: delete the shitlist file once it's empty
- If using custom behaviors: remove entries for deprecations that no longer exist in the new Rails version
- If you've fixed all deprecations, you can switch to `config.active_support.deprecation = :raise` for strict mode going forward

---

## Quick Reference

| What to check | Command |
|---|---|
| Silenced deprecations | `grep -rn "deprecation.*silence\|silenced.*true\|report_deprecations.*false" .` |
| Current behavior | `grep -rn "active_support.deprecation" .` |
| Custom behaviors | `grep -rn "Deprecation.behavior" .` |
| Save deprecation inventory | `DEPRECATION_TRACKER=save bundle exec rspec` |
| Existing warnings | `bundle exec rspec 2>&1 \| grep "DEPRECATION WARNING"` |
