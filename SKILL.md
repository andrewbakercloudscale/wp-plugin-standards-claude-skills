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
| Critical | Echoed `<script>` or `<style>` tags, missing nonce, unescaped output, raw SQL, WordPress.org ownership or contributor mismatch, `Author URI` containing a placeholder domain (`example.com` / `example.org` / `example.net`) — automated hard-reject (`plugin_header_invalid_author_uri_domain`), hidden files (dot-files) present in the plugin directory, admin page reachable without authentication, REST endpoint with `'__return_true'` permission callback, `unserialize()` on user-supplied data, file upload without MIME validation, any `shell_exec()`, `exec()`, `system()`, `passthru()`, `proc_open()`, or `popen()` call (WordPress.org reviewers require **complete removal** — `escapeshellarg()` and `phpcs:ignore` do **not** satisfy the review; the plugin will be rejected regardless of how arguments are sanitised), prefix shorter than 4 characters (e.g. `cs_`), files written to the plugin directory, remote asset offloading from own server/CDN, WP-Cron callback registered via bare `add_action()` without a `Throwable`-catching wrapper, i18n string with a printf placeholder missing the `/* translators: */` comment (`WordPress.WP.I18n.MissingTranslatorsComment`) |
| High | Missing capability check, unsanitised input, bare `die()`, hardcoded URLs, missing ABSPATH guard, admin menu using `'read'` capability, `wp_redirect()` on user-supplied URL (open redirect), user-supplied URL passed to `wp_remote_get()` (SSRF), file path built from user input without traversal check, `is_admin()` used as access-control check, hardcoded API keys or credentials, IDOR (object-level capability not checked), AJAX handler using a nonce helper wrapper instead of `check_ajax_referer()` directly (`NonceVerification.Missing`) |
| Medium | Duplicate helper functions, missing DocBlocks, version string mismatch, missing CHANGELOG entry, global asset enqueue, `async` JS function without `try/catch` (silent failure), `catch` block with no `console.error()` or user-visible message, `getElementById()` result used without null check, `wp_ajax_nopriv_` used for an admin-only action, `maybe_unserialize()` on externally-sourced option values |
| Low | Naming convention violations, missing inline comments, non-autoloaded options, minor i18n issues, PHPCS false-positive suppressions missing for WordPress core hook names (`NonPrefixedHooknameFound`) |

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
- **Direct cURL** (`curl_init`, `curl_exec`, etc.) in plugin-authored code is a **hard WordPress.org rejection** — identical treatment to `shell_exec()`. `phpcs:ignore` suppression silences PHPCS/PCP but **human reviewers will still reject it**. Every `curl_exec` call in your own code must be replaced with `wp_remote_get()` / `wp_remote_post()`. **Third-party vendor libraries** containing cURL are permitted — reviewers explicitly distinguish vendor code from plugin-specific code. The sole technically defensible exception in own code is a sub-second connect timeout requirement (e.g. AWS IMDS polling) where `wp_remote_get()` genuinely cannot substitute; suppress with `// phpcs:ignore WordPress.WP.AlternativeFunctions.curl_curl_init, WordPress.WP.AlternativeFunctions.curl_curl_setopt_array, WordPress.WP.AlternativeFunctions.curl_curl_exec, WordPress.WP.AlternativeFunctions.curl_curl_getinfo, WordPress.WP.AlternativeFunctions.curl_curl_close -- wp_remote_get() does not support sub-second connect timeouts` and a comment, but be aware human reviewers may still flag it.
- **`set_time_limit()`** flagged as discouraged — add `// phpcs:ignore Squiz.PHP.DiscouragedFunctions.Discouraged -- required to prevent PHP timeout on large backups` on each call.
- **Direct DB queries** (`$wpdb->query()` etc.) flagged as discouraged — add `// phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery, WordPress.DB.DirectDatabaseQuery.NoCaching -- [reason]` on each call.
- **`NonceVerification.Missing`** — PHPCS and the WordPress.org Plugin Check only recognise nonce verification when `check_ajax_referer()`, `wp_verify_nonce()`, or `check_admin_referer()` is called **directly in the same function scope** as the `$_POST`/`$_GET`/`$_FILES` access. A helper wrapper (e.g. `ajax_check()`, `cs_verify_nonce()`) that calls `check_ajax_referer()` internally is **not** traced — every `$_POST` read below it is still flagged. Fix: replace every helper call site with a direct `check_ajax_referer()` call. Adding `phpcs:disable NonceVerification.Missing` is not a substitute — Plugin Check still flags the violation.

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
- **`NonceVerification.Missing` via helper delegation** — PHPCS and the WordPress.org Plugin Check only recognise nonce verification when `check_ajax_referer()`, `wp_verify_nonce()`, or `check_admin_referer()` is called **directly in the same handler scope**. A shared helper (e.g. `ajax_check()`, `cs_verify_nonce()`) that wraps `check_ajax_referer()` internally is **invisible** to the sniff — every `$_POST`/`$_GET`/`$_FILES` access below the helper call is flagged, and `phpcs:disable NonceVerification.Missing` does not satisfy Plugin Check. Fix: replace every helper call site with a direct `check_ajax_referer( 'action', 'field' )` call. The helper function can remain for other uses; just remove the delegation at each handler. Audit by grepping for helper calls in `wp_ajax_` actions and confirming `check_ajax_referer()` appears directly in the same closure/function body. See `references/security.md` §Delegated nonce verification for the full pattern.
- **Unhandled async function rejections (silent JS failures)** — `async` functions called from `onclick` attributes return a Promise. If that Promise rejects (due to any runtime error — including calling `.style` on a null `getElementById` result, a network failure, or non-JSON response), the rejection is silently swallowed by the browser. The function stops mid-execution with no error message, no user feedback, and no console output unless `console.error()` is explicitly called in a `catch` block. The symptom is a button that appears to work but produces no result. **Audit rule:** every `async` function must wrap its entire body in `try { ... } catch(err) { console.error(...); /* show user message */ }`. Loops that call async functions should re-enable disabled buttons in `finally {}`. All `getElementById()` results must be null-checked before property access. See `references/coding-standards.md` §JavaScript async error handling.

