---
name: wp-plugin-standards
description: >
  WordPress.org plugin submission standards, code quality, and documentation discipline.
  Covers Plugin Check (PCP) compliance, security hardening, code reuse via shared utility
  classes, DocBlock documentation, version tracking, and CHANGELOG discipline. Use when
  writing, editing, reviewing, or preparing any WordPress plugin for WordPress.org submission.
targets: plugin
requires: wp-plugin-development
wp_version: "6.4+"
php_version: "7.4+"
---

# WordPress Plugin Standards

Companion to `wp-plugin-development`. Apply this skill when a plugin must pass
WordPress.org Plugin Check (PCP), or when improving code quality, documentation,
or submission readiness.

## When to use this skill

- Writing or modifying WordPress plugin PHP files
- Preparing a plugin for WordPress.org submission
- Reviewing plugin code for security, quality, or standards compliance
- Any task where plugin code reuse, documentation, or versioning is involved

## Mandatory workflow

**Step 0 is always required and may never be skipped.** Do not modify any file
until the user explicitly confirms they want to proceed after reviewing the report.

### Step 0 — Review report (always first)

**Before scanning files, derive the expected WordPress.org slug:**
Take the `Plugin Name:` header value, lowercase it, replace spaces with hyphens, and strip special characters. This is the expected `Text Domain:` value. If the plugin header `Text Domain:` does not match this derived slug, flag it as **Critical** immediately — this generates 70+ errors in PCP. Example: "CloudScale Free Backup and Restore" → `cloudscale-free-backup-and-restore`.

Scan all plugin files and produce a findings report grouped by severity before
doing anything else. Use this exact format:

---

**WordPress Plugin Standards — Review**

**Critical** — blocks WordPress.org submission
- `file.php:123` Description of violation

**High** — security or data integrity risk
- `file.php:78` Description of violation

**Medium** — code quality, documentation, reuse
- `file.php:12` Description of violation

**Low** — style, naming, minor standards
- `file.php:34` Description of violation

**Passed**
- Areas with no violations found

X critical · X high · X medium · X low issues found.
Confirm to proceed with all fixes, or specify which severity levels to address.

---

Severity definitions:

| Severity | Examples |
|----------|---------|
| Critical | Echoed `<script>` or `<style>` tags, missing nonce, unescaped output, raw SQL, WordPress.org ownership or contributor mismatch, hidden files (dot-files) present in the plugin directory, admin page reachable without authentication, REST endpoint with `'__return_true'` permission callback, `unserialize()` on user-supplied data, file upload without MIME validation, shell execution with user input |
| High | Missing capability check, unsanitised input, bare `die()`, hardcoded URLs, missing ABSPATH guard, admin menu using `'read'` capability, `wp_redirect()` on user-supplied URL (open redirect), user-supplied URL passed to `wp_remote_get()` (SSRF), file path built from user input without traversal check, `is_admin()` used as access-control check, hardcoded API keys or credentials, IDOR (object-level capability not checked) |
| Medium | Duplicate helper functions, missing DocBlocks, version string mismatch, missing CHANGELOG entry, global asset enqueue, `async` JS function without `try/catch` (silent failure), `catch` block with no `console.error()` or user-visible message, `getElementById()` result used without null check, `wp_ajax_nopriv_` used for an admin-only action, `maybe_unserialize()` on externally-sourced option values |
| Low | Naming convention violations, missing inline comments, non-autoloaded options, minor i18n issues, PHPCS false-positive suppressions missing for delegated-nonce handlers (`NonceVerification.Missing`) or WordPress core hook names (`NonPrefixedHooknameFound`) |

**Do not proceed to Step 1 until the user replies with confirmation.**

### Step 1 — Check Utils

Before writing any new function, read `includes/class-SLUG-utils.php` and confirm
the helper does not already exist there (see `references/reuse.md`).

### Step 2 — Apply fixes

Apply fixes for the severity levels the user confirmed. Use `references/security.md`,
`references/cyber-security.md`, `references/coding-standards.md`,
`references/performance.md`, and `references/accessibility.md` as you go.

### Step 3 — PCP checklist

Step through `references/pcp-checklist.md` and confirm zero remaining violations
in the addressed categories.

### Step 4 — Bump versions

Update plugin header `Version:`, `VERSION` constant, and `readme.txt` `Stable tag:`
in one operation — all three must always match.

### Step 5 — Update CHANGELOG.md

Add a dated entry for every change made (see `references/reuse.md` §3).

---

## Required file structure

