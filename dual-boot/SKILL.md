---
name: dual-boot
description: Sets up and manages dual-boot Ruby and Rails environments using the next_rails gem. Helps configure Gemfile for running two versions of Rails, Ruby, or any core dependency simultaneously, set up CI for both versions, use NextRails.next? for version-dependent code, and clean up after the upgrade is complete. Based on FastRuby.io methodology and "The Complete Guide to Upgrade Rails" ebook.
argument-hint: "[current-version] [target-version]"
---

When invoked via `/dual-boot`, follow the setup workflow in `workflows/setup-workflow.md` to set up dual-boot for this project. If the user provides version arguments (e.g. `/dual-boot 7.0 7.1`), use them as the current and target versions. If no arguments are provided, detect the current version from the Gemfile and ask the user for the target version.

# Dual-Boot Skill

- **Purpose:** Set up and manage dual-boot environments for Rails, Ruby, or core Gemfile dependencies using the `next_rails` gem
- **Key Gem:** [next_rails](https://github.com/fastruby/next_rails)
- **Methodology:** Based on FastRuby.io upgrade best practices and "The Complete Guide to Upgrade Rails" ebook
- **Attribution:** Content based on "The Complete Guide to Upgrade Rails" by FastRuby.io (OmbuLabs)

---

## Overview

Dual-booting allows a Ruby application to maintain two sets of dependencies simultaneously. While most commonly used for **Rails version upgrades**, the same approach works for upgrading **Ruby versions** or any **core dependency** in your Gemfile (e.g., `sidekiq`, `devise`, `pg`).

This skill helps you:

- **Set up** dual-boot with the `next_rails` gem
- **Configure** the Gemfile to support two versions of Rails, Ruby, or other key dependencies
- **Branch** application code using `NextRails.next?`
- **Configure CI** to test against both dependency sets
- **Clean up** dual-boot code after the upgrade is complete

---

## CRITICAL: Always Use `NextRails.next?` — Never Use `respond_to?`

When writing code that must work with both the current and target versions, **always use `NextRails.next?`** from the `next_rails` gem. Never use `respond_to?` or other feature-detection patterns for version branching.

**Why `respond_to?` is problematic:**
- **Hard to understand:** readers must know which version introduced a method to grasp the intent
- **Hard to maintain:** `respond_to?` checks pile up and become impossible to clean up because their purpose is lost
- **Fragile:** may give wrong results if gems monkey-patch methods in or out
- **Obscures intent:** the code says "does this method exist?" when it means "are we on the next Rails version?"

**Why `NextRails.next?` is better:**
- **Explicit and readable:** anyone reading the code immediately understands "this branch is for the next version"
- **Easy to clean up:** after the upgrade, search for `NextRails.next?` and remove all branches
- **The standard:** established dual-boot mechanism in the FastRuby.io methodology

### Pattern

❌ **WRONG — Do NOT use `respond_to?`:**
```ruby
if config.respond_to?(:fixture_paths=)
  config.fixture_paths = ["#{::Rails.root}/spec/fixtures"]
else
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
end
```

✅ **CORRECT — Use `NextRails.next?`:**
```ruby
if NextRails.next?
  config.fixture_paths = ["#{::Rails.root}/spec/fixtures"]
else
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
end
```

### When to Apply

Use `NextRails.next?` branching for:
- **Configuration changes** (e.g., `fixture_path` → `fixture_paths`)
- **API changes** (e.g., method renames, changed signatures, removed methods)
- **Gem version differences** (e.g., different gem APIs across dependency versions)
- **Initializer changes** (e.g., different middleware, different default settings)
- **Ruby version differences** (e.g., syntax changes, stdlib removals)

---

## Trigger Patterns

Claude should activate this skill when user says:

- "Set up dual boot for my Rails app"
- "Help me dual-boot Rails [x] and [y]"
- "Dual-boot Ruby [x] and [y]"
- "Set up dual boot for upgrading [dependency]"
- "Add next_rails to my project"
- "Configure dual-boot CI"
- "Clean up dual-boot code"
- "Remove next_rails after upgrade"
- "How do I use NextRails.next?"
- "Set up Gemfile for two Rails versions"
- "Set up Gemfile for two Ruby versions"

---

## Core Workflows

### Workflow 1: Set Up Dual-Boot

See `workflows/setup-workflow.md` for the complete step-by-step process.

**Summary:**
0. Verify deprecation warnings are not silenced (see `references/deprecation-tracking.md`)
1. Check if dual-boot is already set up (look for `Gemfile.next`)
2. Add `next_rails` gem to Gemfile
3. Run `bundle install`
4. Run `next_rails --init` (only if `Gemfile.next` does NOT exist)
5. Configure Gemfile with `next?` conditionals for the dependency being upgraded
6. Run `bundle install` and `next bundle install`
7. Verify both versions work

### Workflow 2: Add Version-Dependent Code

When proposing code changes that need to work with both dependency sets:

1. Verify `next_rails` is set up (check for `Gemfile.next`)
2. Use `NextRails.next?` for branching — never `respond_to?`
3. Keep the `next?` branch (new version code) on top
4. Keep the `else` branch (old version code) below

See `references/code-patterns.md` for examples.

### Workflow 3: Configure CI

See `references/ci-configuration.md` for CI setup with GitHub Actions, CircleCI, and Jenkins.

### Workflow 4: Clean Up After Upgrade

See `workflows/cleanup-workflow.md` for the complete post-upgrade cleanup process.

**Summary:**
1. Search for all `NextRails.next?` references
2. Keep only the `NextRails.next?` (true) branch code
3. Remove all `else` branches
4. Remove `next?` method from Gemfile
5. Remove `Gemfile.next` and `Gemfile.next.lock`
6. Remove the `next_rails` gem if no longer needed
7. Run `bundle install` and full test suite

---

## Available Resources

### Core Documentation
- `SKILL.md` - This file (entry point)

### Reference Materials
- `references/deprecation-tracking.md` - Detecting silenced deprecations and configuring tracking (Rails 3.0+)
- `references/code-patterns.md` - `NextRails.next?` usage examples in application code
- `references/ci-configuration.md` - CI setup for dual-boot (GitHub Actions, CircleCI, Jenkins)
- `references/gemfile-examples.md` - Gemfile configuration patterns for dual-boot

### Workflows
- `workflows/setup-workflow.md` - Step-by-step dual-boot setup
- `workflows/cleanup-workflow.md` - Post-upgrade dual-boot removal

### Examples
- `examples/basic-setup.md` - Basic dual-boot setup example

---

## Key Principles

1. **Ensure deprecation warnings are visible** — silenced deprecations mean you can't track upgrade progress
2. **Never run `next_rails --init` if `Gemfile.next` exists** — it will duplicate the `next?` method
3. **Always use `NextRails.next?` for version-dependent code** — never `respond_to?`
4. **Test both versions** — run `bundle exec rspec` and `BUNDLE_GEMFILE=Gemfile.next bundle exec rspec`
5. **Clean up after upgrade** — search for and remove all `NextRails.next?` branches
6. **Add `next_rails` at the Gemfile root level** — not inside a `:development` or `:test` group

---

## Quick Reference

| Command | Purpose |
|---------|---------|
| `next_rails --init` | Initialize dual-boot (creates `Gemfile.next` symlink) |
| `next bundle install` | Install gems for the next dependency set |
| `next bundle exec rspec` | Run tests with the next dependency set |
| `BUNDLE_GEMFILE=Gemfile.next bundle exec rails server` | Start server with next version |
| `BUNDLE_GEMFILE=Gemfile.next bundle exec rspec` | Alternative to `next` command |

---

See [CHANGELOG.md](CHANGELOG.md) for version history and current version.

