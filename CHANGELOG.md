# Changelog

All notable changes to wp-plugin-standards-claude-skills are documented here.

## [Unreleased]

## [1.1.10] - 2026-05-06

### Added
- `SKILL.md` — Critical severity expanded: `Author URI` placeholder domain (`example.com`/`example.org`/`example.net`) causes automated hard-reject (`plugin_header_invalid_author_uri_domain`)
- `SKILL.md` — New failure mode: Author URI placeholder domain with grep command for pre-submission check
- `references/pcp-checklist.md` — WordPress.org submission requirements: new 🚨 check for placeholder `Author URI` domain
- `references/pcp-checklist.md` — Pre-submission final check: step 8 — grep Author URI before every upload

## [1.1.9] - 2026-05-06

### Added
- `references/cyber-security.md` — File upload section: added `require_once` of `wp-admin/includes/file.php` (and image.php/media.php for `media_handle_upload()`) when calling outside admin context; added `$upload['error']` check on `wp_handle_upload()` return; added `is_wp_error()` check for `media_handle_upload()` return
- `references/security.md` — Options and transients: added Settings API `register_setting()` pattern with `sanitize_callback`, `settings_fields()` nonce output, and array-per-group storage recommendation

## [1.1.8] - 2026-05-06

### Added
- `SKILL.md` — Critical severity expanded: shell execution (`shell_exec()`, `exec()`, `system()`, `passthru()`, `proc_open()`, `popen()`) is now documented as a hard WordPress.org rejection regardless of `escapeshellarg()` or `phpcs:ignore`
- `SKILL.md` — PCP errors missed section: added shell execution hard-block, `fopen()` on remote URLs, nonce missing `sanitize_text_field`, `WP_CONTENT_DIR` for writable storage, logging unsanitized input before sanitize call
- `SKILL.md` — Six new failure modes: shell execution hard block; `WP_CONTENT_DIR` for writable storage; `WP_PLUGIN_DIR` for own plugin paths; `fopen()` on remote URLs; nonce must use `sanitize_text_field(wp_unslash())`; logging unsanitized input before sanitization
- `references/pcp-checklist.md` — Code quality: exec/shell_exec item rewritten from phpcs:ignore advice to hard rejection with removal requirement; added `fopen()` remote URL item
- `references/pcp-checklist.md` — File and folder structure: new checks for raw `WP_CONTENT_DIR` storage and `WP_PLUGIN_DIR` own-plugin path references; `WPMU_PLUGIN_DIR` and `WP_LANG_DIR` correct-constant guidance
- `references/security.md` — Nonce section: added inline note that `wp_verify_nonce()` requires `sanitize_text_field(wp_unslash(...))`, not just `wp_unslash()`

### Fixed
- `references/coding-standards.md` — Internationalisation: corrected contradiction — removed instruction to call `load_plugin_textdomain()` (discouraged since WP 4.6; PCP flags it)

## [1.1.7] - 2026-04-27

### Added
- `references/coding-standards.md` — Internationalisation: explicit rule that `/* translators: %s: description */` comment is required on the line immediately above any i18n call containing a printf-style placeholder; PCP flags `WordPress.WP.I18n.MissingTranslatorsComment` as an error; added correct/wrong code examples
- `references/pcp-checklist.md` — File and folder structure: new check for unexpected markdown files in the plugin root (`unexpected_markdown_file` PCP warning); lists permitted files (`README.md`, `CHANGELOG.md`) and examples of disallowed ones (`UX-AUDIT.md`, `TODO.md`)
- `SKILL.md` — Critical severity expanded: i18n string with printf placeholder missing `/* translators: */` comment (`WordPress.WP.I18n.MissingTranslatorsComment`)
- `SKILL.md` — Failure modes: new entry for unexpected markdown files in the plugin root; new "PCP errors missed" bullet for `MissingTranslatorsComment`

## [1.1.6] - 2026-04-04

### Added
- `references/security.md` — Output escaping: new sub-section "Computed variables still require escaping" covering `WordPress.Security.EscapeOutput.OutputNotEscaped` on computed variables (`$total`, `$parts`) with `esc_html()` and `array_map('esc_html', ...)` fix patterns
- `references/security.md` — Input sanitisation: new sub-section "Every `$_POST`/`$_GET` value must be sanitised" covering `WordPress.Security.ValidatedSanitizedInput.InputNotSanitized`, explaining that bare `(int)` casts are not recognised by PHPCS and `absint( wp_unslash(...) )` is required

## [1.1.5] - 2026-04-02

### Added
- `SKILL.md` — Critical severity expanded: prefix shorter than 4 characters, files written to plugin directory, remote asset offloading from own server/CDN
- `SKILL.md` — four new failure modes: short prefix (< 4 chars) with list of all affected declaration types; writing files to plugin directory (use `wp_upload_dir()`); remote asset offloading from own CDN/S3; cloud metadata endpoints (link-local IPs such as AWS IMDS at 169.254.169.254) treated as external services requiring `== External services ==` readme.txt documentation
- `references/pcp-checklist.md` — Code quality: 🚨 CRITICAL prefix minimum-length check (≥ 4 chars, rejects `cs_`/`my_`/`ab_`, lists all affected element types)
- `references/pcp-checklist.md` — File and folder structure: no writing files to plugin directory; no remote asset offloading from own server/CDN

## [1.1.4] - 2026-03-22

### Added
- `references/cyber-security.md` — new comprehensive reference: admin access control (menu capabilities, render callbacks, partial templates, AJAX hook split, REST permission callbacks), OWASP Top 10 mapped to WordPress (A01–A10), and WordPress-specific vectors: open redirect (`wp_safe_redirect`), path traversal (`validate_file` + `realpath` boundary), SSRF (`wp_http_validate_url`), PHP object injection (`unserialize` ban), shell injection, file upload safety (`wp_check_filetype_and_ext`, executable extension blocklist, `wp_handle_upload`), privilege escalation, IDOR (object-level `current_user_can`), XXE, race conditions, `is_admin()` misuse, and information disclosure
- `references/security.md` — new sections: Admin access control, Object injection, Open redirect, Path traversal, SSRF, IDOR, with cross-reference to cyber-security.md
- `references/pcp-checklist.md` — new §Cyber security checklist section covering all the above vectors as actionable checkboxes
- `SKILL.md` — Critical severity expanded: admin page publicly reachable, REST `__return_true` permission callback, `unserialize()` on user input, file upload without MIME validation, shell execution with user input
- `SKILL.md` — High severity expanded: admin menu `'read'` capability, open redirect, SSRF, path traversal, `is_admin()` misuse, hardcoded credentials, IDOR
- `SKILL.md` — Medium severity expanded: `wp_ajax_nopriv_` on admin-only action, `maybe_unserialize()` on external options
- `SKILL.md` — four new failure modes documented: admin page publicly reachable, `is_admin()` misuse, `unserialize()` object injection, `__return_true` REST permission callback
- `SKILL.md` — `references/cyber-security.md` added to Step 2 and References table

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