- **PCP errors missed during manual review** — the review skill catches patterns by reading code, but PCP runs PHPCS rules mechanically and flags things that look fine to a human reader (e.g. a variable holding `'checked'` that is never user-controlled, but still needs `esc_attr()`; or `date()` used on a timestamp the developer controls). **The only way to guarantee zero PCP errors is to run the WordPress Plugin Check plugin locally before submission.** The review skill is a guide, not a substitute for a live PCP run. Always treat PCP output as the ground truth. Critical PCP-only catches:
  - `date()` → `gmdate()` (any `date()` call, regardless of context)
  - `mt_rand()` → `wp_rand()` — PCP flags `WordPress.WP.AlternativeFunctions.rand_mt_rand` as an error
  - `unlink()` → `wp_delete_file()`
  - `rmdir()` / `readfile()` → WP Filesystem
  - `wp_die('string')` → `wp_die( esc_html__( 'string', 'slug' ) )`
  - **`exec()` / `shell_exec()` / `system()` / `passthru()` → must be removed entirely** — adding `phpcs:ignore` is not a fix; WordPress.org reviewers reject the plugin outright regardless of whether `escapeshellarg()` is used
  - `file_put_contents()` / `file_get_contents()` without phpcs:ignore — `WordPress.WP.AlternativeFunctions.file_system_operations_*`
  - `fopen()` on remote URLs — use `wp_remote_get()` / `wp_remote_post()`; many hosts block PHP stream wrappers for remote access
  - `wp_verify_nonce()` called with only `wp_unslash()` on the nonce value — must use `sanitize_text_field( wp_unslash( ... ) )` because `wp_verify_nonce()` is pluggable
  - `WP_CONTENT_DIR . '/subfolder/'` for writable storage — use `wp_upload_dir()['basedir'] . '/plugin-slug/'` for all persistent file storage outside the database
  - Logging a raw superglobal value before the `sanitize_*()` call on the same or a later line — WordPress.org flags this even with a `phpcs:ignore InputNotSanitized` comment; sanitize first, log after
  - Any unescaped intermediate variable in HTML output
  - Missing `wp_unslash()` on superglobals — even for integer casts, use `(int) wp_unslash( $_POST['field'] ?? 0 )`
  - Text domain not matching WordPress.org slug (derived from plugin *name*, not folder)
  - readme.txt: >5 tags, >150-char short description
  - `printf()`/`sprintf()` with a placeholder in an i18n string — must have `/* translators: %s: description */` on the line immediately above; PCP flags `WordPress.WP.I18n.MissingTranslatorsComment` as an error on every missing comment

