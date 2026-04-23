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

## CRITICAL: Always Use `NextRails.next?` — Never Use Feature Detection

When code would break on one version and needs a different implementation on the other, **always use `NextRails.next?`** from the `next_rails` gem to branch — never feature detection. Do not use `respond_to?`, `defined?`, `const_defined?`, or any other feature-detection pattern for version branching.

**Why feature detection is problematic:**
- **Hard to understand:** readers must know which version introduced a method or constant to grasp the intent
- **Hard to maintain:** `respond_to?` / `defined?` checks pile up and become impossible to clean up because their purpose is lost
- **Fragile:** may give wrong results if gems monkey-patch methods or constants in or out
- **Obscures intent:** the code says "does this exist?" when it means "are we on the next Rails version?"

**Why `NextRails.next?` is better:**
- **Explicit and readable:** anyone reading the code immediately understands "this branch is for the next version"
- **Easy to clean up:** after the upgrade, search for `NextRails.next?` and remove all branches
- **The standard:** established dual-boot mechanism in the FastRuby.io methodology

### Pattern

Every dual-boot has exactly one conditional that is always required: the Gemfile version pin. That is the canonical use of `next?`. App-code `NextRails.next?` is the exception, reserved for genuine two-sided breakage that cannot be resolved by migrating to a common API.

**The canonical conditional, the Gemfile version pin:**
```ruby
# Gemfile
if next?
  gem 'rails', '~> 7.1.0'
else
  gem 'rails', '~> 7.0.0'
end
```

The two pins genuinely fail resolution on opposite sides, so this conditional is required for bundler to resolve at all. Most dual-boots need nothing more than this plus a handful of gem-version pins for dependencies tied to the Rails version.

**Three-tier approach for application code.** When a version gap affects application code, pick the lowest tier that works:

1. **Tier 1: Unconditional migration (preferred).** Both versions accept the new form, so just use it everywhere. This covers the typical Rails deprecation case because Rails ships the replacement one version before it removes the old form.
2. **Tier 2: `NextRails.next?` conditional at the call site.** The new API doesn't exist on current, or the old one raises on next, and the affected call sites are few (up to ~20).
3. **Tier 3: Backport or forwardport shim.** The gap spans an application layer (all controllers, all mailers, all models with the same concern) or affects ~20+ call sites. A single monkey-patch in one initializer (for runtime gaps) or `test_helper.rb` / `spec_helper.rb` (for test-only gaps) keeps call sites identical on both sides and collapses to a single-file delete at cleanup.

See `references/code-patterns.md` "Three-Tier Approach" for the full decision framework, cleanup semantics per tier, and the Tier 3 `ActionController::Parameters#values` backport example.

**Tier 2 illustration, `ignored_columns` (Rails 4.2 → 5.0):**

❌ **WRONG — Do NOT use feature detection (`respond_to?`, `defined?`, etc.):**
```ruby
test_request =
  if ActionController::TestRequest.respond_to?(:create)
    ActionController::TestRequest.create
  else
    ActionController::TestRequest.new
  end
```

✅ **CORRECT — Use `NextRails.next?`:**
```ruby
test_request =
  if NextRails.next?
    ActionController::TestRequest.create
  else
    ActionController::TestRequest.new
  end
```

