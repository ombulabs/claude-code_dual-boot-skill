# Dual-Boot Setup Workflow

This workflow is split into two phases:

- **Phase 1 (Preparatory):** Install `next_rails`, set up `DeprecationTracker`, and resolve current-version deprecations unconditionally (Tier 1). No Rails version hop yet. Both `Gemfile` and `Gemfile.next` resolve to identical gems.
- **Phase 2 (Dual-Boot):** Pin a different Rails version in the `if next?` Gemfile block, install gems on the next side, and fix genuine two-sided breakage using Tier 2 or Tier 3 patterns.

Keeping deprecation resolution in Phase 1 means most deprecation fixes land as plain, unconditional migrations on the current Rails version, without any `NextRails.next?` branching. Phase 2 is then scoped to the real two-sided breakage that only dual-boot can address.

## Prerequisites

- A Ruby application with a working `Gemfile` and `Gemfile.lock`.
- Know the current and target versions for the dependency you are upgrading (Rails, Ruby, or another core gem).
- **Minimum `next_rails` version: 1.5.0** (released 2026-04-02). The parallel-CI-aware `DeprecationTracker` configuration (`node_index:` option) and the `deprecations merge --delete-shards` CLI require v1.5.0. If your Gemfile currently pins an older version, update it before proceeding.

---

# Phase 1: Preparatory

Phase 1 is everything you do **before** the Rails version hop. At the end of Phase 1 you have `next_rails` installed, `Gemfile.next` in place as a symlink (both sides resolve to the same gems), `DeprecationTracker` configured, and all current-side deprecations fixed unconditionally. No `NextRails.next?` conditionals appear in application code during this phase.

## Step 1: Verify Deprecation Warnings Are Not Silenced

Before setting up dual-boot, ensure deprecation warnings are visible and tracked. See `references/deprecation-tracking.md` for the full procedure (detection of silenced deprecations, how to unsilence, and setting up `DeprecationTracker` to capture a regression-safe inventory).

## Step 2: Check if Dual-Boot is Already Set Up

```bash
# Check if Gemfile.next already exists
ls -la Gemfile.next
```

- If `Gemfile.next` exists, skip Steps 3 and 4 (gem install and `next_rails --init`) and proceed to Step 5.
- If `Gemfile.next` does not exist, continue to Step 3.

**CRITICAL:** Running `next_rails --init` when dual-boot is already set up will duplicate the `next?` method definition in the Gemfile, causing errors.

## Step 3: Add the `next_rails` Gem

Add the gem to the Gemfile **at the root level, not inside any group**:

```ruby
# Gemfile
gem 'next_rails'
```

**If the gem is already present, verify its location.** Grep for it:

```bash
grep -n "next_rails" Gemfile
```

If it lives inside a `group :development`, `group :test`, or `group :development, :test do ... end` block, move it outside. Any `NextRails.next?` in `config/environments/production.rb` or `config/application.rb` will raise `NameError: uninitialized constant NextRails` when the app boots in production (for example, during `assets:precompile RAILS_ENV=production`). The gem must be loadable in every environment where conditional config runs.

Then install:

```bash
bundle install
```

## Step 4: Initialize Dual-Boot

```bash
next_rails --init
```

This creates:

- `Gemfile.next`, a symlink to your `Gemfile`.
- `Gemfile.next.lock`, the lockfile for the next dependency set.

At this point both sides resolve to identical gems. The infrastructure is in place, but no version hop has happened yet.

## Step 5: Configure `DeprecationTracker`

`DeprecationTracker` (shipped with `next_rails`) captures every deprecation warning emitted during a test run and saves it to a JSON shitlist. Setting it up **before** the first test run means a single run captures the full inventory, avoiding a double pass.

Before choosing a configuration, ask the user:

> Do you run tests in CI with parallel execution (multiple nodes/containers running subsets of tests)?

Pick the matching configuration below.

### No / local only / single-process CI

**RSpec**, add to `spec/spec_helper.rb` (or `spec/rails_helper.rb`):

```ruby
RSpec.configure do |config|
  DeprecationTracker.track_rspec(
    config,
    shitlist_path: "spec/support/deprecation_warning.shitlist.json"
  )
end
```

**Minitest**, add to `test/test_helper.rb`:

```ruby
DeprecationTracker.track_minitest(
  shitlist_path: "test/support/deprecation_warning.shitlist.json"
)
```

### Yes, parallel CI