- **Admin page publicly reachable** — registering an admin menu with `'read'` capability or omitting a `current_user_can()` check in the render callback makes the page accessible to any logged-in user (Subscribers, Customers). The `add_menu_page()` capability must be at least `'manage_options'` for administrator-only screens. Every render callback and every file in `admin/partials/` must independently re-check the required capability — WordPress only enforces the capability at the menu registration level if the page is reached via the admin menu. Direct URL access bypasses that check. See `references/cyber-security.md §Admin access control`.

- **`is_admin()` used as an access-control check** — `is_admin()` returns `true` whenever WordPress is serving any admin area request, including AJAX calls triggered from the frontend. A subscriber can send an AJAX request with `is_admin()` returning `true`. It is not an authentication or authorisation check. Always use `current_user_can()`.

- **`unserialize()` on user-supplied or option data** — PHP object injection via a crafted serialised string can trigger arbitrary object destructors and methods, potentially leading to remote code execution. Never deserialise any value that originated outside your own plugin's write path. Use `json_decode()` for all data exchange. See `references/cyber-security.md §A08`.

- **`'__return_true'` as REST permission_callback** — makes the endpoint world-readable with no authentication. Every REST endpoint that reads or modifies data must have a non-trivial `permission_callback`. See `references/cyber-security.md §REST API endpoints`.

- **`NonPrefixedHooknameFound` for WordPress core hooks** — when a plugin calls `apply_filters()` or `do_action()` using a WordPress core hook name (e.g. `the_content`, `https_local_ssl_verify`, `robots_txt`), PHPCS warns that the hook name does not start with the plugin prefix. This is a false positive — the plugin is *invoking* a core hook, not *registering* its own. Suppress inline: `// phpcs:ignore WordPress.NamingConventions.PrefixAllGlobals.NonPrefixedHooknameFound -- [hook-name] is a WordPress core filter`. See `references/pcp-checklist.md` §Code quality.

- **`Author URI` placeholder domain — automated hard-reject** — WordPress.org's automated scanner rejects any plugin where `Author URI:` in the plugin header contains `example.com`, `example.org`, `example.net`, or any RFC 2606 reserved placeholder domain. Error: `plugin_header_invalid_author_uri_domain`. This fires before the submission even reaches human review. **Grep before every submission:** `grep -i "Author URI" plugin-slug.php` and confirm it points to a real URL you own. The same applies to `Plugin URI:` — a 404 or placeholder there is also flagged.

- **Plugin name contains "Free"** — WordPress.org discourages the word "Free" in a plugin display name; it is redundant in the directory. A name like "My Plugin Free" will be rejected and the reviewer will ask you to rename it. Remove "Free" from the display name and slug. If the old slug is already in use (e.g. during submission), request a new slug in your reply email.

- **Missing `== External services ==` section in readme.txt** — if the plugin connects to any third-party or external service (S3, Google Drive, payment gateways, IMDS endpoints, external APIs), the readme.txt **must** contain an `== External services ==` section. For each service, document: what the service is and what it is used for, what data is sent and when, and links to the service's terms of service and privacy policy. This is both a guideline requirement (Guideline 6) and a legal protection for the plugin author. A privacy policy section alone is not sufficient.

