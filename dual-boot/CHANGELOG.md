# Changelog

## v1.1 — 22 April 2026
- Replaced the `fixture_path`/`fixture_paths` and `serialize coder:` examples (both backwards-compat deprecations that don't require a conditional) with a genuine breaking change: `ignorable` gem's `ignore_columns` → native `ignored_columns=` at Rails 4.2 → 5.0, where each side's API raises `NoMethodError` on the other. (#2)
- Added a new "When NOT to Branch: Deprecations" section in `references/code-patterns.md` using `fixture_path` → `fixture_paths` as the counter-example, so readers learn that deprecations are unconditional migrations — not `NextRails.next?` conditionals.
- Broadened the "no feature detection" rule to call out `defined?` and `const_defined?` alongside `respond_to?`.
- Standardized on `NextRails.next?` polarity across the skill (removed the "`.current?` is also fine" clause from `CLAUDE.md`).
- Added invariant #7 in `CLAUDE.md`: only branch for genuine breaking changes.
- Removed the `session_store` and `Sidekiq` examples that didn't survive invariant #7 — both were either cosmetic config differences (session_store) or ambiguous across intra-major gem versions (`default_worker_options` is a deprecated alias on Sidekiq 6.5+, so the example only held for Sidekiq ≤ 6.4 → 7.0).

## v1.0 — 04 March 2025
- Initial release as standalone skill (extracted from `rails-upgrade` skill)
- Setup workflow for dual-boot using `next_rails` gem
- Cleanup workflow for post-upgrade removal
- Reference materials: code patterns, CI configuration, Gemfile examples
- Critical gotcha: always use `NextRails.next?`, never `respond_to?`
- Added `/dual-boot` slash command with version argument support
