# Dual-Boot Setup Workflow

## Prerequisites

- A Ruby application with a working `Gemfile` and `Gemfile.lock`
- Know the current and target versions for the dependency you are upgrading (Rails, Ruby, or another core gem)

---

## Step 0: Verify Deprecation Warnings Are Not Silenced

Before setting up dual-boot, ensure deprecation warnings are visible. Silenced deprecations mean you can't track upgrade progress.

**Check for silenced deprecations:**

```bash
grep -rn "deprecation.*silence\|silenced.*true\|report_deprecations.*false" config/
```

**If deprecations are silenced:**
1. Change `:silence` to `:stderr` or `:log` (or `:raise` if you want strict mode)
2. Remove `ActiveSupport::Deprecation.silenced = true` if found
3. Remove `config.active_support.report_deprecations = false` if found (Rails 7.0+)
4. Run the test suite to see current deprecation warnings

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

Add the gem to the Gemfile (all environments, not just development):

```ruby
# Gemfile
gem 'next_rails'
```

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

# Install next version dependencies
next bundle install
```

If `next bundle install` does not work (e.g., the `next` command is not found in PATH), use:

```bash
BUNDLE_GEMFILE=Gemfile.next bundle install
```

---

## Step 6: Verify Both Dependency Sets Work

```bash
# Run tests with current version
bundle exec rspec

# Run tests with next version
BUNDLE_GEMFILE=Gemfile.next bundle exec rspec
```

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
