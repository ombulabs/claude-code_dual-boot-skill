# Example: Basic Dual-Boot Setup

This example walks through a Rails version upgrade. The same approach applies when upgrading Ruby versions or other core dependencies — just change what goes inside the `if next?` / `else` blocks in the Gemfile.

## Scenario

You have a Rails 6.0 application and want to upgrade to Rails 6.1. You want to set up dual-boot to test both versions during the transition.

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
  gem 'rails', '~> 6.1.0'
else
  gem 'rails', '~> 6.0.0'
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
# Current version (6.0)
bundle exec rspec
# => All green

# Next version (6.1)
BUNDLE_GEMFILE=Gemfile.next bundle exec rspec
# => Some failures — fix using NextRails.next? branching
```

### 6. Fix a Breaking Change

Rails 6.1 removed `ActionDispatch::Http::ParameterFilter` — referencing it under 6.1 raises `NameError: uninitialized constant`. The replacement `ActiveSupport::ParameterFilter` does not exist on 6.0, so a conditional is required:

```ruby
# app/services/log_sanitizer.rb
if NextRails.next?
  filter = ActiveSupport::ParameterFilter.new([:password, :token])
else
  filter = ActionDispatch::Http::ParameterFilter.new([:password, :token])
end
```

Note: a pure deprecation warning (old API still works under next) is **not** a reason to branch — replace the call site directly. Reserve `NextRails.next?` for code that would otherwise fail to load or run.

### 7. Verify Both Pass

```bash
bundle exec rspec          # Current: green
BUNDLE_GEMFILE=Gemfile.next bundle exec rspec  # Next: green
```

### 8. Commit

```bash
git add Gemfile Gemfile.next Gemfile.next.lock app/services/log_sanitizer.rb
git commit -m "Set up dual-boot for Rails 6.0 → 6.1 upgrade"
```

---

## After Upgrade Is Complete

Once you're fully on Rails 6.1, clean up:

```ruby
# app/services/log_sanitizer.rb — BEFORE cleanup
if NextRails.next?
  filter = ActiveSupport::ParameterFilter.new([:password, :token])
else
  filter = ActionDispatch::Http::ParameterFilter.new([:password, :token])
end

# app/services/log_sanitizer.rb — AFTER cleanup
filter = ActiveSupport::ParameterFilter.new([:password, :token])
```

See `workflows/cleanup-workflow.md` for the full cleanup process.
