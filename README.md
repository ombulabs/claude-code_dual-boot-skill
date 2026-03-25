# Dual-Boot Skill

A Claude Code skill that helps you set up and manage dual-boot environments for Ruby and Rails applications using the [`next_rails`](https://github.com/fastruby/next_rails) gem.

## What Does This Skill Do?

The Dual-Boot skill helps you:

- **Set up dual-boot** with the `next_rails` gem so your app runs with two dependency sets simultaneously
- **Write version-dependent code** using `NextRails.next?` (the correct pattern — never `respond_to?`)
- **Configure CI** to test against both dependency sets (GitHub Actions, CircleCI, Jenkins)
- **Clean up** dual-boot code after the upgrade is complete

While most commonly used for **Rails version upgrades**, dual-boot works equally well for upgrading **Ruby versions** or any **core dependency** in your Gemfile (e.g., `sidekiq`, `devise`, `pg`).

## Why Dual-Boot?

Dual-booting is a core part of the [FastRuby.io](https://fastruby.io) upgrade methodology:

- Quickly switch between dependency sets for debugging
- Run test suites against both versions
- Deploy backwards-compatible changes to production before the version bump
- Catch compatibility issues early in CI

## How to Use This Skill

### Installation

```bash
# Clone the repository
git clone https://github.com/ombulabs/claude-code_dual-boot-skill.git

# Add to your Claude Code skills directory
cp -r claude-code_dual-boot-skill/dual-boot ~/.claude/skills/
```

### Basic Usage

In Claude Code, navigate to your Rails application directory and use natural language:

```
"Set up dual boot for my Rails app"
"Help me dual-boot Rails 7.0 and 7.1"
"Dual-boot Ruby 3.1 and 3.2"
"Set up dual boot for upgrading sidekiq"
"Configure dual-boot CI"
"Clean up dual-boot code after upgrade"
```

## Key Principle: Use `NextRails.next?`, Not `respond_to?`

When writing code that must work with two versions, always use `NextRails.next?`:

```ruby
# CORRECT
if NextRails.next?
  config.fixture_paths = ["#{::Rails.root}/spec/fixtures"]
else
  config.fixture_path = "#{::Rails.root}/spec/fixtures"
end
```

Never use `respond_to?` for version branching — it's hard to understand, hard to maintain, and obscures intent.

## Related Skills

- [rails-upgrade skill](https://github.com/ombulabs/claude-code_rails-upgrade-skill) — Comprehensive Rails upgrade assistant (uses this skill for dual-boot setup)
- [rails-load-defaults skill](https://github.com/fastruby/rails-load-defaults-skill) — Incremental `load_defaults` updates

## Contributing

We welcome contributions! Found incorrect information or have a suggestion? [Open an issue](https://github.com/ombulabs/claude-code_dual-boot-skill/issues).

## License

This project is licensed under the MIT License. See [LICENSE](LICENSE) for details.

## Sponsors

### [OmbuLabs.ai](https://ombulabs.ai) | Custom AI Solutions

We build custom AI solutions that integrate with your existing workflows. From Claude Code skills to full AI agent systems.

### [FastRuby.io](https://fastruby.io) | Ruby Maintenance, Done Right

The Rails upgrade experts. We've been upgrading Rails applications professionally since 2017, helping companies stay current and secure.

---

**Questions?** Open an issue or reach out to us at [hello@ombulabs.com](mailto:hello@ombulabs.com)
