# Gemfile Configuration Examples for Dual-Boot

Dual-boot uses the `next?` helper to maintain two dependency sets in a single Gemfile. While most commonly used for Rails upgrades, the same pattern works for Ruby version upgrades or any core dependency.

---

## Rails Version Upgrade

The most common use case — running two Rails versions side by side:

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

# Common gems that work with both versions
gem 'devise', '>= 4.8'
gem 'sidekiq', '>= 6.0'
```

---

## Ruby Version Upgrade

Use `next?` to specify different Ruby versions:

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

gem 'rails', '~> 7.1.0'
```

You can combine Ruby and gem version differences if needed:

```ruby
if next?
  ruby '3.3.0'
  gem 'some_gem', '~> 2.0'  # requires Ruby 3.3+
else
  ruby '3.2.0'
  gem 'some_gem', '~> 1.5'
end
```

---

## Core Dependency Upgrade

Use the same pattern for any gem that requires careful migration:

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

---

## Handling Related Gem Version Differences

When upgrading one dependency forces changes in others:

```ruby
# Gemfile

def next?
  File.basename(__FILE__) == "Gemfile.next"
end

if next?
  gem 'rails', '~> 7.0.0'
  gem 'activeadmin', '~> 3.0'
  gem 'ransack', '~> 4.0'
else
  gem 'rails', '~> 6.1.0'
  gem 'activeadmin', '~> 2.9'
  gem 'ransack', '~> 2.6'
end
```

---

## Complete Example (Rails Upgrade)

```ruby
# Gemfile

source 'https://rubygems.org'

def next?
  File.basename(__FILE__) == "Gemfile.next"
end

# Rails version
if next?
  gem 'rails', '~> 7.0.0'
else
  gem 'rails', '~> 6.1.0'
end

# Database
gem 'pg', '~> 1.4'

# Authentication
if next?
  gem 'devise', '~> 4.9'
else
  gem 'devise', '~> 4.8'
end

# Background jobs
gem 'sidekiq', '~> 7.0'

# Dual-boot support
gem 'next_rails'

# Testing (development/test only)
group :development, :test do
  gem 'rspec-rails', next? ? '~> 6.0' : '~> 5.1'
  gem 'factory_bot_rails'
end
```

---

## Tips

- Place the `next?` method definition at the **top** of the Gemfile, before any conditional usage
- Use `next?` (lowercase, in Gemfile) for gem conditionals and `NextRails.next?` (in Ruby code) for application code
- Keep gems that work with both versions outside of conditionals
- Use inline ternary (`next? ? '~> 6.0' : '~> 5.1'`) for simple version differences
- Use `if/else` blocks for gems that only exist in one version
- You can combine multiple upgrades (e.g., Rails + Ruby) in the same dual-boot setup, but upgrading one thing at a time is safer
