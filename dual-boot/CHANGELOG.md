# Changelog

## v1.1 — 22 April 2026
- Replaced the `fixture_path`/`fixture_paths` and `serialize coder:` examples (both backwards-compat deprecations that don't require a conditional) with a genuine breaking change: `ignorable` gem's `ignore_columns` → native `ignored_columns=` at Rails 4.2 → 5.0, where each side's API raises `NoMethodError` on the other. Added explicit guidance that deprecations should be fixed by direct replacement, not wrapped in `NextRails.next?` (SKILL.md "Pattern", `references/code-patterns.md` "When NOT to Branch: Deprecations"). Broadened the "no feature detection" rule to call out `defined?` alongside `respond_to?`. Standardized on `NextRails.next?` polarity across the skill (removed the "`.current?` is also fine" clause from `CLAUDE.md`). Added invariant #7 codifying "only branch for genuine breaking changes". (#2)

## v1.0 — 04 March 2025
- Initial release as standalone skill (extracted from `rails-upgrade` skill)
- Setup workflow for dual-boot using `next_rails` gem
- Cleanup workflow for post-upgrade removal
- Reference materials: code patterns, CI configuration, Gemfile examples
- Critical gotcha: always use `NextRails.next?`, never `respond_to?`
- Added `/dual-boot` slash command with version argument support
