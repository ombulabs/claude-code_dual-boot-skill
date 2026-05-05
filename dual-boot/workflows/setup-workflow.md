# Dual-Boot Setup Workflow

## Prerequisites

- A Ruby application with a working `Gemfile` and `Gemfile.lock`
- Know the current and target versions for the dependency you are upgrading (Rails, Ruby, or another core gem)

---

## Step 0: Verify Deprecation Warnings Are Not Silenced

Before setting up dual-boot, ensure deprecation warnings are visible. Silenced deprecations mean you can't track upgrade progress.

**Check for silenced deprecations:**

`config/` is not the only place this gets configured. RSpec autoloads `-r` requires from `.rspec`, and test helpers can install hooks like `RSpec.configure { |c| c.raise_errors_for_deprecations! }`. Sweep these locations as well:

- `.rspec`, `spec/.rspec` (look for `-r raise_errors_for_deprecations` and similar autoloads)
- `spec/spec_helper.rb`, `spec/rails_helper.rb`, `spec/support/**/*.rb`
- `Rakefile`, `lib/tasks/*.rake`
- Any Bundler-loaded test-group gem that wires deprecation behavior

```bash
grep -rnE "raise_errors_for_deprecations|ActiveSupport::Deprecation\.(behavior|disallowed_behavior|silenced)\s*=|ActiveSupport::Deprecation\.silence\b|deprecation.*silence|silenced.*true|report_deprecations.*false" \
  .rspec spec/.rspec spec/spec_helper.rb spec/rails_helper.rb Rakefile \
  config/ spec/support/ lib/tasks/ 2>/dev/null
```

The `silence\b` alternation catches the block form `ActiveSupport::Deprecation.silence do ... end`. The `\b` word boundary keeps it from matching `silenced` (already covered by the assignment branch).

**If deprecations are silenced:**
1. Change `:silence` to `:stderr` or `:log` (or `:raise` if you want strict mode)
2. Remove `ActiveSupport::Deprecation.silenced = true` if found
3. Remove `config.active_support.report_deprecations = false` if found (Rails 7.0+)
4. Review any `Deprecation.silence do ... end` blocks. Each one hides a deprecation that should be triaged rather than suppressed.
5. Review any `raise_errors_for_deprecations` autoload. Useful in strict mode, but worth knowing it's installed before interpreting test failures.
6. Run the test suite to see current deprecation warnings

See `references/deprecation-tracking.md` for detailed configuration guidance, including a gradual approach using custom deprecation behaviors for apps with many existing warnings.

---

## Step 1: Check if Dual-Boot is Already Set Up

```bash
# Check if Gemfile.next already exists
ls -la Gemfile.next
```

- If `Gemfile.next` exists → Skip to Step 4
- If `Gemfile.next` does NOT exist → Continue to Step 2

**⚠️ CRITICAL:** Running `next_rails --init` when dual-boot is already set up will duplicate the `next?` method definition in the Gemfile, causing errors.

---

## Step 2: Add `next_rails` Gem

Add the gem to the Gemfile **at the root level, not inside any group**:

```ruby
# Gemfile
gem 'next_rails'
```

**⚠️ If the gem is already present, verify its location.** Grep for it:

```bash
grep -n "next_rails" Gemfile
```

If it lives inside a `group :development`, `group :test`, or `group :development, :test do ... end` block, move it outside. Any `NextRails.next?` in `config/environments/production.rb` or `config/application.rb` will raise `NameError: uninitialized constant NextRails` when the app boots in production (e.g., during `assets:precompile RAILS_ENV=production`). The gem must be loadable in every environment where conditional config runs.

Then install:

```bash
bundle install
```

---

## Step 3: Initialize Dual-Boot

```bash
next_rails --init
```

This creates:
- `Gemfile.next` — Symlink to your Gemfile
- `Gemfile.next.lock` — Lock file for the next dependency set

---

## Step 4: Configure Gemfile with Version Conditionals

The `next_rails` gem adds a `next?` helper method to your Gemfile. Use it to specify different versions of the dependency you are upgrading.

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

### Handling Related Gem Version Differences

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

---

## Step 5: Install Dependencies for Both Versions

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

---

## Step 6: Verify Both Dependency Sets Work

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

If the project uses `rake test` instead of `bin/rails test`, or a parallel runner (`parallel_rspec`, `parallel_test`, `turbo_tests`), substitute that binary — the `BUNDLE_GEMFILE=Gemfile.next` prefix (or the `next` CLI) works with any of them.

---

## Step 7: Set Up Deprecation Tracking with DeprecationTracker

The `next_rails` gem includes `DeprecationTracker`, which captures all deprecation warnings during test runs and saves them to a JSON file. Set it up now so you have a complete deprecation inventory from the start.

### Configure RSpec

Add the following to `spec/spec_helper.rb` (or `spec/rails_helper.rb`):

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

### Generate the Initial Deprecation Inventory

```bash
DEPRECATION_TRACKER=save bundle exec rspec
```

This creates `spec/support/deprecation_warning.shitlist.json` — a JSON file listing every unique deprecation warning found during the run. Review it to understand the scope of deprecations you need to address.

### Minitest

For Minitest apps, use `DeprecationTracker.track_minitest` in `test/test_helper.rb`:

```ruby
DeprecationTracker.track_minitest(
  shitlist_path: "test/support/deprecation_warning.shitlist.json",
  mode: ENV.fetch("DEPRECATION_TRACKER", "save"),
  transform_message: -> (message) { message.gsub("#{Rails.root}/", "") }
)
```

Then generate the inventory with `DEPRECATION_TRACKER=save bin/rails test`. If `track_minitest` doesn't fit your setup (e.g., `minitest/parallel_fork`), fall back to the custom deprecation behavior approach in `references/deprecation-tracking.md` (Step 3).

See `references/deprecation-tracking.md` for the full workflow (updating the shitlist, preventing regressions, CI with parallel execution, and alternative approaches).

---

## Step 8: Commit Dual-Boot Setup

```bash
git add Gemfile Gemfile.next Gemfile.next.lock
git commit -m "Add dual-boot setup for upgrade"
```

---

## Next Steps

- Configure CI to test both versions (see `references/ci-configuration.md`)
- Start fixing breaking changes using `NextRails.next?` branching
- See `references/code-patterns.md` for code examples