- **Privacy policy falsely claims no external services** — if the readme.txt Privacy Policy section says "no external requests of any kind" but the plugin does make external requests (e.g. optional S3 sync, rclone, AWS IMDS), this is flagged in review. Ensure the privacy policy accurately reflects what the plugin does, and cross-reference the `== External services ==` section for the conditional cases.

- **Plugin URI returns 404** — the `Plugin URI:` header in the plugin file must point to a real, reachable URL. A 404 is flagged by reviewers. Use a URL you control (`https://yoursite.com`) or the WordPress.org plugin page (`https://wordpress.org/plugins/your-slug/`) once approved. When checking links, follow the three-pass rule below.

- **Broken link false positives from bot-protection walls** — many high-traffic sites (Reuters, WatchMojo, etc.) return `401 Unauthorized` or `403 Forbidden` to server-side HTTP requests not because authentication is required, but as a Cloudflare JA3/JA4 fingerprinting or bot-detection wall. These are not genuine broken links. When checking any URL with `WebFetch`:
  1. **Pass 1** — fetch the URL directly. If `200–299`, the link is OK.
  2. **Pass 2** — if `400–499`, check the status: `401`, `403`, or `405` all indicate the server is alive and responding — treat these as **likely OK** (CDN/bot-protection). A `404` is a genuine broken link. A `5xx` means the server is down.
  3. **Pass 3** — if still uncertain after pass 2, attempt a HEAD request. `401`, `403`, or `405` on HEAD confirms the server is reachable; report the link as **likely OK** with the actual status code noted.
  Only report a link as broken if it returns `404` or a network-level failure (no response at all).

- **Development and build files included in distribution zip** — files like `docs/`, `generate-*.js`, `build.sh`, and other dev tooling must be excluded from the zip submitted to WordPress.org. These are not plugin code and can contain URLs (e.g. CDN download links) that trigger the "calling files remotely" violation. Add all such paths to the rsync/zip exclusion rules. Verify with `unzip -l plugin.zip | grep -E 'docs/|generate-|build\.'` before submitting.

- **Trademark in plugin name/slug — ownership assertion required** — if the plugin name or slug contains a brand name (trademark) at the beginning, WordPress.org reviewers will flag it as a potential trademark issue and ask you to prove ownership or rename. If you own the trademark, reply explicitly: "I am the founder/owner of [Brand] and own this trademark." A brief direct statement is sufficient. If you do not own it, restructure the name as "DescriptiveName for Trademark" (trademark at the end).

- **Short prefix (< 4 characters)** — WordPress.org requires every function, class, constant (`define()`), option key (`update_option()`), transient, post meta key, registered hook name, AJAX action suffix (`wp_ajax_{action}`), and script/style handle to be prefixed with a **unique prefix of at least 4 characters**. A 2- or 3-character prefix such as `cs_`, `my_`, or `ab_` is explicitly rejected. Count only the characters before the first underscore — `cs_` is 2 characters. Prefixes `wp_`, `_`, and `__` are reserved for WordPress core and must not be used. Choose a prefix derived from your plugin name or slug (e.g. `csbr_` for "CloudScale Backup & Restore"). The review email will list every affected identifier.

- **Writing files to the plugin directory** — plugins must never write any file inside `WP_PLUGIN_DIR` or any path under `plugin_dir_path()`. Plugin directories are deleted on upgrade, so stored files are lost. Files written there are also publicly accessible. WordPress.org flags any `copy()`, `file_put_contents()`, or equivalent call whose destination resolves to the plugin folder. Fix: use `wp_upload_dir()['basedir'] . '/plugin-slug/'` for persistent file storage, or the WordPress options API for settings data. Note: `wp_upload_dir()` must be called at runtime (inside a function), not at file-load time.

