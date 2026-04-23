# Example: Basic Dual-Boot Setup

This example walks through a Rails version upgrade. The same approach applies when upgrading Ruby versions or other core dependencies — just change what goes inside the `if next?` / `else` blocks in the Gemfile.

## Scenario

You have a Rails 4.2 application and want to upgrade to Rails 5.0. You want to set up dual-boot to test both versions during the transition.

---

## Before You Start: Resolve Current-Version Deprecations

Dual-boot is the workflow for handling genuine two-sided breakage between the current and next Rails versions. Before you get there, resolve the deprecation warnings your application already emits on its **current** Rails version (4.2 in this example). Those warnings are the roadmap for the hop: each one names an API that will fail on the next side. Fixing them up front turns most of the apparent "upgrade work" into routine current-version cleanup, and leaves only the genuinely two-sided problems for dual-boot to address.

Resolve current-version deprecations unconditionally (Tier 1 in the three-tier framework). Rails ships the replacement API one version before it removes the old form, so the new call almost always works on both the current and next sides. That means no `NextRails.next?` branch is needed, and wrapping a plain deprecation fix in a conditional produces a dead branch you will only have to clean up later. Save `NextRails.next?` for Step 6, where it addresses real two-sided breakage.

To get visibility into the full inventory of current-version deprecations and track them as you fix each one, see `references/deprecation-tracking.md` for `DeprecationTracker` setup. One sequencing note: configure `DeprecationTracker` **before** running the test suite for the first time. A single run then captures the complete set of warnings, avoiding a double pass just to build the inventory.

If you are following the broader Rails upgrade methodology from the rails-upgrade skill, current-version deprecation resolution is a prerequisite to the dual-boot hop, not a step inside it. The workflow below assumes that preparatory pass is done.

See `references/code-patterns.md` "Three-Tier Approach" for the full framing of Tier 1 (unconditional migration) versus Tier 2 and Tier 3.

---

## Step-by-Step

### 1. Add `next_rails` to Gemfile

```ruby
# Gemfile
gem 'next_rails'
```

### 2. Install and Initialize

```bash
bundle install
next_rails --init
```

### 3. Configure Gemfile

```ruby
# Gemfile

source 'https://rubygems.org'

def next?
  File.basename(__FILE__) == "Gemfile.next"
end

if next?
  gem 'rails', '~> 5.0.0'
else
  gem 'rails', '~> 4.2.0'
end

gem 'next_rails'
gem 'pg'
gem 'puma'

# ... rest of gems
```

### 4. Install Dependencies for Both Versions

```bash
bundle install
BUNDLE_GEMFILE=Gemfile.next bundle install
```

### 5. Run Tests Against Both

This example uses RSpec. If your project uses Minitest, substitute `bin/rails test` (or `rake test`) for `bundle exec rspec` throughout. The `BUNDLE_GEMFILE=Gemfile.next` prefix works with any test runner. See `workflows/setup-workflow.md` Step 11 for the full detection logic (RSpec, Minitest, `bin/test`, `parallel_tests`, `turbo_tests`).

**RSpec:**

```bash
# Current version (4.2)
bundle exec rspec
# => All green

# Next version (5.0)
BUNDLE_GEMFILE=Gemfile.next bundle exec rspec
# => Some failures — fix using NextRails.next? branching
```

**Minitest equivalent:**

```bash
# Current version (4.2)
bin/rails test

# Next version (5.0)
BUNDLE_GEMFILE=Gemfile.next bin/rails test
```

### 6. Fix a Breaking Change

On Rails 4.2, `ActionController::TestRequest.new` takes an optional `env`. On 5.0, `new` requires two non-optional arguments (so the 4.2 call raises `ArgumentError`), and 5.0 introduces `TestRequest.create` — which doesn't exist on 4.2 (`NoMethodError`). Each side raises on the other. A conditional is required:

```ruby
# spec/requests/projects_spec.rb
test_request =
  if NextRails.next?
    ActionController::TestRequest.create
  else
    ActionController::TestRequest.new
  end
```

### 7. Verify Both Pass

```bash
bundle exec rspec          # Current: green
BUNDLE_GEMFILE=Gemfile.next bundle exec rspec  # Next: green
```

### 8. Commit

```bash
git add Gemfile Gemfile.next Gemfile.next.lock spec/requests/projects_spec.rb
git commit -m "Set up dual-boot for Rails 4.2 → 5.0 upgrade"
```

---

## After Upgrade Is Complete

Once you're fully on Rails 5.0, clean up:

```ruby
# spec/requests/projects_spec.rb — BEFORE cleanup
test_request =
  if NextRails.next?
    ActionController::TestRequest.create
  else
    ActionController::TestRequest.new
  end

# spec/requests/projects_spec.rb — AFTER cleanup
test_request = ActionController::TestRequest.create
```

See `workflows/cleanup-workflow.md` for the full cleanup process.
