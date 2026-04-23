# Dual-Boot Cleanup Workflow

Use this workflow after the upgrade is complete and you're ready to remove dual-boot. This applies whether you were upgrading Rails, Ruby, or another core dependency.

---

## Prerequisites

- The upgrade to the target version is complete
- All tests pass with the new dependency set
- The old version is no longer needed

---

## Step 1: Find All `NextRails.next?` References

```bash
grep -r "NextRails.next?" . --include="*.rb" -l
```

This lists all files that contain dual-boot branching code.

---

## Step 2: Remove Dual-Boot Branches (Tier 2 cleanup)

For each file found in Step 1, keep only the `NextRails.next?` (true) branch and remove the `else` branch. This step handles per-call-site conditionals, where a single `if NextRails.next? / else` sits inside a method, a class body, or a config block.

### Before:
```ruby
test_request =
  if NextRails.next?
    ActionController::TestRequest.create
  else
    ActionController::TestRequest.new
  end
```

### After:
```ruby
test_request = ActionController::TestRequest.create
```

---

## Step 3: Delete Backport / Forwardport Shim Files (Tier 3 cleanup)

Step 2 removes branches at individual call sites. Tier-3 shims are different: an entire file exists only to make one side of the dual-boot behave like the other, typically a monkey-patch in `config/initializers/` (runtime gaps) or in `test/test_helper.rb` / `spec/spec_helper.rb` (test-only gaps). The file's outermost construct is an `if NextRails.next?` or `if !NextRails.next?` guard that wraps a `module`/`class` redefinition, and the rest of the file is the body of that guard.

### Find candidate shim files:

```bash
grep -rln -E "^if\s*!?NextRails\.next\?\s*$" config/initializers/ test/ spec/
```

The pattern matches files whose guard is on its own line at the start of a line. If your project keeps shims in other locations, broaden the search paths accordingly.

### Verify each match is a shim, not a regular call-site branch:

Open the file and look at what the guard wraps:

- **Tier-3 shim**: the guard wraps a `module X; class Y; def foo; ...; end; end; end` redefinition (or a similar `class_eval` / `prepend` block that redefines behavior process-wide). The entire file exists to install that patch.
- **Not a shim**: the guard wraps a single method call, a config setting, or a small block that happens to live at the top level of a Ruby file. That is a Tier-2 call-site branch and was already handled in Step 2.

### Delete the file:

```bash
git rm config/initializers/ac_parameters_values_backport.rb
```

Delete the whole file, not just the `if` guard. The file exists only for the shim. On the target version, the guard was already false (backport) or redundant (forwardport), so the shim never ran or no longer applies, and deleting it has no runtime effect on the surviving side.

---

## Step 4: Clean Up the Gemfile

### Remove the `next?` method definition:

```ruby
# Remove this block from Gemfile:
def next?
  File.basename(__FILE__) == "Gemfile.next"
end
```

### Remove all `if next?` / `else` conditionals:

Keep only the new version gems.

### Before:
```ruby
if next?
  gem 'rails', '~> 7.1.0'
else
  gem 'rails', '~> 7.0.0'
end
```

### After:
```ruby
gem 'rails', '~> 7.1.0'
```

---

## Step 5: Remove `next_rails` Gem

If the gem is no longer needed:

```ruby
# Remove from Gemfile:
gem 'next_rails'
```

---

## Step 6: Remove Dual-Boot Files and Preserve Lock Versions

Replace `Gemfile.lock` with `Gemfile.next.lock` to keep the exact gem versions that were tested during the upgrade, and remove `Gemfile.next` which is no longer needed — running `bundle install` from scratch could resolve to different versions.

```bash
rm Gemfile.next Gemfile.lock
mv Gemfile.next.lock Gemfile.lock
```

---

## Step 7: Run Full Test Suite

Run the project's detected test command (see SKILL.md Key Principle #4). Pick the one that matches this project:

```bash
# RSpec
bundle exec rspec

# Minitest
bin/rails test

# Or a project wrapper / parallel runner
bin/test
bundle exec parallel_rspec spec/
bundle exec turbo_tests
```

Ensure all tests pass without dual-boot.

---

## Step 8: Update CI Configuration

Remove the dual-boot CI job/matrix entry that ran tests with `BUNDLE_GEMFILE=Gemfile.next`.

---

## Step 9: Commit Cleanup

```bash
git add -A
git commit -m "Remove dual-boot setup after upgrade"
```