Set `node_index:` to the environment variable your CI provider uses for the per-shard index (`CI_NODE_INDEX`, `CIRCLE_NODE_INDEX`, `BUILDKITE_PARALLEL_JOB`, `SEMAPHORE_JOB_INDEX`, and so on).

**RSpec:**

```ruby
RSpec.configure do |config|
  DeprecationTracker.track_rspec(
    config,
    shitlist_path: "spec/support/deprecation_warning.shitlist.json",
    node_index: ENV["CI_NODE_INDEX"]  # or CIRCLE_NODE_INDEX, BUILDKITE_PARALLEL_JOB, etc.
  )
end
```

**Minitest:**

```ruby
DeprecationTracker.track_minitest(
  shitlist_path: "test/support/deprecation_warning.shitlist.json",
  node_index: ENV["CI_NODE_INDEX"]
)
```

With `node_index:` set, each shard writes `deprecation_warning.shitlist.node-<N>.json` instead of the canonical file. After all nodes finish, a fan-in step runs:

```bash
deprecations merge --delete-shards
```

This merges every `node-<N>.json` shard into the canonical shitlist and removes the shard files. See `references/deprecation-tracking.md` for the full CI pipeline (save, merge, compare phases).

## Step 6: Capture the Deprecation Inventory

Run the test suite on current Rails with `DEPRECATION_TRACKER=save` to generate the initial shitlist:

```bash
DEPRECATION_TRACKER=save bundle exec rspec
```