```
plugin-slug/
├── plugin-slug.php                    Main file — plugin header + VERSION constant
├── readme.txt                         WordPress.org readme
├── CHANGELOG.md                       Keep a Changelog format
├── uninstall.php                      Removes all plugin data on uninstall
├── includes/
│   ├── class-plugin-slug.php          Core plugin class
│   └── class-plugin-slug-utils.php   Shared helpers — single source of truth
├── admin/
│   ├── class-plugin-slug-admin.php
│   └── partials/
├── public/
│   ├── class-plugin-slug-public.php
│   └── partials/
└── assets/
    ├── css/
    └── js/
```

---

## Non-negotiable rules

These apply to every file, every task. No exceptions.

**Security**
- Verify a nonce before processing any form, AJAX handler, or REST endpoint
- Sanitise all superglobal input on the way in; escape all output on the way out
- Use `$wpdb->prepare()` for every dynamic DB query — no interpolated SQL ever
- Gate every privileged action with `current_user_can()`

**Code reuse**
- Shared helpers live in `class-SLUG-utils.php` only — never duplicated across files
- Check Utils before writing any new function; if it exists there, call it

**Documentation**
- Every file, class, method, and function gets a full DocBlock (`@since`, `@param`, `@return`)
- Inline comments explain the *why*, not the *what*

**Version tracking**
- Plugin header `Version:`, `VERSION` constant, and `readme.txt` `Stable tag:` must always match
- `CHANGELOG.md` updated on every commit that changes behaviour

**PCP compliance**
- No echoed `<script>` or `<style>` tags anywhere — the top WordPress.org rejection reason
- No hidden files (filenames beginning with `.`) in the plugin directory — WordPress.org automated scan rejects them with `hidden_files` error. Common culprits: `.distignore`, `.gitignore`, `.DS_Store`. Exclude all dot-files from the distribution zip.
- No `error_log()`, `var_dump()`, `print_r()`, or bare `die()` in committed code
- No hardcoded URLs — use `plugin_dir_url()` and `plugin_dir_path()`
- **Text domain must match the WordPress.org plugin slug** — the slug is derived from the plugin *name*, not the folder name (e.g. "My Cool Plugin" → `my-cool-plugin`). PCP reports `textdomain_mismatch` and `WordPress.WP.I18n.TextDomainMismatch` on every i18n call if wrong. Always verify: slugify the plugin name and confirm it matches `Text Domain:` in the header.
- All enqueued assets versioned with the plugin version constant
- WordPress.org `Contributors:` field must match the submitting username; email domain must relate to the plugin's declared URLs
- **No `date()`** — use `gmdate()` (UTC) or `wp_date()` (localised). PCP flags `WordPress.DateTime.RestrictedFunctions.date_date` as an error on every `date()` call.
- **No `unlink()`** — use `wp_delete_file()`. PCP flags `WordPress.WP.AlternativeFunctions.unlink_unlink`.
- **No `rmdir()` or `readfile()`** — use WP Filesystem API. PCP flags these as errors.
- **`wp_die()` must receive escaped strings** — `wp_die( esc_html__( 'Forbidden', 'slug' ) )` not `wp_die( 'Forbidden' )`. PCP flags `EscapeOutput.OutputNotEscaped`.
- **Every echoed variable must be escaped** — including intermediate variables that only hold safe values (e.g. `'checked'`, hex colours, pre-computed CSS class strings). PCP's `EscapeOutput.OutputNotEscaped` fires on any unescaped variable. No exceptions. **Specific context rule:** variables echoed inside `onclick="..."` attributes must use `esc_js()`, not `esc_attr()`. A variable already sanitised earlier (e.g. `$row_ami_id = esc_attr(...)`) still triggers the error if re-echoed without wrapping — always escape at the point of output.
- **`wp_unslash()` required before every `sanitize_*()`** on superglobal input — `sanitize_text_field( wp_unslash( $_POST['field'] ?? '' ) )`. PCP flags `MissingUnslash` otherwise.
- **`InputNotSanitized`** — PCP flags `$_POST` array values even when they are validated via `array_map('intval', ...)` or `array_intersect()` against a whitelist. Add `// phpcs:ignore WordPress.Security.ValidatedSanitizedInput.InputNotSanitized -- sanitised via [method]` on those lines.
- **`set_time_limit()` with non-zero values** (e.g. `set_time_limit(120)`) also triggers `Squiz.PHP.DiscouragedFunctions` — the suppress pattern applies to all values, not just `set_time_limit(0)`.
- **Schema queries (`SHOW CREATE TABLE`, `DESCRIBE`)** require `WordPress.DB.DirectDatabaseQuery.SchemaChange` in addition to `DirectQuery` and `NoCaching` in the phpcs:ignore list.
- **Multi-line `$wpdb->prepare()` calls** — when a `phpcs:ignore` comment sits on the line above a multi-line statement, it only suppresses the first line. Use `phpcs:disable` / `phpcs:enable` blocks to cover all lines of multi-line DB calls containing interpolated table names.
- **readme.txt: max 5 tags, max 150-char short description** — PCP flags both violations.
- **Direct cURL** (`curl_init` etc.) flagged by PCP — use `wp_remote_get()` / `wp_remote_post()` where possible; suppress with `phpcs:ignore WordPress.WP.AlternativeFunctions.curl_*` and a comment explaining why `wp_remote_get()` is insufficient (e.g. sub-second timeout requirement).
- **`set_time_limit()`** flagged as discouraged — add `// phpcs:ignore Squiz.PHP.DiscouragedFunctions.Discouraged -- required to prevent PHP timeout on large backups` on each call.
- **Direct DB queries** (`$wpdb->query()` etc.) flagged as discouraged — add `// phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery, WordPress.DB.DirectDatabaseQuery.NoCaching -- [reason]` on each call.
- **`NonceVerification.Missing` false positives** — AJAX handlers using a shared nonce-check helper must add `// phpcs:disable WordPress.Security.NonceVerification.Missing -- nonce verified via [helper]()` / `// phpcs:enable` blocks around every `$_POST`/`$_GET`/`$_FILES` read.

