# Deprecation Tracking with Dual-Boot

**Ensuring deprecation warnings are visible and tracked during upgrades**

---

## Why This Matters

Deprecation warnings are your roadmap during a Rails upgrade. They tell you exactly what will break in the next version. If deprecations are silenced, you're flying blind. Before setting up dual-boot, you must ensure deprecation warnings are visible and tracked.

---

## Step 1: Detect If Deprecations Are Silenced

Check these files for silenced deprecations:

### Check `config/environments/test.rb`

```ruby
# BAD (deprecations are invisible)
config.active_support.deprecation = :silence

# BAD (all deprecation behaviors disabled, Rails 7.0+)
config.active_support.report_deprecations = false
```

### Check `config/environments/development.rb`

Same patterns as above: `:silence` or `report_deprecations = false`.

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

> **Minimum `next_rails` version: 1.5.0** (released 2026-04-02). The parallel-CI-aware `node_index:` option and the `deprecations merge` CLI require v1.5.0. If your Gemfile currently pins an older version, update it before proceeding.

### Setup

Ensure `next_rails` is in your Gemfile at the **root level** (not inside a `group` block), with the version constraint:

```ruby
# Gemfile
# MUST be at root level, not in :development or :test group
gem 'next_rails', '>= 1.5.0'
```

Then configure your test runner.

**RSpec**, in `spec/spec_helper.rb` (or `spec/rails_helper.rb`):

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

**Minitest**, in `test/test_helper.rb`:

```ruby
DeprecationTracker.track_minitest(
  shitlist_path: "test/support/deprecation_warning.shitlist.json",
  mode: ENV.fetch("DEPRECATION_TRACKER", "save"),
  transform_message: -> (message) { message.gsub("#{Rails.root}/", "") }
)
```

- **`shitlist_path`**: where to save/read the deprecation inventory (required, no default)
- **`mode`**: defaults to `save` so it always tracks
- **`transform_message`**: strips the Rails root path from messages so the shitlist is portable across environments

### Save the Deprecation Shitlist

