# Changelog

All notable changes to wp-plugin-standards-claude-skills are documented here.

## [Unreleased]

## [1.2.0] - 2026-05-31

### Added
- `SKILL.md` / `references/pcp-checklist.md` — named the exact Plugin Check sniff code **`PluginCheck.CodeAnalysis.WriteFile.PluginDirectoryWrite`** on both the plugin-directory-write rule and the `WP_CONTENT_DIR` writable-storage rule. Key clarification from a live rejection: despite the "PluginDirectory" name, this sniff **also fires on `WP_CONTENT_DIR`** ("Detected usage of constant `WP_CONTENT_DIR`. Use `wp_upload_dir()` … or save to the database instead.") whenever the constant feeds a write function (`copy()`, `file_put_contents()`, `fwrite()`, `fopen()` write-mode, `move_uploaded_file()`) — so the code must not be read as "literal plugin folder only." Added to the Critical severity row.
- `references/wordpress-org-guidelines.md` — **new reference mapping all 18 Detailed Plugin Guidelines** (the human-reviewer policy criteria that Plugin Check / PHPCS do not catch). Gap analysis against the live guidelines found the skill was strong on the automated/technical half of review but under-covered the policy half — the most likely cause of repeat human-review rejections. Each guideline includes concrete audit greps. Emphasis on the nine commonly-missed gaps:
  - G1 — GPL-compatibility of **all** bundled assets (images, fonts, JS/CSS libs), not just PHP libraries
  - G4 — human-readable code: no obfuscation; minified-only JS/CSS without source is rejected
  - G5 — no trialware (features disabled until payment / trial-expiry / licence-key gating)
  - G6 — SaaS plugins must provide real local functionality, not be a validator/storefront shell
  - G7 — no tracking / phoning-home without explicit default-OFF opt-in; document in `== External services ==`
  - G8 — no third-party CDN for non-font assets, no admin `<iframe>` to external services, no installing plugins/themes from outside WordPress.org
  - G10 — "Powered by" / credit links must be opt-in and default-hidden
  - G11 — no admin-dashboard hijacking (non-dismissible/site-wide notices, dashboard ad referral tracking)
  - G12 — no readme spam (≤ 5 tags, no competitor/trademark tags, no affiliate links, no keyword stuffing)
  - G13 — use WordPress's bundled libraries (no own copy of jQuery, PHPMailer, SimplePie, Backbone, etc.)
- `SKILL.md` — Step 0 now requires reviewing against the 18 guidelines and running the policy audit greps before producing the findings report.
- `SKILL.md` — severity table extended with the guideline policy violations (trialware, opt-in tracking, bundled libraries, CDN/iframe/external-code loading, obfuscation as Critical; "powered by", admin-notice hijacking, readme spam, GPL-incompatible assets as High).
- `SKILL.md` — References table now lists `wordpress-org-guidelines.md` as an always-read reference.
- `references/pcp-checklist.md` — new "Detailed Plugin Guidelines (human review)" checklist section with the policy items and audit greps.

## [1.1.14] - 2026-05-30

### Fixed
- `SKILL.md` — **Corrected unsafe advice.** The "Runtime PHP code generation" entry previously recommended bundling static PHP files in `assets/` and deploying them via `copy()` as the *correct* pattern. WordPress.org rejected exactly this approach (cloudscale-backup, Review R…/29May26): copying an executable `.php` file into a runtime location is a hard rejection **regardless of whether the content was generated or shipped as a static asset, and regardless of destination — including the uploads directory.** Entry renamed to "Deploying executable code to disk at runtime" and rewritten to state there is no compliant runtime-install path; features that must deploy a plugin/drop-in/script provide copy-paste install text for an administrator.
- `references/pcp-checklist.md` — Same correction: "No runtime PHP code generation" item rewritten to "No executable code deployed to disk at runtime — generation OR `copy()`", removing the now-incorrect "deploy via `copy()`" guidance.

### Added
- `SKILL.md` — Audit greps for the code-deployment rule now include `copy()`, `rename()`, and `move_uploaded_file()` as write vectors (previously only `file_put_contents`/`fwrite`), since `copy()` of a `.php`/`.sh` file is the most common repeat-rejection and was missed by content-only greps.
- `SKILL.md` — `WP_CONTENT_DIR` storage rule expanded to explicitly cover writes to the `/wp-content` **root** (e.g. a companion `slug-config.json`) — including plain data files, not just code — and names the only recognised `/wp-content` exceptions (`cache/`, `backups/`, canonical core drop-ins).
- `SKILL.md` — Critical severity row expanded: executable code (`.php`/`.sh`) deployed to disk at runtime by any means, to any destination including the uploads directory.
- `references/pcp-checklist.md` — `WP_CONTENT_DIR` checklist item expanded for the `/wp-content` root case and the recognised-exceptions list.
- `SKILL.md` / `references/pcp-checklist.md` — cURL rule extended to drop-ins, mu-plugins, and bundled `assets/*.php`: the "WP HTTP API isn't loaded that early" rationale does **not** justify cURL in an early-loading drop-in (`fatal-error-handler.php`, `object-cache.php`, etc.). PCP flags `curl_init`/`curl_setopt_array`/`curl_exec`/`curl_close` there too. Correct pattern documented: guard with `function_exists( 'wp_remote_post' )` and skip the request when the API is unavailable — never fall back to cURL. Added explicit `grep` of `assets/` since drop-in `.php` files bypass the main plugin bootstrap and are easy to miss.

## [1.1.13] - 2026-05-16

### Added
- `SKILL.md` / `references/pcp-checklist.md` — `parse_url()` → `wp_parse_url()` (PCP flags `WordPress.WP.AlternativeFunctions.parse_url_parse_url`)
- `SKILL.md` / `references/pcp-checklist.md` — `$_SERVER` superglobal handling: `InputNotValidated` / `MissingUnslash` / `InputNotSanitized`; preferred fix is WordPress URL helpers (`home_url()`, `admin_url()`, `wp_parse_url()`, `add_query_arg()`) to avoid `$_SERVER` entirely
- `references/pcp-checklist.md` — direct DB query `phpcs:ignore` pattern reinforced for intentional test/crash-test files

## [1.1.12] - 2026-05-16

### Added
- `SKILL.md` — New failure mode: writing to system paths (`/usr/local/bin`, `/etc/cron.d`, `/var/log`) — applies equally to `uninstall.php` cleanup
- `SKILL.md` — New failure mode: runtime PHP code generation (`file_put_contents()` with assembled PHP strings)

### Fixed
- `SKILL.md` — `WP_CONTENT_DIR` audit rule now requires tracing helper-function return values, not just grepping for the constant at call sites
- `SKILL.md` — Cron `Throwable` wrapper rule strengthened: `try {` must be the first executable statement; an inner `try/catch` covering only part of the body is not sufficient

## [1.1.11] - 2026-05-06

### Fixed
- `SKILL.md` — cURL rule corrected: `curl_exec` / `curl_init` in plugin-authored code is now documented as a **hard WordPress.org rejection** (same as `shell_exec`), not a suppress-with-phpcs:ignore case. `phpcs:ignore` silences PHPCS/PCP but human reviewers still reject it; every own-code cURL call must be replaced with `wp_remote_get()` / `wp_remote_post()`.
- `references/pcp-checklist.md` — cURL checklist item rewritten to hard-rejection: explicitly covers all own-code API integrations (Dropbox, Google Drive, AWS S3, OneDrive, OAuth flows); clarifies that vendor/third-party libraries are exempt; narrows the phpcs:ignore exception to sub-second timeout requirements only.

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
