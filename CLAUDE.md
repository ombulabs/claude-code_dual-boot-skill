# CLAUDE.md

This file guides Claude Code when working in **this repository**. This repo is the
source for the `dual-boot` Claude Code skill — a plugin that helps end users set up
and manage dual-boot Ruby/Rails environments with the [`next_rails`](https://github.com/fastruby/next_rails)
gem.

When Claude is invoked here, the task is almost always to **author, maintain, or
enhance the skill itself** — not to perform a Rails upgrade on some host app.

---

## What this repository is

A packaged Claude Code skill distributed as a plugin. The skill's authoring surface
lives in `dual-boot/`:

- `dual-boot/SKILL.md` — skill entry point and trigger patterns (authoritative)
- `dual-boot/workflows/` — step-by-step procedures (`setup-workflow.md`, `cleanup-workflow.md`)
- `dual-boot/references/` — supporting docs (code patterns, CI, Gemfile examples, deprecation tracking)
- `dual-boot/examples/` — worked examples
- `dual-boot/CHANGELOG.md` — version history

Top-level files:

- `.claude-plugin/plugin.json` — plugin manifest
- `README.md` — user-facing install + usage
- `LICENSE`

---

## Scope: stay inside the dual-boot boundary

This skill is part of a three-skill FastRuby.io toolkit. Each skill owns a distinct
concern, and **this repo must not grow guidance that belongs to the siblings.**

**In scope for this skill:**
- Installing/configuring the `next_rails` gem
- Gemfile conventions for two dependency sets (Rails, Ruby, or any core gem)
- `NextRails.next?` / `NextRails.current?` branching patterns in application code
- CI wiring for both dependency sets (GitHub Actions, CircleCI, Jenkins)
- `Gemfile.next` / `Gemfile.next.lock` lifecycle
- Cleanup after the upgrade lands
- Deprecation-warning visibility (because silenced warnings make dual-boot useless)

**Out of scope — belongs to sibling skills, do NOT add here:**
- Rails version-specific breaking changes, deprecations, or migration steps → [rails-upgrade](https://github.com/ombulabs/claude-code_rails-upgrade-skill)
- `app:update` guidance, per-version upgrade reports, detection scripts → rails-upgrade
- Walking `config.load_defaults` forward, `new_framework_defaults_*.rb` handling,
  per-config risk tiers, initializer consolidation → [rails-load-defaults](https://github.com/ombulabs/claude-code_rails-load-defaults-skill)
- Gem compatibility resolution, monkeypatch debugging, time estimates, LTS upgrades

If a task drifts toward upgrade mechanics or `load_defaults` mechanics, stop and
redirect: either point the work at the correct sibling repo, or — if a cross-link
is genuinely needed here — keep it to a one-line reference, not embedded content.

When in doubt: this skill answers "how do I run two versions side by side?", not
"which version should I go to or what breaks when I get there?"

---

## Invariants this skill teaches (and must not contradict)

These are load-bearing across the skill's content. If edits would weaken or
contradict any of them, stop and surface the conflict instead of shipping.

1. **Branch with `next_rails`, never with `respond_to?`.** The gem exposes both
   `NextRails.next?` and `NextRails.current?` — either is fine, but a given
   codebase should pick **one** and apply it consistently so readers don't have
   to mentally flip the polarity of each check. See `dual-boot/SKILL.md` for the
   rationale on why `respond_to?` is the wrong tool.
2. **Never run `next_rails --init` if `Gemfile.next` already exists** — it duplicates
   the `next?` method.
3. **Add `next_rails` at the Gemfile root**, not inside a `:development` or `:test` group.
4. **Order branches so the "next version" case reads on top.** With
   `NextRails.next?`, that means the `if` branch is next-version code and `else`
   is current-version code; with `NextRails.current?`, invert the branches so the
   next-version code still appears first.
5. **Cleanup preserves `Gemfile.next.lock` versions** — replace `Gemfile.lock` with
   `Gemfile.next.lock`, don't re-resolve.
6. **Deprecation warnings must stay visible** during dual-boot.

---

## Working conventions

- **Edit existing files over creating new ones.** The skill's shape is intentional;
  new top-level files or new `references/` entries should be justified.
- **SKILL.md is the entry point** — if you add a workflow or reference, link it from
  SKILL.md's "Available Resources" section so Claude can discover it at runtime.
- **Trigger patterns matter.** New user phrasings that should activate the skill go
  in SKILL.md's "Trigger Patterns" section.
- **Update `dual-boot/CHANGELOG.md`** for user-visible changes to the skill.
- **Keep examples runnable and minimal.** Don't invent app-specific detail users
  won't have.
- **Don't add emojis** unless the surrounding file already uses them.

---

## Validation before proposing changes

There is no automated test suite. Before opening a PR:

- Re-read the edited file end-to-end — the skill is prose, and coherence is the test.
- Cross-check any referenced path (`workflows/...`, `references/...`) actually exists.
- If you changed a workflow step, walk the full workflow mentally against a fresh
  Rails app to confirm the steps still compose.
- Confirm nothing you added belongs in rails-upgrade or rails-load-defaults (see
  "Scope" above).