- **Remote asset offloading from own server or CDN** — loading JS, CSS, images, or any non-WordPress-core file from your own domain, S3 bucket, or CDN is prohibited. All assets must be bundled locally in the plugin zip and served via `wp_enqueue_script()` / `wp_enqueue_style()`. A remote download URL (e.g. `<a href="https://your-s3.amazonaws.com/plugin.zip">`) inside a shipped HTML help page also triggers this violation. Permitted exceptions: Google Fonts (GPL-compatible), oEmbed provider calls, API callbacks to your own service (document in `== External services ==`). Fix: bundle all assets locally; exclude dev/help HTML files from the distribution zip.

- **Cloud metadata endpoints (link-local IPs) treated as external services** — AWS EC2 Instance Metadata Service (IMDS) at `169.254.169.254`, Azure IMDS, and GCP metadata at `metadata.google.internal` are link-local addresses. Despite being unreachable from the public internet, WordPress.org treats any `wp_remote_*` or cURL call to these endpoints as an external service requiring documentation in `== External services ==`. Document: what the endpoint is (AWS EC2 IMDS for cloud environment auto-detection), what data is exchanged (no user data — a PUT for a token, then a GET for metadata), when it fires, and note that AWS infrastructure has no separate Terms/Privacy URL. Also note: `sslverify => false` on link-local calls is expected (no certificate chain exists), but must be suppressed with `// phpcs:ignore` and a comment explaining why.

- **Versioned asset copies written to plugin directory** — creating versioned copies of JS/CSS files inside the plugin directory (e.g. `script-3-2-0.js` from `script.js`) is not permitted. Plugin directories are deleted on upgrade, so writing there is unreliable. Additionally, WordPress.org flags it as saving data in the plugin folder. Use WordPress's built-in cache-busting: pass the version string as the fourth parameter of `wp_enqueue_script()`/`wp_enqueue_style()` — WordPress appends `?ver=X.X.X` to the URL automatically.

- **Unexpected markdown files in the plugin root** — PCP warns `unexpected_markdown_file` for any `.md` file in the plugin root that is not one of the expected files (`README.md`, `CHANGELOG.md`). Planning, audit, and dev files such as `UX-AUDIT.md`, `TODO.md`, `NOTES.md`, or `DECISIONS.md` must never be committed to the plugin root, or must be excluded from the distribution zip. Fix: delete the file, move it outside the plugin directory, or add it to the rsync/zip exclusion rules used to build the distribution package. Verify with `unzip -l plugin.zip | grep '\.md$'` before submitting — only `README.md` and `CHANGELOG.md` should appear at the root level.

- **Shell execution functions — hard WordPress.org rejection, not a PHPCS suppress** — `shell_exec()`, `exec()`, `system()`, `passthru()`, `proc_open()`, and `popen()` cause immediate rejection by WordPress.org reviewers when found anywhere in plugin code. This is a reviewer judgement call, not a PCP/PHPCS rule — `phpcs:ignore WordPress.PHP.DiscouragedPHPFunctions.system_calls_exec` and even fully-escaped `escapeshellarg()` arguments do not satisfy the review. The current PCP checklist instruction to suppress with phpcs:ignore is only sufficient to pass automated Plugin Check; it does not pass human review. **The only acceptable fix is complete removal.** For backup/recovery plugins that need these capabilities, rewrites using `ZipArchive` (zip), `$wpdb->query()` (DB operations), WP Filesystem API (file ops), and `wp_remote_*` (HTTP) are required. If shell execution is architecturally non-negotiable, the plugin cannot be submitted to WordPress.org in its current form.

- **`WP_CONTENT_DIR` used as a writable storage root** — Writing files to `WP_CONTENT_DIR . '/plugin-slug/'` is rejected. Reviewers require all writable storage to use `wp_upload_dir()['basedir'] . '/plugin-slug/'`. This applies to every custom directory the plugin creates: backup dirs, staging dirs, chunk/temp dirs. `wp_upload_dir()` must be called at runtime inside a function — never at file-load time. Corollary: writing WordPress core drop-in files (e.g. `fatal-error-handler.php`) directly to `WP_CONTENT_DIR` is also flagged; document the intent explicitly in your review reply if this is architecturally required.

