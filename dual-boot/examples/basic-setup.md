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

If your 4.2 app already depends on a gem that needs dropping on the 5.0 side (the `ignorable` gem in this example), move it from the Gemfile root into the `else` branch so it's installed only for Rails 4.2.

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
  gem 'ignorable' # used for `ignore_columns` on 4.2; dropped on 5.0
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

Rails 5.0 introduced a native `ignored_columns=` setter, replacing the `ignorable` gem's `ignore_columns` class method. With the gem dropped on the 5.0 side, `ignore_columns` raises `NoMethodError` there; `ignored_columns=` raises `NoMethodError` on 4.2 where it doesn't yet exist. A conditional is required:

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

### 7. Verify Both Pass

```bash
bundle exec rspec          # Current: green
BUNDLE_GEMFILE=Gemfile.next bundle exec rspec  # Next: green
```

### 8. Commit

```bash
git add Gemfile Gemfile.next Gemfile.next.lock app/models/project.rb
git commit -m "Set up dual-boot for Rails 4.2 → 5.0 upgrade"
```

---

## After Upgrade Is Complete

Once you're fully on Rails 5.0, clean up:

```ruby
# app/models/project.rb — BEFORE cleanup
class Project < ActiveRecord::Base
  if NextRails.next?
    self.ignored_columns += [:category]
  else
    ignore_columns :category
  end
end

# app/models/project.rb — AFTER cleanup
class Project < ActiveRecord::Base
  self.ignored_columns += [:category]
end
```

See `workflows/cleanup-workflow.md` for the full cleanup process.
