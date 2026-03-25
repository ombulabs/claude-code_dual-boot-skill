# Example: Basic Dual-Boot Setup

This example walks through a Rails version upgrade. The same approach applies when upgrading Ruby versions or other core dependencies — just change what goes inside the `if next?` / `else` blocks in the Gemfile.

## Scenario

You have a Rails 7.0 application and want to upgrade to Rails 7.1. You want to set up dual-boot to test both versions during the transition.

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
  gem 'rails', '~> 7.1.0'
else
  gem 'rails', '~> 7.0.0'
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
# Current version (7.0)
bundle exec rspec
# => All green

# Next version (7.1)
BUNDLE_GEMFILE=Gemfile.next bundle exec rspec
# => Some failures — fix using NextRails.next? branching
```

### 6. Fix a Breaking Change

Rails 7.1 changed `fixture_path` to `fixture_paths`. Fix it:

```ruby
# spec/rails_helper.rb
if NextRails.next?
  config.fixture_paths = ["#{::Rails.root}/spec/fixtures"]
else
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
end
```

### 7. Verify Both Pass

```bash
bundle exec rspec          # Current: green
BUNDLE_GEMFILE=Gemfile.next bundle exec rspec  # Next: green
```

### 8. Commit

```bash
git add Gemfile Gemfile.next Gemfile.next.lock spec/rails_helper.rb
git commit -m "Set up dual-boot for Rails 7.0 → 7.1 upgrade"
```

---

## After Upgrade Is Complete

Once you're fully on Rails 7.1, clean up:

```ruby
# spec/rails_helper.rb — BEFORE cleanup
if NextRails.next?
  config.fixture_paths = ["#{::Rails.root}/spec/fixtures"]
else
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
end

# spec/rails_helper.rb — AFTER cleanup
config.fixture_paths = ["#{::Rails.root}/spec/fixtures"]
```

See `workflows/cleanup-workflow.md` for the full cleanup process.