---

## Verification

After fixes are applied, confirm:

- [ ] Review report was produced and user confirmed before any file was touched
- [ ] PCP checklist passes with zero errors in all addressed categories
- [ ] All three version strings match
- [ ] `CHANGELOG.md` has a dated entry for this change
- [ ] No helper functions duplicated — Utils is the single source of truth
- [ ] Every new function has a DocBlock with `@since`, `@param`, and `@return`

---

## References

| File | Read when |
|------|-----------|
| `references/security.md` | Any input, output, DB, AJAX, REST, or capability work |
| `references/cyber-security.md` | Admin screens, access control, OWASP checks, file upload, SSRF, open redirect, path traversal, object injection |
| `references/coding-standards.md` | Always — naming, DocBlocks, formatting, i18n, error handling |
| `references/performance.md` | Asset enqueuing, DB queries, transients, background tasks |
| `references/accessibility.md` | Any admin UI, forms, notices, or modal dialogs |
| `references/reuse.md` | Always — Utils class, version tracking, CHANGELOG, readme.txt, uninstall |
| `references/pcp-checklist.md` | Before finalising any file — full PCP compliance checklist |

---

## Failure modes

- **Hidden files in the distribution zip** — WordPress.org automated scanning rejects any plugin containing files whose names begin with `.` (e.g. `.distignore`, `.gitignore`, `.env`, `.DS_Store`). The error is `hidden_files: Hidden files are not permitted.` Fix: ensure every dot-file is listed in the rsync/zip exclusion rules used to build the distribution package. Check with `unzip -l plugin.zip | grep '/\.'` before submitting.
- **Echoed `<script>` or `<style>` tags** — the single most common WordPress.org rejection. Grep the entire codebase for `<script` and `<style` before submitting. Every hit is a violation. Use `wp_enqueue_script()`, `wp_add_inline_script()`, `wp_enqueue_style()`, and `wp_add_inline_style()` exclusively. See `references/performance.md`.
- **Ownership mismatch** — if the submitting WordPress.org username is not in `Contributors:`, or the account email domain does not relate to the plugin's declared URLs, the submission is held. Resolve via DNS TXT record, email change, or account transfer. See `references/pcp-checklist.md` WordPress.org submission section.
- **Global asset enqueue** — PCP flags CSS/JS enqueued on every page. Always gate on `$hook` in admin, conditional tags on frontend.
- **Missing nonce** — every AJAX handler and form needs one. The most common PCP security rejection.
- **Duplicate helpers** — always check Utils before writing. Copy-paste across files causes divergence and is a review failure.
- **Version mismatch** — if the three version strings differ, WordPress.org validation will fail.
- **Bare `die()`** — use `wp_die()` in HTTP contexts. Bare `die()` is flagged by PCP.
- **Missing ABSPATH guard** — every included PHP file needs `if ( ! defined( 'ABSPATH' ) ) { exit; }` as its first executable line.
- **Downgrading `Tested up to`** — WordPress.org automated scanning rejects plugins where `Tested up to` is **lower** than the current WordPress stable release (error: `outdated_tested_upto_header`). Never lower this value during a review. If the value appears to be a future version, verify against wordpress.org/news before acting — the version may simply be ahead of the reviewer's knowledge cutoff. Only ever raise `Tested up to`, never lower it.
- **Broken buttons after `onclick` refactor** — when removing inline `onclick` attributes from PHP-rendered HTML buttons to satisfy PCP, the replacement JS event-binding code must use a stable selector. A common mistake is leaving the binding code as `querySelector('[onclick="fnName()"]')` — this selector worked while the attribute was present but returns `null` once the `onclick` is removed, silently dropping the click handler. The symptom is a button that renders correctly but does nothing when clicked, with no JS error. **Audit rule:** after every PCP `onclick` removal, verify that (a) the button has an `id`, and (b) `addEventListener` is attached via that `id`. Never use `querySelector('[onclick=...]')` as a binding selector — it is inherently self-defeating. See `references/pcp-checklist.md` §onclick refactor checklist.
- **`NonceVerification.Missing` false positives via helper delegation** — PHPCS cannot trace nonce verification through a shared helper such as `ajax_check()`. Every `$_POST` access below the helper call is flagged even though the nonce was checked. Fix: wrap the affected `$_POST` reads in a `// phpcs:disable` / `// phpcs:enable` block with an explanation comment naming the helper. Example:
  ```php
  $this->ajax_check(); // verifies nonce
  // phpcs:disable WordPress.Security.NonceVerification.Missing -- nonce checked via ajax_check()
  $post_id = (int) ( $_POST['post_id'] ?? 0 );
  // phpcs:enable WordPress.Security.NonceVerification.Missing
  ```
  Review every AJAX handler that calls a shared nonce-check helper rather than `check_ajax_referer()` directly. See `references/security.md` §Nonce verification for the full pattern.
