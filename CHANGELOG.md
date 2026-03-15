# Changelog

All notable changes to wp-plugin-standards-claude-skills are documented here.

## [Unreleased]

## [1.1.0] - 2026-03-15

### Added
- `references/performance.md` — `WP_Query post__not_in` (and related exclusion params) guidance: when acceptable, when to refactor, and required `phpcs:ignore` pattern with inline justification
- `references/pcp-checklist.md` — 4 new PCP checks learned from WordPress.org submission feedback: `post__not_in`, `author__not_in`, `tag__not_in`, and `category__not_in` warnings

## [1.0.1] - 2026-03-12

### Added
- `commands/wp-plugin-standards-review.md` — slash command template for installing `/wp-plugin-standards-review` in Claude Code

### Changed
- Slash command renamed from `wp-plugin-standards` to `wp-plugin-standards-review` for clarity

## [1.0.0] - 2026-03-12

### Added
- `SKILL.md` — main skill file with mandatory five-step workflow and severity-graded review format
- `references/security.md` — nonce verification, sanitisation, escaping, DB queries, capability checks, filesystem access
- `references/coding-standards.md` — naming conventions, DocBlocks, formatting, Yoda conditions, i18n, error handling
- `references/performance.md` — asset enqueuing, DB query optimisation, transients, Options API, background processing
- `references/accessibility.md` — WCAG 2.1 AA, labels, ARIA attributes, keyboard navigation, colour contrast, modals
- `references/reuse.md` — Utils class pattern, version tracking, CHANGELOG format, readme.txt format, uninstall cleanup
- `references/pcp-checklist.md` — full WordPress.org Plugin Check compliance checklist covering 60+ line items
