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

## Step 2: Remove Dual-Boot Branches

For each file found in Step 1, keep only the `NextRails.next?` (true) branch and remove the `else` branch.

### Before:
```ruby
if NextRails.next?
  config.fixture_paths = ["#{::Rails.root}/spec/fixtures"]
else
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
end
```

### After:
```ruby
config.fixture_paths = ["#{::Rails.root}/spec/fixtures"]
```

---

## Step 3: Clean Up the Gemfile

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

## Step 4: Remove `next_rails` Gem

If the gem is no longer needed:

```ruby
# Remove from Gemfile:
gem 'next_rails'
```

---

## Step 5: Remove Dual-Boot Files and Preserve Lock Versions

Replace `Gemfile.lock` with `Gemfile.next.lock` to keep the exact gem versions that were tested during the upgrade, and remove `Gemfile.next` which is no longer needed — running `bundle install` from scratch could resolve to different versions.

```bash
rm Gemfile.next Gemfile.lock
mv Gemfile.next.lock Gemfile.lock
```

---

## Step 6: Run Full Test Suite

Use the project's test runner:

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

## Step 7: Update CI Configuration

Remove the dual-boot CI job/matrix entry that ran tests with `BUNDLE_GEMFILE=Gemfile.next`.

---

## Step 8: Commit Cleanup

```bash
git add -A
git commit -m "Remove dual-boot setup after upgrade"
```
