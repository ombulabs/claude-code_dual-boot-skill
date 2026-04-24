# Changelog

## v1.1 — 24 April 2026
- Swapped the canonical NextRails.next? example across the skill to `ActionController::TestRequest.new` → `TestRequest.create` at Rails 4.2 → 5.0: `new` takes optional `env` on 4.2 but requires 2 non-optional args on 5.0, and `create` doesn't exist on 4.2 — each side raises on the other. Updated SKILL.md, README.md, `references/code-patterns.md`, `workflows/cleanup-workflow.md`, and `examples/basic-setup.md`.
- Previously replaced the `fixture_path`/`fixture_paths` and `serialize coder:` examples (both backwards-compat deprecations that don't require a conditional) with a genuine breaking change. (#2)
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
