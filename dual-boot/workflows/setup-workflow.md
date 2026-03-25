# Dual-Boot Setup Workflow

## Prerequisites

- A Ruby application with a working `Gemfile` and `Gemfile.lock`
- Know the current and target versions for the dependency you are upgrading (Rails, Ruby, or another core gem)

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

See `reference/gemfile-examples.md` for more patterns.

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

## Step 7: Commit Dual-Boot Setup

```bash
git add Gemfile Gemfile.next Gemfile.next.lock
git commit -m "Add dual-boot setup for upgrade"
```

---

## Next Steps

- Configure CI to test both versions (see `reference/ci-configuration.md`)
- Start fixing breaking changes using `NextRails.next?` branching
- See `reference/code-patterns.md` for code examples