- **`WP_PLUGIN_DIR` for your own plugin's file paths** — Using `WP_PLUGIN_DIR . '/' . $your_slug . '/'` to reference your own plugin's files is flagged. For own-plugin paths use `plugin_dir_path( __FILE__ )` (absolute path) and `plugin_dir_url( __FILE__ )` (URL). Define these as constants at plugin boot: `define( 'MYPLUGIN_DIR', plugin_dir_path( __FILE__ ) )`. Using `WP_PLUGIN_DIR` is only appropriate when inspecting *other* plugins by their known relative path (e.g. crash-recovery scanning installed plugins), and even then reviewers will query it — be ready to explain. Corollary: use `WP_LANG_DIR` instead of `WP_CONTENT_DIR . '/languages'` and `WPMU_PLUGIN_DIR` directly instead of `defined('WPMU_PLUGIN_DIR') ? WPMU_PLUGIN_DIR : WP_CONTENT_DIR . '/mu-plugins'` — that constant is always defined in WordPress.

- **`fopen()` on remote URLs** — Flagged by reviewers as equivalent to `file_get_contents()` on remote resources. Many hosts block PHP stream wrappers for remote access. Use `wp_remote_get()` / `wp_remote_post()` (WP HTTP API). Exception: third-party bundled vendor libraries you cannot modify — document this in your review reply.

- **Nonce value must use `sanitize_text_field( wp_unslash() )` before `wp_verify_nonce()`** — Because `wp_verify_nonce()` is a pluggable function, reviewers require its argument to be fully sanitised. Passing only `wp_unslash( $_POST['_wpnonce'] ?? '' )` is flagged. Correct pattern: `wp_verify_nonce( sanitize_text_field( wp_unslash( $_POST['_wpnonce'] ?? '' ) ), 'action_name' )`.

- **Logging unsanitized input before the sanitize call** — Even with a `// phpcs:ignore WordPress.Security.ValidatedSanitizedInput.InputNotSanitized` comment, passing a raw superglobal to a log function on any line *before* the `sanitize_*()` call is flagged by reviewers as processing unsanitized data. Fix: sanitize first, then log. If you genuinely need the raw value for diagnostic purposes, assign it separately, add a comment that this is intentional and logged-only, and never pass it to any other function.

- **Unprotected WP-Cron callback (PHP-FPM crash loop)** — a cron callback registered directly via `add_action( 'hook', $callback )` with no `Throwable`-catching wrapper will kill the PHP-FPM worker if `$callback` throws. PHP 8 turned many previously-silent conditions (e.g. `fread($handle, 0)`, type coercions) into `ValueError`/`TypeError`. The crash flow: uncaught exception → worker SIGSEGV → PHP-FPM spawns a replacement → next page load triggers WP-Cron again → same exception → crash loop → all workers dead → 503. No log entry is written because the worker dies before any logger can flush. **Fix:** never register a cron hook directly. Always use a wrapper that catches `Throwable` and calls `error_log()`. For class-based plugins add a `private static function cron_action( string $hook, callable $callback ): void` helper; for procedural plugins add a prefixed standalone function with the same signature. Pattern: `add_action( $hook, static function () use ( $hook, $callback ): void { try { $callback(); } catch ( \Throwable $e ) { error_log( sprintf( '[plugin] cron "%s" exception (%s): %s in %s line %d', $hook, get_class($e), $e->getMessage(), $e->getFile(), $e->getLine() ) ); } } );`. Every cron registration in the plugin must go through this wrapper — audit with `grep -n "add_action.*cron\|wp_schedule_event" *.php includes/*.php` and verify each hook has a wrapper. Logs appear in PHP error log / `docker logs` container output.