Run the test suite with `DEPRECATION_TRACKER=save` to collect all deprecation warnings (substitute your project's test command, e.g. `bundle exec rspec`, `bin/rails test`, `bin/test`):

```bash
DEPRECATION_TRACKER=save bundle exec rspec
```

This generates `spec/support/deprecation_warning.shitlist.json`, a JSON file listing every unique deprecation warning found during the run.

> **Note:** Make sure the directory for `shitlist_path` exists before running.

### Prevent Regressions

Once you have a shitlist, run with `DEPRECATION_TRACKER=save` after fixing deprecations to update the file. As you fix deprecations, the shitlist shrinks. Any new deprecation not in the shitlist will cause a failure, preventing regressions.

### Workflow

1. `DEPRECATION_TRACKER=save bundle exec rspec` generates the initial shitlist
2. Review the shitlist to understand the scope of deprecations
3. Fix one category of deprecation (e.g., `update_attributes`)
4. `DEPRECATION_TRACKER=save bundle exec rspec` updates the shitlist (it should shrink)
5. Repeat until the shitlist is empty
6. Commit the shitlist file so the team tracks progress together

### CI with Parallel Test Execution

Starting with `next_rails` v1.5.0, `DeprecationTracker` has native support for parallel CI. When CI splits tests across multiple nodes or containers (CircleCI parallelism, Buildkite, GitLab, Semaphore, GitHub Actions matrix), each node writes its own shard file. A one-off merge step fans the shards back into the canonical shitlist.

#### Configure `node_index:`

Pass a `node_index:` value to `track_rspec` or `track_minitest`. When set, the tracker writes to `<shitlist_path>.node-<index>.json` instead of the canonical file, so parallel nodes don't clobber each other.

You can pass `node_index:` explicitly, or omit it and rely on auto-detection from standard CI environment variables:

| CI platform | Env var auto-detected |
|---|---|
| CircleCI | `CIRCLE_NODE_INDEX` |
| Buildkite | `BUILDKITE_PARALLEL_JOB` |
| GitLab CI | `CI_NODE_INDEX` |
| Semaphore | `SEMAPHORE_JOB_INDEX` |
| Generic | `CI_NODE_INDEX` |

**RSpec** with explicit `node_index:`:

```ruby
RSpec.configure do |config|
  if ENV["DEPRECATION_TRACKER"]
    DeprecationTracker.track_rspec(
      config,
      shitlist_path: "spec/support/deprecation_warning.shitlist.json",
      mode: ENV.fetch("DEPRECATION_TRACKER", "save"),
      node_index: ENV["CI_NODE_INDEX"],
      transform_message: -> (message) { message.gsub("#{Rails.root}/", "") }
    )
  end
end
```

If your CI sets one of the env vars in the table above, you can drop the `node_index:` argument entirely and let auto-detection handle it.

**Minitest** with the same pattern:

```ruby
DeprecationTracker.track_minitest(
  shitlist_path: "test/support/deprecation_warning.shitlist.json",
  mode: ENV.fetch("DEPRECATION_TRACKER", "save"),
  node_index: ENV["CI_NODE_INDEX"],
  transform_message: -> (message) { message.gsub("#{Rails.root}/", "") }
)
```

#### Three-phase CI workflow

**Save phase** (each parallel node writes a shard):

```bash
DEPRECATION_TRACKER=save CI_NODE_INDEX=$NODE bundle exec rspec <subset>
```

Each node produces a file like `spec/support/deprecation_warning.shitlist.node-0.json`, `...node-1.json`, etc.

**Merge phase** (runs once after all nodes finish, with shards gathered into one workspace):

```bash
deprecations merge --delete-shards
```

This fans all shards into the canonical `spec/support/deprecation_warning.shitlist.json` and removes the per-node files. To merge the next-Rails shitlist (the shards produced when running under the next Rails version in dual-boot), pass `--next`:

```bash
deprecations merge --next --delete-shards
```

For custom flows, call the programmatic form directly:

```ruby
DeprecationTracker.merge_shards(
  "spec/support/deprecation_warning.shitlist.json",
  delete_shards: true
)
```

**Compare phase** (each parallel node compares its subset against the merged canonical file):

```bash
DEPRECATION_TRACKER=compare CI_NODE_INDEX=$NODE bundle exec rspec <subset>
```

The shard/canonical split means compare mode works per-node without false positives: each node only checks deprecations it actually emits, and the canonical shitlist already contains every deprecation from every node.

#### Gathering shards between phases

The exact mechanism to move shards from save-phase nodes into the merge job depends on your CI:

- **CircleCI**: use `persist_to_workspace` in each parallel container, `attach_workspace` in the merge job
- **GitHub Actions**: use `upload-artifact` from each matrix job, `download-artifact` in the merge job
- **Buildkite**: use pipeline artifacts (`buildkite-agent artifact upload` / `download`)
- **GitLab CI**: use job artifacts with `dependencies:` in the merge job
- **parallel_tests gem (single host)**: each process writes to the same filesystem, so the merge step can run immediately without any artifact shuffle

> **Tip:** Run the initial `DEPRECATION_TRACKER=save` locally without `node_index:` to generate the first complete canonical shitlist. Use the parallel sharded workflow in CI for ongoing regression checks.

### Limitations

- **Not compatible with `minitest/parallel_fork`.** Use standard Minitest, RSpec, or the custom deprecation behavior approach below.
- **Requires `next_rails` >= 1.5.0** for native parallel CI support. Older versions need a manual merge script.
- **Supports RSpec (`track_rspec`) and Minitest (`track_minitest`).** Other frameworks require manual integration.

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
  # Deprecations you've already fixed, raise if they reappear
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
| Merge parallel shards | `deprecations merge --delete-shards` |
| Existing warnings | `bundle exec rspec 2>&1 \| grep "DEPRECATION WARNING"` |

---

## Last-resort fallback

If `DeprecationTracker` won't install or configure on your app (rare, see Limitations), you can capture deprecations by tailing the test log:

```bash
bundle exec rspec 2>&1 | grep "DEPRECATION WARNING" | sort -u
```

This loses structure, compare mode, and regression gating. Use `DeprecationTracker` when possible.