(Substitute your project's test command: `bin/rails test`, `bin/test`, `bundle exec parallel_rspec spec/`, and so on. The `DEPRECATION_TRACKER=save` environment variable is what matters.)

This creates `spec/support/deprecation_warning.shitlist.json`, a JSON file listing every unique deprecation warning found during the run. Review it to understand the scope of deprecations you need to address.

## Step 7: Fix Current-Side Deprecations Unconditionally (Tier 1)

Rails uses a deprecate-then-remove policy: the replacement API ships one version before the old form is removed. That means the new call almost always works on both the current and next sides, and the correct fix is Tier 1, an unconditional migration. **Do not wrap deprecation fixes in `NextRails.next?`**; that produces a dead branch you will only have to clean up later.

Workflow:

1. Pick a category of deprecation from the shitlist (for example, `update_attributes` → `update`).
2. Replace every call site with the new API, on the current Rails version.
3. Re-run `DEPRECATION_TRACKER=save bundle exec rspec`. The shitlist should shrink.
4. Commit the fix.
5. Repeat until the shitlist is empty or down to deprecations that genuinely cannot be resolved on current Rails.

See `references/code-patterns.md` "Tier 1: Unconditional Migration" for the underlying rule and examples.

---

# Phase 2: Dual-Boot

Phase 2 is the Rails version hop. You now pin a different Rails version on the next side, install gems on that side, and fix genuine two-sided breakage that cannot be resolved by Tier 1.

## Step 8: Configure the Gemfile with the Version Conditional

`next_rails` adds a `next?` helper to your Gemfile. Use it to pin a different version of the dependency you are upgrading.

**Do not dual-boot `config.load_defaults`.** It is post-upgrade work (rails-upgrade Step 10 / rails-load-defaults skill).

### Example: Rails version upgrade

```ruby
# Gemfile

def next?
  File.basename(__FILE__) == "Gemfile.next"
end

if next?
  gem 'rails', '~> 7.1.0'
else
  gem 'rails', '~> 7.0.0'
end
```

### Example: Ruby version upgrade

```ruby
# Gemfile

def next?
  File.basename(__FILE__) == "Gemfile.next"
end

if next?
  ruby '3.3.0'
else
  ruby '3.2.0'
end
```

### Example: Core dependency upgrade

```ruby
# Gemfile

def next?
  File.basename(__FILE__) == "Gemfile.next"
end

if next?
  gem 'sidekiq', '~> 7.0'
else
  gem 'sidekiq', '~> 6.5'
end
```

### Handling related gem version differences

If other gems also need different versions to stay compatible:

```ruby
if next?
  gem 'rails', '~> 7.1.0'
  gem 'activeadmin', '~> 3.0'
else
  gem 'rails', '~> 7.0.0'
  gem 'activeadmin', '~> 2.9'
end
```

See `references/gemfile-examples.md` for more patterns.

## Step 9: Identify and Pin Incompatible Gems

`BUNDLE_GEMFILE=Gemfile.next bundle install` often fails because the target Rails version pulls in incompatible majors of other gems (Rack, sprockets, and so on). Surface these conflicts **before** attempting the next-side install.

1. **Run `bundle_report compatibility` up front.** `next_rails` ships a `bundle_report` CLI that flags which gems need version pinning for the target Rails:

   ```bash
   bundle_report compatibility --rails-version=<target_rails_version>
   ```

   Example: `bundle_report compatibility --rails-version=7.1`. The output lists gems whose current pins are incompatible with the target Rails, along with known-compatible versions.

2. **If conflicts still arise during `BUNDLE_GEMFILE=Gemfile.next bundle install`**, read bundler's conflict output to identify the offending gem.

3. **Look up a compatible version** via [RailsBump](https://railsbump.org/) or the gem's CHANGELOG.

4. **Pin the next-compatible version inside the `if next?` block:**

   ```ruby
   if next?
     gem 'rails', '~> 7.1.0'
     gem 'some_gem', '~> 3.0'
   else
     gem 'rails', '~> 7.0.0'
     gem 'some_gem', '~> 2.5'
   end
   ```

5. **Re-run** `BUNDLE_GEMFILE=Gemfile.next bundle install` until it resolves cleanly.

## Step 10: Install Dependencies for Both Versions

```bash
# Install current version dependencies
bundle install

# IMPORTANT: Copy Gemfile.lock BEFORE running the next bundle install.
# Without this, bundler would resolve all dependencies from scratch and may
# upgrade unrelated gems. By copying first, only gems in `if next?` blocks will change.
cp Gemfile.lock Gemfile.next.lock

# Install next version dependencies
BUNDLE_GEMFILE=Gemfile.next bundle install
```

If `BUNDLE_GEMFILE=Gemfile.next bundle install` fails with a resolution error, loop back to Step 9.

## Step 11: Verify Both Dependency Sets Boot and Run Tests

Use the project's own test runner. Detect it first: `rspec-rails` in the Gemfile and a `spec/` directory → RSpec; `test/` and no `spec/` → Minitest; a `bin/test` wrapper → prefer it; `parallel_tests` / `turbo_tests` → use their binaries.

### RSpec

```bash
# Run tests with current version
bundle exec rspec

# Run tests with next version
BUNDLE_GEMFILE=Gemfile.next bundle exec rspec
```

### Minitest

```bash
# Run tests with current version
bin/rails test

# Run tests with next version
BUNDLE_GEMFILE=Gemfile.next bin/rails test
```

If the project uses `rake test` instead of `bin/rails test`, or a parallel runner (`parallel_rspec`, `parallel_test`, `turbo_tests`), substitute that binary. The `BUNDLE_GEMFILE=Gemfile.next` prefix (or the `next` CLI) works with any of them.

## Step 12: Fix Two-Sided Breakage (Tier 2 / Tier 3)

With current-side deprecations already resolved in Phase 1, anything still failing on the next side is genuine two-sided breakage: an API gone on the next side whose replacement does not backport to current, a behavior change between the two versions, or a gem-version gap where the gems pinned on each side expose different APIs.

Use `references/code-patterns.md` "Three-Tier Approach" to decide how to handle each one:

- **Tier 2**, a `NextRails.next?` conditional at the call site. Fits small gaps (up to about 20 call sites).
- **Tier 3**, a backport or forwardport shim in a single file. Fits gaps that span an application layer (all controllers, all mailers) or 20+ call sites. Cleanup is a single-file delete.

Do not fix deprecation warnings emitted by the **next** version here. Those belong to the preparatory phase of the _following_ upgrade (current-version deprecations for that hop), not this one's dual-boot work.

## Step 13: Commit Dual-Boot Setup

```bash
git add Gemfile Gemfile.next Gemfile.next.lock
git commit -m "Add dual-boot setup for upgrade"
```

Commit Tier 2 and Tier 3 fixes separately as you make them, ideally one per category.

---

## Next Steps

- **CI is optional.** Local-only dual-boot is fully valid. Running both suites manually (`bundle exec rspec` and `BUNDLE_GEMFILE=Gemfile.next bundle exec rspec`) is a complete workflow. `references/ci-configuration.md` is a pointer for readers who want to automate, not a mandate.
- Continue fixing breaking changes using the three-tier approach in `references/code-patterns.md`.
- When the upgrade is complete, remove dual-boot scaffolding via `workflows/cleanup-workflow.md` (Tier 2: drop `else` branches; Tier 3: delete shim files; Gemfile: collapse the `if next?` block).