This is a genuine two-sided case (Rails 4.2 → 5.0). On 4.2, [`ActionController::TestRequest.new`](https://github.com/rails/rails/blob/11f2bdf75a888682b34df0f9be03b94f54fc6796/actionpack/lib/action_controller/test_case.rb#L201) takes an optional `env`; on 5.0 `new` [requires two non-optional arguments](https://github.com/rails/rails/blob/c4d3e202e10ae627b3b9c34498afb45450652421/actionpack/lib/action_controller/test_case.rb#L48) so the 4.2 call fails. Rails 5.0 introduces `TestRequest.create`, which does not exist on 4.2 — so each side raises on the other.

Put the next-version branch on top so cleanup is mechanical: after the upgrade, keep the `if` body and drop the `else`. See `references/code-patterns.md` for more examples.

### When to Apply

Most dual-boots only use `next?` in the Gemfile. App-code branching (Tier 2 or Tier 3) is reserved for genuine two-sided breakage where migrating to a common API (Tier 1) isn't possible. Reach for a call-site conditional or a shim when:

- **Removed methods or constants** where the replacement only exists on the new side (e.g., a gem-provided method replaced by a native Rails method with different syntax)
- **Incompatible signature or return-type changes** where the old and new forms genuinely fail on opposite sides
- **Gem version differences** where the gem's API is different across versions pinned to each Rails version
- **Initializer changes** where a config or middleware raises on one side
- **Ruby version differences** (e.g., syntax changes, stdlib removals)

Whether to resolve the gap with a call-site conditional (Tier 2) or a single shim (Tier 3) depends on how many call sites are affected and whether the gap spans a whole application layer. See `references/code-patterns.md` "Three-Tier Approach".

**Not a reason to branch:** plain deprecation warnings. If the new API works on both versions (e.g. `config.fixture_path=` → `config.fixture_paths=`, `update_attributes` → `update` before its removal), migrate the call site directly. Do not wrap it in `NextRails.next?`. See `references/code-patterns.md` "Tier 1: Unconditional Migration".

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

*Phase 1, preparatory (before any Rails version hop):*
1. Verify deprecation warnings are not silenced (see `references/deprecation-tracking.md`)
2. Check if dual-boot is already set up (look for `Gemfile.next`)
3. Add `next_rails` gem at the Gemfile root and run `bundle install`
4. Run `next_rails --init` (only if `Gemfile.next` does NOT exist)
5. Configure `DeprecationTracker` (ask upfront whether CI runs tests in parallel)
6. Capture the deprecation inventory: `DEPRECATION_TRACKER=save bundle exec rspec` (substitute the project's test command: `bin/rails test`, `bin/test`, `bundle exec parallel_rspec spec/`, etc.)
7. Fix current-side deprecations unconditionally (Tier 1, no `NextRails.next?`)

*Phase 2, dual-boot (the version hop):*
8. Configure the Gemfile with the `if next?` version conditional
9. Identify and pin incompatible gems via `bundle_report compatibility --rails-version=<target>`
10. Install dependencies for both versions:
    - Current: `bundle install`
    - Next: `cp Gemfile.lock Gemfile.next.lock && BUNDLE_GEMFILE=Gemfile.next bundle install`
11. Verify both sides boot and run tests
12. Fix two-sided breakage using Tier 2 or Tier 3 (`references/code-patterns.md`)

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
2. At each call site, keep only the `NextRails.next?` (true) branch and remove the `else` branch (Tier 2 cleanup)
3. Delete backport/forwardport shim files whose outermost guard is `if NextRails.next?` or `if !NextRails.next?` (Tier 3 cleanup)
4. Remove `next?` method and `if next?` blocks from Gemfile
5. Remove `next_rails` gem if no longer needed
6. Remove `Gemfile.next` and replace `Gemfile.lock` with `Gemfile.next.lock`
7. Run full test suite
8. Update CI to remove dual-boot configuration
9. Commit cleanup changes

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
4. **Test both versions using the project's own test runner.** Detect it before running anything: check the `Gemfile` for `rspec-rails`, `parallel_tests`, `turbo_tests`, or `minitest-parallel_fork`, check whether the app has `spec/` or `test/`, and prefer wrappers like `bin/test` when they exist. Every `bundle exec rspec` in the workflows, examples, and CI reference is illustrative; substitute the detected command (e.g. `bin/rails test`, `bundle exec parallel_rspec spec/`, `bin/test`) and prefix with `BUNDLE_GEMFILE=Gemfile.next` (or the `next` CLI) to run it against the next dependency set.
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

