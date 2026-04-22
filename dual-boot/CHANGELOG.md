# Changelog

## v1.1 — 22 April 2026
- Replaced `fixture_path`/`fixture_paths` and `serialize coder:` examples (both are backwards-compat deprecations) with genuine breaking changes (`ActionDispatch::Http::ParameterFilter` removal in Rails 6.1). Added explicit guidance: deprecations should be fixed by direct replacement, not wrapped in `NextRails.next?`. (#2)

## v1.0 — 04 March 2025
- Initial release as standalone skill (extracted from `rails-upgrade` skill)
- Setup workflow for dual-boot using `next_rails` gem
- Cleanup workflow for post-upgrade removal
- Reference materials: code patterns, CI configuration, Gemfile examples
- Critical gotcha: always use `NextRails.next?`, never `respond_to?`
- Added `/dual-boot` slash command with version argument support
