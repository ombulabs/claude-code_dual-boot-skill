# Example: Basic Dual-Boot Setup

This example walks through a Rails version upgrade. The same approach applies when upgrading Ruby versions or other core dependencies — just change what goes inside the `if next?` / `else` blocks in the Gemfile.

## Scenario

You have a Rails 4.2 application and want to upgrade to Rails 5.0. You want to set up dual-boot to test both versions during the transition.

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

```bash
# Current version (4.2)
bundle exec rspec
# => All green

# Next version (5.0)
BUNDLE_GEMFILE=Gemfile.next bundle exec rspec
# => Some failures — fix using NextRails.next? branching
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
