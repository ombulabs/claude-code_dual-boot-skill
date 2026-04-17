# Changelog

## v1.0 — March 2025
- Initial release as standalone skill (extracted from `rails-upgrade` skill)
- Setup workflow for dual-boot using `next_rails` gem
- Cleanup workflow for post-upgrade removal
- Reference materials: code patterns, CI configuration, Gemfile examples
- Critical gotcha: always use `NextRails.next?`, never `respond_to?`
- Added `/dual-boot` slash command with version argument support