- **Unhandled async function rejections (silent JS failures)** — `async` functions called from `onclick` attributes return a Promise. If that Promise rejects (due to any runtime error — including calling `.style` on a null `getElementById` result, a network failure, or non-JSON response), the rejection is silently swallowed by the browser. The function stops mid-execution with no error message, no user feedback, and no console output unless `console.error()` is explicitly called in a `catch` block. The symptom is a button that appears to work but produces no result. **Audit rule:** every `async` function must wrap its entire body in `try { ... } catch(err) { console.error(...); /* show user message */ }`. Loops that call async functions should re-enable disabled buttons in `finally {}`. All `getElementById()` results must be null-checked before property access. See `references/coding-standards.md` §JavaScript async error handling.

- **PCP errors missed during manual review** — the review skill catches patterns by reading code, but PCP runs PHPCS rules mechanically and flags things that look fine to a human reader (e.g. a variable holding `'checked'` that is never user-controlled, but still needs `esc_attr()`; or `date()` used on a timestamp the developer controls). **The only way to guarantee zero PCP errors is to run the WordPress Plugin Check plugin locally before submission.** The review skill is a guide, not a substitute for a live PCP run. Always treat PCP output as the ground truth. Critical PCP-only catches:
  - `date()` → `gmdate()` (any `date()` call, regardless of context)
  - `unlink()` → `wp_delete_file()`
  - `rmdir()` / `readfile()` → WP Filesystem
  - `wp_die('string')` → `wp_die( esc_html__( 'string', 'slug' ) )`
  - Any unescaped intermediate variable in HTML output
  - Missing `wp_unslash()` on superglobals
  - Text domain not matching WordPress.org slug (derived from plugin *name*, not folder)
  - readme.txt: >5 tags, >150-char short description

- **Admin page publicly reachable** — registering an admin menu with `'read'` capability or omitting a `current_user_can()` check in the render callback makes the page accessible to any logged-in user (Subscribers, Customers). The `add_menu_page()` capability must be at least `'manage_options'` for administrator-only screens. Every render callback and every file in `admin/partials/` must independently re-check the required capability — WordPress only enforces the capability at the menu registration level if the page is reached via the admin menu. Direct URL access bypasses that check. See `references/cyber-security.md §Admin access control`.

- **`is_admin()` used as an access-control check** — `is_admin()` returns `true` whenever WordPress is serving any admin area request, including AJAX calls triggered from the frontend. A subscriber can send an AJAX request with `is_admin()` returning `true`. It is not an authentication or authorisation check. Always use `current_user_can()`.

- **`unserialize()` on user-supplied or option data** — PHP object injection via a crafted serialised string can trigger arbitrary object destructors and methods, potentially leading to remote code execution. Never deserialise any value that originated outside your own plugin's write path. Use `json_decode()` for all data exchange. See `references/cyber-security.md §A08`.

- **`'__return_true'` as REST permission_callback** — makes the endpoint world-readable with no authentication. Every REST endpoint that reads or modifies data must have a non-trivial `permission_callback`. See `references/cyber-security.md §REST API endpoints`.

- **`NonPrefixedHooknameFound` for WordPress core hooks** — when a plugin calls `apply_filters()` or `do_action()` using a WordPress core hook name (e.g. `the_content`, `https_local_ssl_verify`, `robots_txt`), PHPCS warns that the hook name does not start with the plugin prefix. This is a false positive — the plugin is *invoking* a core hook, not *registering* its own. Suppress inline: `// phpcs:ignore WordPress.NamingConventions.PrefixAllGlobals.NonPrefixedHooknameFound -- [hook-name] is a WordPress core filter`. See `references/pcp-checklist.md` §Code quality.
