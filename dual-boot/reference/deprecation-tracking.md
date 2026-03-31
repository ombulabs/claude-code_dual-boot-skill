# Deprecation Tracking with Dual-Boot

**Ensuring deprecation warnings are visible and tracked during upgrades**

---

## Why This Matters

Deprecation warnings are your roadmap during a Rails upgrade — they tell you exactly what will break in the next version. If deprecations are silenced, you're flying blind. Before setting up dual-boot, you must ensure deprecation warnings are visible and tracked.

---

## Step 1: Detect If Deprecations Are Silenced

Check these files for silenced deprecations:

### Check `config/environments/test.rb`

```ruby
# BAD — deprecations are invisible
config.active_support.deprecation = :silence

# BAD — all deprecation behaviors disabled (Rails 7.0+)
config.active_support.report_deprecations = false
```

### Check `config/environments/development.rb`

Same patterns as above — `:silence` or `report_deprecations = false`.

### Check for Global Silencing

Search for these patterns anywhere in the codebase:

```ruby
# Silences all deprecation warnings globally
ActiveSupport::Deprecation.silenced = true

# Silences within a block (may be acceptable in narrow cases)
ActiveSupport::Deprecation.silence { ... }
```

### Detection Commands

```bash
# Find all deprecation configuration
grep -rn "active_support.deprecation" config/
grep -rn "report_deprecations" config/
grep -rn "Deprecation.silenced" app/ lib/ config/
grep -rn "Deprecation.silence" app/ lib/ config/
```

If any of these are silencing deprecations, fix them before proceeding with the upgrade.

---

## Step 2: Configure Deprecation Behavior

### Recommended Settings

```ruby
# config/environments/development.rb
config.active_support.deprecation = :raise

# config/environments/test.rb
config.active_support.deprecation = :raise

# config/environments/production.rb
config.active_support.deprecation = :notify
```

Setting `:raise` in test/development means you'll see deprecations immediately as failures. This is the ideal end state, but if your app has many existing deprecations, you may need a gradual approach — see "Custom Deprecation Behavior" below.

---

## Step 3: Custom Deprecation Behavior (Gradual Approach)

If setting `:raise` produces too many failures at once, use a custom behavior to selectively raise on deprecations you've already fixed, while logging the rest. This prevents regressions on fixed deprecations while giving you time to address the remaining ones.

Based on [FastRuby.io's custom deprecation behavior approach](https://www.fastruby.io/blog/custom-deprecation-behavior.html):

### Rails 6.1+ (Built-in Support)

```ruby
# config/environments/test.rb

# Raise on specific deprecations you've already fixed (prevents regressions)
config.active_support.disallowed_deprecation = :raise
config.active_support.disallowed_deprecation_warnings = [
  /update_attributes/,          # Already migrated to .update
  /Merging .* no longer/,       # Already fixed merge conditions
  "before_filter",              # Already migrated to before_action
]

# Log everything else (so you can see remaining work)
config.active_support.deprecation = :log
```

### Rails 5.2+ (Backported via Custom Lambda)

```ruby
# config/environments/test.rb
config.active_support.deprecation = :stderr

# config/initializers/custom_deprecation_behavior.rb
ActiveSupport::Deprecation.behavior = ->(message, callstack, deprecation_horizon, gem_name) {
  # Deprecations you've already fixed — raise if they reappear
  disallowed = [
    "update_attributes",
    /before_filter/,
  ]

  if disallowed.any? { |pattern|
    pattern.is_a?(Regexp) ? pattern === message : message.include?(pattern)
  }
    ActiveSupport::Deprecation::DEFAULT_BEHAVIORS[:raise].call(
      message, callstack, deprecation_horizon, gem_name
    )
  else
    ActiveSupport::Deprecation::DEFAULT_BEHAVIORS[:stderr].call(
      message, callstack, deprecation_horizon, gem_name
    )
  end
}
```

### Rails 4.0–5.1 (2-Argument Signature)

```ruby
# config/initializers/custom_deprecation_behavior.rb
ActiveSupport::Deprecation.behavior = ->(message, callstack) {
  disallowed = [
    "update_attributes",
    /before_filter/,
  ]

  if disallowed.any? { |pattern|
    pattern.is_a?(Regexp) ? pattern === message : message.include?(pattern)
  }
    ActiveSupport::Deprecation::DEFAULT_BEHAVIORS[:raise].call(message, callstack)
  else
    ActiveSupport::Deprecation::DEFAULT_BEHAVIORS[:stderr].call(message, callstack)
  end
}
```

### Rails 3.0–3.2 (No `:raise` Behavior)

```ruby
# config/initializers/custom_deprecation_behavior.rb
ActiveSupport::Deprecation.behavior = ->(message, callstack) {
  disallowed = [
    "update_attributes",
    /before_filter/,
  ]

  if disallowed.any? { |pattern|
    pattern.is_a?(Regexp) ? pattern === message : message.include?(pattern)
  }
    raise message
  end

  ActiveSupport::Deprecation::DEFAULT_BEHAVIORS[:stderr].call(message, callstack)
}
```

### Workflow for Custom Behavior

1. Start with an empty `disallowed` list and `:stderr` or `:log` as the default
2. Run tests — collect all deprecation warnings
3. Fix one category of deprecation (e.g., `update_attributes`)
4. Add it to the `disallowed` list
5. Repeat until all deprecations are fixed
6. Switch to `config.active_support.deprecation = :raise` when the list covers everything

---

## Step 4: Maintenance

After completing the upgrade:

- Review the disallowed list and remove entries for deprecations that no longer exist in the new Rails version
- This avoids unnecessary string comparisons on every deprecation check
- If you've fixed all deprecations, switch to `:raise` and remove the custom behavior entirely

---

## Quick Reference

| What to check | Command |
|---|---|
| Silenced deprecations | `grep -rn "deprecation.*silence\|silenced.*true\|report_deprecations.*false" config/` |
| Current behavior | `grep -rn "active_support.deprecation" config/environments/` |
| Custom behaviors | `grep -rn "Deprecation.behavior" config/ app/ lib/` |
| Existing warnings | `bundle exec rspec 2>&1 \| grep "DEPRECATION WARNING"` |
