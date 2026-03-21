# Changelog

All notable changes to wp-plugin-standards-claude-skills are documented here.

## [Unreleased]

## [1.1.3] - 2026-03-21

### Added
- `SKILL.md` — text domain slug derivation rule: derive expected slug from `Plugin Name:` header before scanning; flag mismatch as Critical immediately
- `SKILL.md` + `references/pcp-checklist.md` — `date()` → `gmdate()`/`wp_date()`, `unlink()` → `wp_delete_file()`, `rmdir()`/`readfile()` → WP Filesystem API
- `SKILL.md` + `references/pcp-checklist.md` — `wp_die()` must receive an escaped string; unescaped literal triggers `EscapeOutput.OutputNotEscaped`
- `SKILL.md` + `references/pcp-checklist.md` — `wp_unslash()` required before every `sanitize_*()` on superglobal input
- `SKILL.md` + `references/pcp-checklist.md` — `InputNotSanitized` false positives: `array_map('intval')` / `array_intersect()` validation requires inline `phpcs:ignore`
- `SKILL.md` + `references/pcp-checklist.md` — direct cURL (`curl_init` etc.) suppression pattern with justification comment
- `SKILL.md` + `references/pcp-checklist.md` — `set_time_limit()` discouraged function suppression pattern
- `SKILL.md` + `references/pcp-checklist.md` — schema queries (`SHOW CREATE TABLE`, `DESCRIBE`) require `SchemaChange` in phpcs:ignore list
- `SKILL.md` + `references/pcp-checklist.md` — multi-line `$wpdb->prepare()` calls need `phpcs:disable`/`phpcs:enable` blocks, not single-line `phpcs:ignore`
- `references/pcp-checklist.md` — `readme.txt` limits: max 5 tags, max 150-char short description
- `SKILL.md` — "PCP errors missed during manual review" callout: PCP is ground truth; lists critical PCP-only catches

## [1.1.2] - 2026-03-16

### Added
- `references/coding-standards.md` — new §JavaScript async error handling: required `try/catch` pattern for `async` functions, mandatory `console.error()` + user-visible message in `catch`, `finally` for re-enabling buttons, and null-checking `getElementById` results
- `SKILL.md` — unhandled async function rejections documented as a known failure mode: `async` functions called from `onclick` return a Promise whose rejection is silently swallowed by the browser, causing "button does nothing" bugs
- `SKILL.md` — `NonPrefixedHooknameFound` false-positive pattern: invoking a WordPress core hook (e.g. `the_content`) triggers PHPCS; documents correct inline suppression
- `SKILL.md` — `NonceVerification.Missing` delegation pattern: AJAX handlers that delegate nonce verification to a shared helper should use `phpcs:disable` / `phpcs:enable` block suppression
- `SKILL.md` — Medium severity table expanded with JS audit items: `async` function without `try/catch`, `catch` with no `console.error()`, `getElementById` result used without null check

## [1.1.1] - 2026-03-16

### Fixed
- `SKILL.md` — never downgrade the `Tested up to` header; WordPress.org rejects submissions with an `outdated_tested_upto_header` error if the value is lower than the current WP release

## [1.1.0] - 2026-03-15

### Added
- `references/performance.md` — `WP_Query post__not_in` (and related exclusion params) guidance: when acceptable, when to refactor, and required `phpcs:ignore` pattern with inline justification
- `references/pcp-checklist.md` — 4 new PCP checks learned from WordPress.org submission feedback: `post__not_in`, `author__not_in`, `tag__not_in`, and `category__not_in` warnings
- `SKILL.md` + `references/pcp-checklist.md` — hidden files (dot-files) added as a Critical PCP check; WordPress.org automated scanning rejects any plugin zip containing dot-files with `hidden_files` error; documents fix and `unzip -l` verification command

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
