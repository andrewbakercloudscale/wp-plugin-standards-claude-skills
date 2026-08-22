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

**Then review against the 18 Detailed Plugin Guidelines** (`references/wordpress-org-guidelines.md`) — these are the human-reviewer policy rules that Plugin Check does **not** catch and that cause most repeat rejections of otherwise-clean code. Explicitly check: trialware / locked features (G5), tracking without explicit opt-in (G7), "Powered by"/credit links not opt-in (G10), admin-notice hijacking (G11), bundling libraries WordPress ships (G13), third-party CDN / admin iframes / external code loading (G8), obfuscated or minified-only code (G4), GPL-incompatible bundled assets (G1), and readme spam (G12). Run the audit greps in that reference file.

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
| Critical | Echoed `<script>` or `<style>` tags, missing nonce, unescaped output, raw SQL, WordPress.org ownership or contributor mismatch (`Contributors:` in readme.txt must contain the **actual WordPress.org login username** of the plugin owner — not a brand slug like `cloudscale`; mismatch triggers automated warning before human review), `Author URI` containing a placeholder domain (`example.com` / `example.org` / `example.net`) — automated hard-reject (`plugin_header_invalid_author_uri_domain`), **any URL in readme.txt returning HTTP 404** — the automated pre-reviewer validates `Plugin URI`, `Author URI`, and all links in `== External services ==` (Terms, Privacy) before the submission reaches a human; 404s are reported as failures, hidden files (dot-files) present in the plugin directory, admin page reachable without authentication, REST endpoint with `'__return_true'` permission callback **for an endpoint that (a) returns non-public data (counts or details for private/draft posts, user-specific data) or (b) processes writes without a per-object `current_user_can()` check** — `__return_true` is permitted only when every piece of data returned is already anonymously visible AND the handler itself gates on `get_post_status( $id ) === 'publish'` before returning any per-object data; write endpoints must use `current_user_can( 'edit_post', $id )` or equivalent regardless of nonce validity — a `wp_rest` nonce authenticates session context, not the caller's capability over specific content, `unserialize()` on user-supplied data, file upload without MIME validation, any `shell_exec()`, `exec()`, `system()`, `passthru()`, `proc_open()`, or `popen()` call (WordPress.org reviewers require **complete removal** — `escapeshellarg()` and `phpcs:ignore` do **not** satisfy the review; the plugin will be rejected regardless of how arguments are sanitised), prefix shorter than 4 characters (e.g. `cs_`), files written to the plugin directory or to `WP_CONTENT_DIR` via a write function (`PluginCheck.CodeAnalysis.WriteFile.PluginDirectoryWrite`), executable code (`.php`/`.sh`) deployed to disk at runtime by any means — runtime generation **or** `copy()`/`rename()` of a bundled static file — to any destination **including the uploads directory**, remote asset offloading from own server/CDN, WP-Cron callback registered via bare `add_action()` without a `Throwable`-catching wrapper, i18n string with a printf placeholder missing the `/* translators: */` comment (`WordPress.WP.I18n.MissingTranslatorsComment`), **trialware** — functionality disabled until payment / trial-expiry / licence-key gating (Guideline 5), **tracking/phoning-home without explicit opt-in** (default-off) (Guideline 7), **bundling a library WordPress already ships** (own copy of jQuery / PHPMailer / SimplePie / Backbone, etc.) (Guideline 13), **loading code or assets from a third-party CDN** (non-font) or **embedding admin pages via an external `<iframe>`** or **installing plugins/themes from outside WordPress.org** (Guideline 8), **obfuscated code or minified-only JS/CSS with no source** (Guideline 4), **readme.txt `== Description ==` section over 2,500 words** — the WordPress.org importer truncates it on import and shows "The `Description` section is too long and was truncated. A maximum of 2,500 words is supported." (visible only to authors/committers); content past the limit silently vanishes from the public listing, **i18n text domain passed as a variable** (`WordPress.WP.I18n.NonSingularStringLiteralDomain`) — every `__()`, `_e()`, `esc_html__()`, `esc_attr__()`, `_x()`, `_n()` and all i18n variant calls must pass the `$domain` parameter as a **string literal**, not a variable (`$td`, `$text_domain`, `$this->td`); fires once per call — a class with 25 translated strings generates 25 Critical errors, **`application_detected`** — development/build tooling files (`phpcs.xml.dist`, `.phpcs.xml.dist`, `phpunit.xml.dist`, `package.json`, `composer.json`, `Gruntfile.js`, `webpack.config.js`) included in the distribution zip are rejected before human review, **cryptocurrency mining or botnet code** — any code that mines crypto, participates in distributed computing without explicit user consent, or commandeers server resources for third-party benefit is an immediate hard-rejection and grounds for developer account ban (Guideline 9); **audit:** `grep -rn "coinhive\|cryptonight\|monero\|nicehash\|mining\|botnet\|xmlrpc.*flood\|curl_multi_exec.*broadcast" --include=*.php`, **PHP short tags** (`<?` or `<?=`) — `short_open_tag = Off` (common on many hosts) makes them fail silently; PHPCS/PCP cannot detect missing escaping inside short-tag blocks; every occurrence is flagged, **`ALLOW_UNFILTERED_UPLOADS` set to `true`** or referenced in conditional logic — permits uploading executable `.php` files; immediate hard-rejection regardless of context or intent, **`_e()` or `_ex()` used anywhere in plugin code** — these echo the translation directly without escaping; replace every `_e('text','slug')` with `esc_html_e('text','slug')` or `esc_attr_e('text','slug')` depending on context (one error per call), **HEREDOC (`<<<EOT`) or NOWDOC (`<<<'EOT'`) syntax for any output** — PHPCS/PCP cannot trace escaping through heredoc blocks; reviewers flag all occurrences, **framework or library-only plugin** — plugins that are pure utility templates or pure dependency libraries for other plugins to import/modify are not accepted; every plugin must be self-contained, **100% duplicate of another plugin, or plugin that only reimplements WordPress core functionality** — must provide genuinely new value, **plugin programmatically activates or deactivates other plugins** (calls `activate_plugin()`, `deactivate_plugins()` outside of dependency-error handling) — violates user control, **built-in custom update checker / "phones home" to your own server to check for plugin updates** — WordPress.org provides the update service; all custom update-check code must be removed |
| High | Missing capability check, unsanitised input, bare `die()`, hardcoded URLs, missing ABSPATH guard, admin menu using `'read'` capability, `wp_redirect()` on user-supplied URL (open redirect), user-supplied URL passed to `wp_remote_get()` (SSRF), file path built from user input without traversal check, `is_admin()` used as access-control check, hardcoded API keys or credentials, IDOR (object-level capability not checked), AJAX handler using a nonce helper wrapper instead of `check_ajax_referer()` directly (`NonceVerification.Missing`), **front-end "Powered by"/credit link not opt-in and default-hidden** (Guideline 10), **non-dismissible or site-wide admin notice / dashboard nag / dashboard ad with referral tracking** (Guideline 11), **readme spam** — more than 5 tags, competitor/trademark tags, affiliate links, keyword stuffing (Guideline 12), **GPL-incompatible bundled asset** — image/font/library under a non-GPL-compatible licence (Guideline 1), **`esc_url_raw()` used as HTML output escaper** — it is a sanitiser for DB storage and redirect targets; use `esc_url()` for escaping URLs in HTML attributes and output, **`json_encode()` instead of `wp_json_encode()`** — WordPress alternative required; `json_encode()` bypasses WordPress safety checks, **`ini_set()` called at global scope** (on `init` or at file load) — affects the entire site; scope it to the specific function that needs it, **`date_default_timezone_set()` called anywhere** — WordPress expects UTC internally; calling this breaks `get_post_time()`, `current_time()`, and all WP date helpers, **`error_reporting()` present in committed code** — remove entirely; site operators control this via `WP_DEBUG`, **`filter_var()` / `filter_input()` called without a filter parameter** — `FILTER_DEFAULT` does not sanitise; always specify e.g. `FILTER_SANITIZE_NUMBER_INT`, **iterating over entire `$_POST` / `$_REQUEST` / `$_GET` in a loop** instead of accessing specific named keys, **`esc_html()` applied to content that legitimately contains HTML tags** — strips tags; use `wp_kses_post()` or scoped `wp_kses()` |
| Medium | Duplicate helper functions, missing DocBlocks, version string mismatch, missing CHANGELOG entry, global asset enqueue, `async` JS function without `try/catch` (silent failure), `catch` block with no `console.error()` or user-visible message, `getElementById()` result used without null check, `wp_ajax_nopriv_` used for an admin-only action, `maybe_unserialize()` on externally-sourced option values, **main plugin filename not matching the plugin slug** (slug is derived from `Plugin Name:` header; file must be named `slug.php`), **non-standard file types in plugin zip** (permitted: `.php`, `.js`, `.css`, `.txt`, `.md`, `.png`, `.svg`, `.jpg`, `.json`, `.xml`; anything else requires justification), **`composer.json` absent when plugin uses Composer dependencies** (prevents others from reviewing or forking the dependency tree) |
| Low | Naming convention violations, missing inline comments, non-autoloaded options, minor i18n issues, PHPCS false-positive suppressions missing for WordPress core hook names (`NonPrefixedHooknameFound`) |

**Do not proceed to Step 1 until the user replies with confirmation.**

### Step 0.5 — Mandatory mechanical grep audit

**Run these bash commands immediately after receiving confirmation. Do not skip. These greps catch violations that LLM file-reading routinely misses — they are deterministic and must always run before any file edits.**

```bash
# 1. i18n domain passed as variable (NonSingularStringLiteralDomain) — one Critical per call
grep -rn "__(\|_e(\|esc_html__(\|esc_attr__(\|_x(\|_n(\|esc_html_e(\|esc_attr_e(\|esc_html_x(\|esc_attr_x(" --include=*.php . | grep '\$[a-zA-Z_]' | grep -v "vendor/\|node_modules/"

# 2. cURL usage — hard rejection
grep -rn "curl_init\|curl_exec\|curl_multi_init\|curl_share_init\|curl_file_create" --include=*.php . | grep -v "vendor/\|node_modules/"

# 3. Shell execution — hard rejection
grep -rn "\bshell_exec\b\|\bexec(\|\bsystem(\|\bpassthru(\|\bproc_open\|\bpopen(" --include=*.php . | grep -v "vendor/\|node_modules/"

# 4. _e() / _ex() unescaped output
grep -rn "\b_e(\|\b_ex(" --include=*.php . | grep -v "vendor/\|node_modules/"

# 5. Short tags
grep -rn "<?[^p]" --include=*.php . | grep -v "vendor/\|node_modules/"

# 6. application_detected — tooling files in zip
find . -maxdepth 3 \( -name "phpcs.xml*" -o -name ".phpcs.xml*" -o -name "phpunit.xml*" -o -name "package.json" -o -name "composer.json" -o -name "Gruntfile.js" -o -name "webpack.config.js" \) | grep -v "vendor/"

# 7. OutputNotEscaped — echo/print with unescaped $variables
# Lines containing echo/print AND a $variable but NO escaping function on the same line.
# False-negative caveat: a line like `echo esc_html($a) . $b` has esc_ so it won't appear here —
# read any line with multiple concatenated variables carefully even if it passes this filter.
grep -rn "\becho\b\|\bprint\b\|\bprintf\b\|<?" --include=*.php . \
  | grep '\$[a-zA-Z_]' \
  | grep -v 'esc_html\|esc_attr\|esc_url\|esc_js\|esc_textarea\|esc_sql\|esc_xml\|wp_kses\|absint\|intval\|number_format\|wp_json_encode\|sanitize_\|true\|false\|count(\|strlen(' \
  | grep -v "vendor/\|node_modules/\|//\s*phpcs:ignore"

# 8. parse_url — must use wp_parse_url() (WordPress.WP.AlternativeFunctions.parse_url_parse_url)
grep -rn "\bparse_url\s*(" --include=*.php . | grep -v "vendor/\|node_modules/"

# 9. InputNotSanitized — raw superglobal access; each hit must be sanitized via wp_unslash + sanitize_*
grep -rn "\$_POST\b\|\$_GET\b\|\$_REQUEST\b\|\$_COOKIE\b\|\$_SERVER\b" --include=*.php . \
  | grep -v "vendor/\|node_modules/\|//\s*phpcs:ignore" \
  | grep -v "wp_unslash\|sanitize_\|absint\|intval\|check_ajax_referer\|wp_verify_nonce\|check_admin_referer"

# 10. NonPrefixedClassFound — all class/interface/trait declarations; verify each starts with plugin prefix
grep -rn "^class \|^abstract class \|^final class \|^interface \|^trait " --include=*.php . \
  | grep -v "vendor/\|node_modules/"
```

Report every hit as a Critical finding before proceeding to Step 1.

**Note on grep #7 (OutputNotEscaped):** This grep surfaces most violations but can miss a line that has *both* an escaped variable and an unescaped one (e.g. `echo esc_html($a) . $b`). After running the grep, also read every PHP file section that outputs HTML and confirm every interpolated or concatenated `$variable` is wrapped in `esc_html()`, `esc_attr()`, `esc_url()`, `esc_js()`, or `absint()` at the point of output — not just earlier in the function.

**Note on grep #9 (InputNotSanitized):** This grep surfaces superglobal reads for review; lines that only check `isset( $_POST['key'] )` without using the value are usually fine. Validate that every line where the value is *used* wraps it as `sanitize_text_field( wp_unslash( $_POST['key'] ) )` or equivalent.

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
- WordPress.org `Contributors:` field must contain the **exact WordPress.org login username** of the plugin owner/submitter — not a brand name, company slug, or display name (e.g. `andrewjbaker`, not `cloudscale`). The automated pre-reviewer warns if the submitting account's username is absent.
- All URLs in `readme.txt` must return HTTP 200 — `Plugin URI`, `Author URI`, and every link in `== External services ==` (Terms, Privacy Policy) are validated by the automated pre-reviewer before human review; 404s block submission. Verify with `curl -sI <url> | head -1` before submitting.
- **Code comments must not contain external domain names** — the automated pre-reviewer scans all file content (not just string literals) for URL-like patterns. A PHP doc comment such as `* (as listed at cloudflare.com/ips-v4)` is flagged as "Calling files remotely" even though no request is made. Remove or rewrite domain references in comments to avoid URL patterns (e.g. `// see the Cloudflare IP range documentation` instead of `// cloudflare.com/ips-v4`).
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
- **readme.txt `== Description ==` max 2,500 words** — the WordPress.org importer runs the Description through `wp_trim_words()` (whitespace word split) and truncates anything past 2,500 words on import, surfacing the warning "The `Description` section is too long and was truncated. A maximum of 2,500 words is supported." to authors/committers only. Truncated content is dropped from the public listing with no error to end users. Count with a whitespace split (matches `wp_trim_words`), not `str_word_count()` (which under-counts). Keep the section comfortably under the limit (target ≤ 2,400 words) and move overflow into other readme sections (FAQ, screenshots) or onto the help site. **Audit:** `awk '/^==[[:space:]]*Description[[:space:]]*==/{f=1;next} /^==[[:space:]]/{f=0} f' readme.txt | wc -w` — flag Critical if the result exceeds 2,500.
- **Direct cURL** (`curl_init`, `curl_exec`, etc.) in plugin-authored code is a **hard WordPress.org rejection** — identical treatment to `shell_exec()`. `phpcs:ignore` suppression silences PHPCS/PCP but **human reviewers will still reject it**. Every `curl_exec` call in your own code must be replaced with `wp_remote_get()` / `wp_remote_post()`. **Third-party vendor libraries** containing cURL are permitted — reviewers explicitly distinguish vendor code from plugin-specific code. The sole technically defensible exception in own code is a sub-second connect timeout requirement (e.g. AWS IMDS polling) where `wp_remote_get()` genuinely cannot substitute; suppress with `// phpcs:ignore WordPress.WP.AlternativeFunctions.curl_curl_init, WordPress.WP.AlternativeFunctions.curl_curl_setopt_array, WordPress.WP.AlternativeFunctions.curl_curl_exec, WordPress.WP.AlternativeFunctions.curl_curl_getinfo, WordPress.WP.AlternativeFunctions.curl_curl_close -- wp_remote_get() does not support sub-second connect timeouts` and a comment, but be aware human reviewers may still flag it.

- **cURL in drop-ins / mu-plugins / bundled `assets/*.php` is also rejected — the "HTTP API isn't loaded that early" excuse does not hold** — PCP and human reviewers scan **every** PHP file shipped in the package, including early-loading drop-ins (`fatal-error-handler.php`, `object-cache.php`, `advanced-cache.php`, `db.php`, `sunrise.php`). A drop-in that phones home (e.g. a fatal-error handler sending a crash alert) is the classic place developers reach for `curl_init()`/`curl_exec()` because `wp_remote_post()` may not yet be loaded that early — but the cURL is still flagged (`WordPress.WP.AlternativeFunctions.curl_curl_init` / `_curl_setopt_array` / `_curl_exec` / `_curl_close`) and rejected. **The correct pattern is a `function_exists()` guard that skips the request when the API is unavailable — never a cURL fallback:**
  ```php
  // Best-effort notification. By the time a plugin/theme fatal fires, wp_remote_post()
  // is normally loaded; for a rare fatal before the HTTP API loads, it simply isn't sent.
  if ( function_exists( 'wp_remote_post' ) ) {
      wp_remote_post( $url, [ 'timeout' => 5, 'blocking' => false, 'body' => $body ] );
  }
  ```
  Skipping a best-effort alert in the rare pre-bootstrap-fatal case is acceptable; shipping cURL is not. **Audit:** `grep -rn "curl_init\|curl_exec\|curl_setopt" assets/ includes/ *.php` — drop-in and asset `.php` files are easy to miss because they are not loaded through the main plugin bootstrap.
- **`set_time_limit()`** flagged as discouraged — add `// phpcs:ignore Squiz.PHP.DiscouragedFunctions.Discouraged -- required to prevent PHP timeout on large backups` on each call.
- **Direct DB queries** (`$wpdb->query()` etc.) flagged as discouraged — the complete `phpcs:ignore` comment requires **three** sniff codes, not two: `// phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery, WordPress.DB.DirectDatabaseQuery.NoCaching, PluginCheck.Security.DirectDB.UnescapedDBParameter -- [reason]`. Omitting `PluginCheck.Security.DirectDB.UnescapedDBParameter` leaves a second wave of warnings: Plugin Check traces every variable used in a DB call back to its assignment and flags it as "assigned unsafely" if the assignment is not a recognised safe source. This fires on `$table`, `$cnt`, `$ref_table`, `$date_expr` — any variable in the query string. The proper resolution depends on the variable type:
  - **Table names derived from `$wpdb->prefix`** — use `esc_sql()` at assignment: `$table = esc_sql( $wpdb->prefix . 'plugin_tablename' );`. This satisfies the sniff without a suppression comment, signals intent to reviewers, and is safe because `$wpdb->prefix` contains only alphanumeric characters and underscores.
  - **SQL expressions from an internal conditional** (e.g. `$cnt = $unique ? 'COUNT(DISTINCT visitor_hash)' : 'SUM(view_count)'`) — not user input, cannot be parameterised. Suppress: `// phpcs:ignore PluginCheck.Security.DirectDB.UnescapedDBParameter -- internal SQL expression selected from a hardcoded conditional, not user input`.
  - **`$limit_sql`, `$date_expr`, `$placeholders`** — same pattern: suppress if the value is built entirely from trusted/validated internal data; fix at the assignment with `absint()` / `intval()` / `esc_sql()` if any part could be external.

- **`PluginCheck.Security.DirectDB.UnescapedDBParameter` fires even when `$wpdb->prepare()` is used** — Plugin Check's own sniff runs separately from PHPCS's `WordPress.DB.PreparedSQL.InterpolatedNotPrepared`. A query can pass `$wpdb->prepare()` correctly (all user values parameterised) and still get flagged for `UnescapedDBParameter` on the interpolated table/column names. Both suppressions are needed for each query that uses both interpolated identifiers and prepared value placeholders.
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
| `references/wordpress-org-guidelines.md` | **Always** — the 18 Detailed Plugin Guidelines a *human* reviewer enforces (trialware, opt-in tracking, "powered by", admin-notice hijacking, bundled libraries, CDN/iframe rules, GPL assets, readme spam). These are not caught by PCP and cause most repeat rejections |

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
  - `parse_url()` → `wp_parse_url()` — PCP flags `WordPress.WP.AlternativeFunctions.parse_url_parse_url` as an error on every native call; always use `wp_parse_url()` instead
  - `unlink()` → `wp_delete_file()`
  - `rmdir()` / `readfile()` → WP Filesystem
  - `wp_die('string')` → `wp_die( esc_html__( 'string', 'slug' ) )`
  - **`exec()` / `shell_exec()` / `system()` / `passthru()` → must be removed entirely** — adding `phpcs:ignore` is not a fix; WordPress.org reviewers reject the plugin outright regardless of whether `escapeshellarg()` is used
  - `file_put_contents()` / `file_get_contents()` without phpcs:ignore — `WordPress.WP.AlternativeFunctions.file_system_operations_*`
  - `fopen()` on remote URLs — use `wp_remote_get()` / `wp_remote_post()`; many hosts block PHP stream wrappers for remote access
  - `fread()` / `fclose()` / `fwrite()` → WP Filesystem API (`$wp_filesystem->get_contents()` / `$wp_filesystem->put_contents()`). PCP flags `WordPress.WP.AlternativeFunctions.file_system_operations_fread`, `_fclose`, and `_fwrite` the same way it flags `fopen`; all four functions are violations
  - `slow_db_query_meta_key` — PCP warns whenever `meta_key` appears in a `WP_Query`/`get_posts()` call (`WordPress.DB.SlowDBQuery.slow_db_query_meta_key`). The fix is to ensure the column is indexed, or suppress with `// phpcs:ignore WordPress.DB.SlowDBQuery.slow_db_query_meta_key -- indexed on post_id and meta_key`
  - **`PreparedSQLPlaceholders.ReplacementsWrongNumber`** — `$wpdb->prepare()` received a different number of replacement values than there are `%d`/`%s`/`%f` placeholders in the query string. This is a real bug, not a style warning — fix the placeholder count, do not suppress
  - `wp_verify_nonce()` called with only `wp_unslash()` on the nonce value — must use `sanitize_text_field( wp_unslash( ... ) )` because `wp_verify_nonce()` is pluggable
  - `WP_CONTENT_DIR . '/subfolder/'` for writable storage — use `wp_upload_dir()['basedir'] . '/plugin-slug/'` for all persistent file storage outside the database
  - Logging a raw superglobal value before the `sanitize_*()` call on the same or a later line — WordPress.org flags this even with a `phpcs:ignore InputNotSanitized` comment; sanitize first, log after
  - Any unescaped intermediate variable in HTML output
  - Missing `wp_unslash()` on superglobals — even for integer casts, use `(int) wp_unslash( $_POST['field'] ?? 0 )`
  - **`$_SERVER` superglobals** (`HTTP_HOST`, `REQUEST_URI`, `SCRIPT_NAME`, etc.) require the same treatment as `$_POST`: validate the key exists (`?? ''`), `wp_unslash()`, then `sanitize_text_field()`. PCP fires `InputNotValidated`, `MissingUnslash`, and `InputNotSanitized` on every bare `$_SERVER[...]` access. **Preferred pattern:** avoid `$_SERVER` entirely for URL construction — use `home_url()`, `admin_url()`, `wp_parse_url( home_url(), PHP_URL_HOST )`, and `add_query_arg()` instead. These WordPress helpers are already slashed/sanitised and produce the correct value regardless of server config. Direct `$_SERVER` reads for URL building are almost always replaceable by a WordPress equivalent.
  - Text domain not matching WordPress.org slug (derived from plugin *name*, not folder)
  - readme.txt: >5 tags, >150-char short description
  - `printf()`/`sprintf()` with a placeholder in an i18n string — must have `/* translators: %s: description */` on the line immediately above; PCP flags `WordPress.WP.I18n.MissingTranslatorsComment` as an error on every missing comment
  - `__( 'Text', $td )` / `esc_html__( 'Text', $text_domain )` → `__( 'Text', 'plugin-slug' )` — text domain must be a **string literal** in every i18n call (`WordPress.WP.I18n.NonSingularStringLiteralDomain`); one Critical error per call — a class with a shared `$this->td` property and 25 translated strings generates 25 errors
  - `_e( 'Text', 'slug' )` / `_ex( 'Text', 'ctx', 'slug' )` → `esc_html_e( 'Text', 'slug' )` — `_e()` and `_ex()` output unescaped; replace with the escaping variants
  - `esc_url_raw( $url )` in output context → `esc_url( $url )` — `esc_url_raw()` is a sanitiser, not an output escaper
  - `json_encode( $data )` → `wp_json_encode( $data )` — PCP flags `WordPress.WP.AlternativeFunctions.json_encode_json_encode`
  - `<? ` / `<?=` short tags → `<?php` — `WordPress.PHP.DisallowShortTernary` / short_open_tag violations
  - `ini_set()` at global scope → scope to the function that needs it
  - `date_default_timezone_set()` → remove; use `gmdate()` or `wp_date()`
  - `error_reporting()` in production → remove entirely

- **Admin page publicly reachable** — registering an admin menu with `'read'` capability or omitting a `current_user_can()` check in the render callback makes the page accessible to any logged-in user (Subscribers, Customers). The `add_menu_page()` capability must be at least `'manage_options'` for administrator-only screens. Every render callback and every file in `admin/partials/` must independently re-check the required capability — WordPress only enforces the capability at the menu registration level if the page is reached via the admin menu. Direct URL access bypasses that check. See `references/cyber-security.md §Admin access control`.

- **`is_admin()` used as an access-control check** — `is_admin()` returns `true` whenever WordPress is serving any admin area request, including AJAX calls triggered from the frontend. A subscriber can send an AJAX request with `is_admin()` returning `true`. It is not an authentication or authorisation check. Always use `current_user_can()`.

- **`unserialize()` on user-supplied or option data** — PHP object injection via a crafted serialised string can trigger arbitrary object destructors and methods, potentially leading to remote code execution. Never deserialise any value that originated outside your own plugin's write path. Use `json_decode()` for all data exchange. See `references/cyber-security.md §A08`.

- **`'__return_true'` as REST permission_callback** — makes the endpoint world-readable with no authentication. Every REST endpoint that reads or modifies data must have a non-trivial `permission_callback`. See `references/cyber-security.md §REST API endpoints`.

- **`NonPrefixedHooknameFound` for WordPress core hooks** — when a plugin calls `apply_filters()` or `do_action()` using a WordPress core hook name (e.g. `the_content`, `https_local_ssl_verify`, `robots_txt`), PHPCS warns that the hook name does not start with the plugin prefix. This is a false positive — the plugin is *invoking* a core hook, not *registering* its own. Suppress inline: `// phpcs:ignore WordPress.NamingConventions.PrefixAllGlobals.NonPrefixedHooknameFound -- [hook-name] is a WordPress core filter`. See `references/pcp-checklist.md` §Code quality.

- **`NonPrefixedClassFound` — class name not recognised as starting with the plugin prefix** (`WordPress.NamingConventions.PrefixAllGlobals.NonPrefixedClassFound`) — PHPCS's `PrefixAllGlobals` sniff does a **case-sensitive** prefix match. If the plugin registers `cloudscale` (lowercase) as its prefix in `phpcs.xml`, then a class named `CloudScale_Licence` (PascalCase) is flagged as non-prefixed — even though it clearly belongs to the plugin. This surprises developers who assume the prefix check is case-insensitive. The same applies to interfaces and traits. **Two fixes (choose one):**
  1. **Register all case variants** in `phpcs.xml` (or `.phpcs.xml.dist`) — the simplest fix when renaming classes is impractical:
     ```xml
     <rule ref="WordPress.NamingConventions.PrefixAllGlobals">
       <properties>
         <property name="prefixes" type="array">
           <element value="your_prefix"/>
           <element value="Your_Prefix"/>
           <element value="YOUR_PREFIX"/>
         </property>
       </properties>
     </rule>
     ```
  2. **Rename the class** to match the registered prefix exactly (e.g. `Yourprefix_Licence` if `yourprefix` is registered). This is the cleaner long-term fix but requires updating every reference. **Audit:** `grep -rn "^class \|^abstract class \|^final class \|^interface \|^trait " --include=*.php .` — compare each class name against the prefix(es) in `phpcs.xml`. Any class whose name (case-exact) does not start with a registered prefix is a violation. Also covers `define()` constants and global function names — the same sniff fires for `MYPLUGIN_VERSION` if `myplugin` is registered but `MYPLUGIN` is not.

- **`Author URI` placeholder domain — automated hard-reject** — WordPress.org's automated scanner rejects any plugin where `Author URI:` in the plugin header contains `example.com`, `example.org`, `example.net`, or any RFC 2606 reserved placeholder domain. Error: `plugin_header_invalid_author_uri_domain`. This fires before the submission even reaches human review. **Grep before every submission:** `grep -i "Author URI" plugin-slug.php` and confirm it points to a real URL you own. The same applies to `Plugin URI:` — a 404 or placeholder there is also flagged.

- **Plugin name contains "Free"** — WordPress.org discourages the word "Free" in a plugin display name; it is redundant in the directory. A name like "My Plugin Free" will be rejected and the reviewer will ask you to rename it. Remove "Free" from the display name and slug. If the old slug is already in use (e.g. during submission), request a new slug in your reply email.

- **Missing `== External services ==` section in readme.txt** — if the plugin connects to any third-party or external service (S3, Google Drive, payment gateways, IMDS endpoints, external APIs), the readme.txt **must** contain an `== External services ==` section. For each service, document: what the service is and what it is used for, what data is sent and when, and links to the service's terms of service and privacy policy. This is both a guideline requirement (Guideline 6) and a legal protection for the plugin author. A privacy policy section alone is not sufficient.

- **Privacy policy falsely claims no external services** — if the readme.txt Privacy Policy section says "no external requests of any kind" but the plugin does make external requests (e.g. optional S3 sync, rclone, AWS IMDS), this is flagged in review. Ensure the privacy policy accurately reflects what the plugin does, and cross-reference the `== External services ==` section for the conditional cases.

- **A disclosed endpoint whose stated scope doesn't match its real trigger condition is treated as undisclosed** — naming the URL somewhere in the readme is not sufficient; WordPress.org (and this review) checks whether the **When:** condition attached to it is actually true for every call site that hits it. This has caused repeat rejections in practice: an own-IP lookup (`/whoami`) ran unconditionally from cron and from an always-visible admin panel, but the only readme mention of it lived inside a subsection titled "optional, paid monthly subscription" whose own **When:** clause read "AI data only while Use Managed AI is on... Billing data only when you click Subscribe/Cancel/...". Neither condition covered the unconditional call, so the endpoint read as undocumented for the use that actually mattered — even though the literal string was present in the file. The fix is not "add the URL to the readme" (it was already there); it is "make the readme's scoping language true for every call site." **Audit:** for each `wp_remote_*`/`curl_init` call whose full URL includes a path segment, find every readme subsection that mentions that URL, read its **When:**/scope sentence, then check the *code path* actually calling it — is it gated by the option/feature/subscription check the readme claims, or does it fire from `cron`, `init`, an unconditional method, or a different feature entirely? If the code path and the stated scope disagree, that is a Critical finding: either move the endpoint's disclosure to (or add one under) the subsection whose scope is actually true, or fix the subsection's **When:** clause to cover every real trigger. Do this for every distinct call site, not just once per host+path — the same endpoint can be genuinely disclosed for one caller and undisclosed for another.

- **Dynamically-built URLs (constant + variable, or a `_base_url()`/`_url()` helper method) must be resolved before judging disclosure, not skipped** — `self::BROKER . $path` or `$this->paypal_base_url() . '/v2/checkout/orders/' . $id . '/capture'` never appears as a literal `https://...` string at the call site, so a naive text/regex scan (including this repo's own `check-fetched-urls-documented.php`) will not see it as a URL at all and will silently pass over it — not flag it, just never evaluate it. Do not let that silence read as compliance. **Audit:** grep for `wp_remote_(get|post|request|head)\s*\(\s*\$` and for any `. $` / `.$` concatenation feeding those calls; for each one, trace the constant or the called method's own body (including every literal it can return across branches, e.g. sandbox vs. live) to reconstruct the real URL(s), then check disclosure for each reconstructed path exactly as you would a literal one. A base URL used by five different paths needs five path-level disclosures (or a documented family), not one.

- **readme.txt Description over 2,500 words — silently truncated on import** — the WordPress.org plugin importer caps the `== Description ==` section at 2,500 words. Anything past the limit is dropped from the public plugin page, and the only signal is an author/committer-only notice on the plugin page: "During the last import of your plugin the following warnings were encountered… The `Description` section is too long and was truncated. A maximum of 2,500 words is supported." End users never see the warning, and the build/PCP toolchain does not catch it — so a bloated Description ships and the bottom of the listing quietly disappears. The importer measures with `wp_trim_words()`, which counts words by splitting on whitespace (`[\n\r\t ]+`); this counts **higher** than PHP's `str_word_count()` (which discards numbers and most punctuation) and roughly matches `wc -w`, so always measure with a whitespace split — never trust `str_word_count()` for this check. **Audit:** `awk '/^==[[:space:]]*Description[[:space:]]*==/{f=1;next} /^==[[:space:]]/{f=0} f' readme.txt | wc -w`. Flag **Critical** if the count exceeds 2,500. Fix: trim the Description to ≤ 2,400 words (leave headroom — markdown markup and shortcodes can shift the parsed count) by relocating overflow into the `== Frequently Asked Questions ==` section, `== Screenshots ==` captions, or the external help site, keeping only the core feature summary in the Description. Re-run the audit after trimming to confirm.

- **Plugin URI returns 404** — the `Plugin URI:` header in the plugin file must point to a real, reachable URL. A 404 is flagged by reviewers. Use a URL you control (`https://yoursite.com`) or the WordPress.org plugin page (`https://wordpress.org/plugins/your-slug/`) once approved. When checking links, follow the three-pass rule below.

- **Broken link false positives from bot-protection walls** — many high-traffic sites (Reuters, WatchMojo, etc.) return `401 Unauthorized` or `403 Forbidden` to server-side HTTP requests not because authentication is required, but as a Cloudflare JA3/JA4 fingerprinting or bot-detection wall. These are not genuine broken links. When checking any URL with `WebFetch`:
  1. **Pass 1** — fetch the URL directly. If `200–299`, the link is OK.
  2. **Pass 2** — if `400–499`, check the status: `401`, `403`, or `405` all indicate the server is alive and responding — treat these as **likely OK** (CDN/bot-protection). A `404` is a genuine broken link. A `5xx` means the server is down.
  3. **Pass 3** — if still uncertain after pass 2, attempt a HEAD request. `401`, `403`, or `405` on HEAD confirms the server is reachable; report the link as **likely OK** with the actual status code noted.
  Only report a link as broken if it returns `404` or a network-level failure (no response at all).

- **Bundled vendor/library directories produce PHPCS errors in Plugin Check** — Plugin Check runs PHPCS against every `.php` file in the plugin zip, including third-party libraries under `lib/`, `vendor/`, or similar subdirectories. These libraries (MaxMind DB reader, Guzzle, Monolog, etc.) are written for generic PHP and typically contain dozens of `ExceptionNotEscaped`, `file_system_operations_fread/fclose`, `InterpolatedNotPrepared`, and similar violations that are false positives for library code. **The correct fix is a `.phpcs.xml.dist` file in the plugin root** with exclusion patterns — Plugin Check respects this file:

  ```xml
  <?xml version="1.0"?>
  <ruleset name="Plugin Name Standards">
      <rule ref="WordPress"/>
      <exclude-pattern>*/lib/*</exclude-pattern>
      <exclude-pattern>*/vendor/*</exclude-pattern>
  </ruleset>
  ```

  Add one `<exclude-pattern>` line per bundled library directory. Do **not** add `phpcs:ignoreFile` comments to vendor files — modifying third-party code makes future upgrades painful and may introduce merge conflicts. The `.phpcs.xml.dist` approach is zero-touch. **Audit:** if Plugin Check shows errors only in `lib/` or `vendor/` subdirectories, the plugin is missing this config file. Create it before re-running Plugin Check.

  **Limitation — PCP file-level checks vs PHPCS checks:** `.phpcs.xml.dist` exclusions suppress PHPCS-based sniffs (which covers the vast majority: `ExceptionNotEscaped`, `file_system_operations_*`, `NonPrefixedFunctionFound`, `PluginCheck.Security.DirectDB.UnescapedDBParameter`, etc.). However, Plugin Check has some file-level checks that run outside PHPCS and are **not** suppressed by `.phpcs.xml.dist`. The most common example is **`missing_direct_file_access_protection`** — this fires when a PHP file does not begin with `if ( ! defined( 'ABSPATH' ) ) { exit; }`. If this error still appears on vendor files after adding `.phpcs.xml.dist`, the fix is to prepend that one-line guard immediately after the opening `<?php` tag in the vendor file. This is safe — the file is always loaded via `require`/`include` from within a live WordPress installation where `ABSPATH` is always defined, so the guard never actually triggers.

- **`ExceptionNotEscaped` — PHP exceptions flagged as unescaped output** — PCP fires `WordPress.Security.EscapeOutput.ExceptionNotEscaped` when a variable is used as the message in a `throw new \Exception(...)` statement, because the sniff treats any variable as potential HTML output. Exception messages are never rendered as HTML — they are caught internally, written to logs, or converted to WP error responses. For own code, suppress inline: `// phpcs:ignore WordPress.Security.EscapeOutput.ExceptionNotEscaped -- exception message is not HTML output`. For third-party library code, suppress via `.phpcs.xml.dist` exclusion (see above) rather than modifying vendor files.

- **`InterpolatedNotPrepared` at scale in analytics/stats plugins** — Analytics plugins that use custom DB tables (separate tables for page views, referrers, sessions, geo data, etc.) generate this warning dozens of times because `$wpdb->prepare()` cannot parameterise table names or column-name expressions — SQL identifiers cannot be bound as placeholders. This is **expected and acceptable** when:
  - `$table` is derived from `$wpdb->prefix . 'plugin_tablename'` (WordPress table prefix + hardcoded suffix — never user input)
  - `$cnt` / `$col` is selected from an internal allowlist (e.g. `$cnt = $unique ? 'COUNT(DISTINCT visitor_hash)' : 'SUM(view_count)'`) validated by the plugin's own logic, never `$_POST`/`$_GET`
  
  The suppression comment must go on the **same line** as the query string for single-line queries, or inside a `phpcs:disable` / `phpcs:enable` block for multi-line queries. Include the justification — reviewers spot-check these:

  ```php
  // Single-line — phpcs:ignore on the same line as the string:
  $results = $wpdb->get_results( $wpdb->prepare( // phpcs:ignore WordPress.DB.PreparedSQL.InterpolatedNotPrepared -- $table is $wpdb->prefix value; $cnt is internal enum
      "SELECT post_id, {$cnt} AS views FROM `{$table}` WHERE viewed_at >= %s",
      $since
  ) );

  // Multi-line — phpcs:disable / phpcs:enable block:
  // phpcs:disable WordPress.DB.PreparedSQL.InterpolatedNotPrepared -- $table is $wpdb->prefix value
  $results = $wpdb->get_results(
      $wpdb->prepare(
          "SELECT {$cnt} AS views\n FROM `{$table}`\n WHERE viewed_at BETWEEN %s AND %s",
          $start, $end
      )
  );
  // phpcs:enable WordPress.DB.PreparedSQL.InterpolatedNotPrepared
  ```

  **Do not suppress when the interpolated value derives from user input** — that is a real SQL injection risk requiring `$wpdb->esc_sql()` or query restructuring. A plugin with dozens of custom-table queries should expect 50–100 of these warnings and plan the suppression comments up front, not after receiving PCP output.

- **Development and build files included in distribution zip** — files like `docs/`, `generate-*.js`, `build.sh`, and other dev tooling must be excluded from the zip submitted to WordPress.org. These are not plugin code and can contain URLs (e.g. CDN download links) that trigger the "calling files remotely" violation. Add all such paths to the rsync/zip exclusion rules. Verify with `unzip -l plugin.zip | grep -E 'docs/|generate-|build\.'` before submitting.

- **Trademark in plugin name/slug — ownership assertion required** — if the plugin name or slug contains a brand name (trademark) at the beginning, WordPress.org reviewers will flag it as a potential trademark issue and ask you to prove ownership or rename. If you own the trademark, reply explicitly: "I am the founder/owner of [Brand] and own this trademark." A brief direct statement is sufficient. If you do not own it, restructure the name as "DescriptiveName for Trademark" (trademark at the end).

- **Short prefix (< 4 characters)** — WordPress.org requires every function, class, constant (`define()`), option key (`update_option()`), transient, post meta key, registered hook name, AJAX action suffix (`wp_ajax_{action}`), and script/style handle to be prefixed with a **unique prefix of at least 4 characters**. A 2- or 3-character prefix such as `cs_`, `my_`, or `ab_` is explicitly rejected. Count only the characters before the first underscore — `cs_` is 2 characters. Prefixes `wp_`, `_`, and `__` are reserved for WordPress core and must not be used. Choose a prefix derived from your plugin name or slug (e.g. `csbr_` for "CloudScale Backup & Restore"). The review email will list every affected identifier.

- **Writing files to the plugin directory** — plugins must never write any file inside `WP_PLUGIN_DIR` or any path under `plugin_dir_path()`. Plugin directories are deleted on upgrade, so stored files are lost. Files written there are also publicly accessible. WordPress.org flags any `copy()`, `file_put_contents()`, or equivalent call whose destination resolves to the plugin folder. PCP error code: **`PluginCheck.CodeAnalysis.WriteFile.PluginDirectoryWrite`** ("Plugin folders are deleted when upgraded. Do not save data to the plugin folder using `copy()`."). **Important — this sniff also fires on `WP_CONTENT_DIR`:** the message names the plugin folder, but the underlying detection triggers on the *constant used to build the path* combined with a write function. A call like `copy( $src, WP_CONTENT_DIR . '/slug/file' )` reports `PluginCheck.CodeAnalysis.WriteFile.PluginDirectoryWrite` with "Detected usage of constant `WP_CONTENT_DIR`. Use `wp_upload_dir()` to get the uploads directory path or save to the database instead." — so do not assume this code only means a literal `WP_PLUGIN_DIR` path. Fix: use `wp_upload_dir()['basedir'] . '/plugin-slug/'` for persistent file storage, or the WordPress options API for settings data. Note: `wp_upload_dir()` must be called at runtime (inside a function), not at file-load time.

- **Remote asset offloading from own server or CDN** — loading JS, CSS, images, or any non-WordPress-core file from your own domain, S3 bucket, or CDN is prohibited. All assets must be bundled locally in the plugin zip and served via `wp_enqueue_script()` / `wp_enqueue_style()`. A remote download URL (e.g. `<a href="https://your-s3.amazonaws.com/plugin.zip">`) inside a shipped HTML help page also triggers this violation. Permitted exceptions: Google Fonts (GPL-compatible), oEmbed provider calls, API callbacks to your own service (document in `== External services ==`). Fix: bundle all assets locally; exclude dev/help HTML files from the distribution zip.

- **Cloud metadata endpoints (link-local IPs) treated as external services** — AWS EC2 Instance Metadata Service (IMDS) at `169.254.169.254`, Azure IMDS, and GCP metadata at `metadata.google.internal` are link-local addresses. Despite being unreachable from the public internet, WordPress.org treats any `wp_remote_*` or cURL call to these endpoints as an external service requiring documentation in `== External services ==`. Document: what the endpoint is (AWS EC2 IMDS for cloud environment auto-detection), what data is exchanged (no user data — a PUT for a token, then a GET for metadata), when it fires, and note that AWS infrastructure has no separate Terms/Privacy URL. Also note: `sslverify => false` on link-local calls is expected (no certificate chain exists), but must be suppressed with `// phpcs:ignore` and a comment explaining why.

- **Versioned asset copies written to plugin directory** — creating versioned copies of JS/CSS files inside the plugin directory (e.g. `script-3-2-0.js` from `script.js`) is not permitted. Plugin directories are deleted on upgrade, so writing there is unreliable. Additionally, WordPress.org flags it as saving data in the plugin folder. Use WordPress's built-in cache-busting: pass the version string as the fourth parameter of `wp_enqueue_script()`/`wp_enqueue_style()` — WordPress appends `?ver=X.X.X` to the URL automatically.

- **Unexpected markdown files in the plugin root** — PCP warns `unexpected_markdown_file` for any `.md` file in the plugin root that is not one of the expected files (`README.md`, `CHANGELOG.md`). Planning, audit, and dev files such as `UX-AUDIT.md`, `TODO.md`, `NOTES.md`, or `DECISIONS.md` must never be committed to the plugin root, or must be excluded from the distribution zip. Fix: delete the file, move it outside the plugin directory, or add it to the rsync/zip exclusion rules used to build the distribution package. Verify with `unzip -l plugin.zip | grep '\.md$'` before submitting — only `README.md` and `CHANGELOG.md` should appear at the root level.

- **Shell execution functions — hard WordPress.org rejection, not a PHPCS suppress** — `shell_exec()`, `exec()`, `system()`, `passthru()`, `proc_open()`, and `popen()` cause immediate rejection by WordPress.org reviewers when found anywhere in plugin code. This is a reviewer judgement call, not a PCP/PHPCS rule — `phpcs:ignore WordPress.PHP.DiscouragedPHPFunctions.system_calls_exec` and even fully-escaped `escapeshellarg()` arguments do not satisfy the review. The current PCP checklist instruction to suppress with phpcs:ignore is only sufficient to pass automated Plugin Check; it does not pass human review. **The only acceptable fix is complete removal.** For backup/recovery plugins that need these capabilities, rewrites using `ZipArchive` (zip), `$wpdb->query()` (DB operations), WP Filesystem API (file ops), and `wp_remote_*` (HTTP) are required. If shell execution is architecturally non-negotiable, the plugin cannot be submitted to WordPress.org in its current form.

- **`WP_CONTENT_DIR` (including the `/wp-content` root) used as writable storage** — Writing files to `WP_CONTENT_DIR . '/plugin-slug/'` is rejected, and writing to the `/wp-content` **root** directly (e.g. a companion `WP_CONTENT_DIR . '/cloudscale-backup-par.json'`) is also rejected — this applies to plain data files such as JSON config, not just code. PCP error code: **`PluginCheck.CodeAnalysis.WriteFile.PluginDirectoryWrite`** — despite the "PluginDirectory" name, this is the sniff that fires on `WP_CONTENT_DIR` ("Detected usage of constant `WP_CONTENT_DIR`. Use `wp_upload_dir()` to get the uploads directory path or save to the database instead.") whenever the constant feeds a write function (`copy()`, `file_put_contents()`, `fwrite()`, `fopen()` in write mode, `move_uploaded_file()`, etc.). Reviewers recognise only a narrow set of `/wp-content` exceptions: `cache/`, `backups/` (for backup plugins, and even then scrutinised), and core drop-ins written to their canonical drop-in path. Everything else must use `wp_upload_dir()['basedir'] . '/plugin-slug/'`. This applies to every custom directory the plugin creates: backup dirs, staging dirs, chunk/temp dirs, and any sidecar/companion data file. `wp_upload_dir()` must be called at runtime inside a function — never at file-load time. Corollary: writing WordPress core drop-in files (e.g. `fatal-error-handler.php`) directly to `WP_CONTENT_DIR` is also flagged; document the intent explicitly in your review reply if this is architecturally required. **Audit gap:** grepping only for `WP_CONTENT_DIR` at call sites misses violations hidden inside helper functions (e.g. `function backup_dir() { return WP_CONTENT_DIR . '/cloudscale-backups/'; }`). Read every function that returns a file path and trace its return value — do not stop at the grep hit. Also grep for the plugin-specific directory name as a string literal (e.g. `cloudscale-backups`) to surface hardcoded paths that bypass the constant entirely.

- **`WP_PLUGIN_DIR` for your own plugin's file paths** — Using `WP_PLUGIN_DIR . '/' . $your_slug . '/'` to reference your own plugin's files is flagged. For own-plugin paths use `plugin_dir_path( __FILE__ )` (absolute path) and `plugin_dir_url( __FILE__ )` (URL). Define these as constants at plugin boot: `define( 'MYPLUGIN_DIR', plugin_dir_path( __FILE__ ) )`. Using `WP_PLUGIN_DIR` is only appropriate when inspecting *other* plugins by their known relative path (e.g. crash-recovery scanning installed plugins), and even then reviewers will query it — be ready to explain. Corollary: use `WP_LANG_DIR` instead of `WP_CONTENT_DIR . '/languages'` and `WPMU_PLUGIN_DIR` directly instead of `defined('WPMU_PLUGIN_DIR') ? WPMU_PLUGIN_DIR : WP_CONTENT_DIR . '/mu-plugins'` — that constant is always defined in WordPress.

- **Writing to system paths — hard rejection** — plugins must never write, create, or delete files at OS-level paths: `/usr/local/bin`, `/etc/cron.d`, `/var/log`, or any path outside the WordPress installation root. WordPress.org categorises this under "saving data in the plugin folder" even when the violation is *outside* WordPress entirely. The correct pattern: all persistent storage uses `wp_upload_dir()['basedir'] . '/plugin-slug/'`; log files use a path within that directory; scheduled tasks use WP-Cron, never a system cron entry written to `/etc/cron.d`. Server-side scripts that must reside on the OS (e.g. a watchdog bash script) cannot be auto-installed by the plugin — the plugin must provide the script content and cron line as copy-paste text for a server administrator to apply manually. **`uninstall.php` is not exempt:** removing system-path files during uninstall is equally prohibited; omit any `wp_delete_file()` or filesystem call whose target resolves outside WordPress. **Audit:** `grep -rn "/usr/local\|/etc/cron\|/var/log\|chmod\|chown" *.php includes/*.php uninstall.php` and review every hit.

- **Deploying executable code to disk at runtime — hard rejection (generation *and* `copy()` of bundled files)** — plugins must not place executable code files (`.php`, `.sh`, or any script the server will run) on disk at runtime, by **any** mechanism and to **any** destination. This covers two patterns that are commonly mistaken for each other:
  - **Runtime code generation** — assembling PHP/shell source as strings and writing it via `file_put_contents()`, `fwrite()`, or a WP Filesystem write (constructing `<?php ... ?>` bodies, building plugin headers, interpolating variables into code syntax). Flagged as arbitrary-code-execution and obfuscation risk.
  - **Copying a pre-bundled static code file into a runtime location** — e.g. `copy( PLUGIN_DIR . 'assets/crash-test.php', $dest )`. This is **not** a valid workaround for the above, and getting caught here is the single most common repeat-rejection on this guideline. WordPress.org rejects it regardless of (a) whether the content was generated or shipped as a static asset, and (b) the destination — **including the uploads directory.** The uploads dir is for *data and media*, never executable code; copying a `.php` file there (`wp_upload_dir()['basedir'] . '/slug/foo.php'`) is rejected just as firmly as writing into `WP_PLUGIN_DIR`. There is no "approved" location for plugin-deployed executable code.
  
  **There is no compliant way to make the plugin install another plugin/drop-in/script at runtime.** If a feature architecturally requires deploying executable code (a crash-test plugin, a watchdog script, a custom drop-in), that feature cannot ship to WordPress.org as an automated action — provide the file content and installation steps as copy-paste text for a site/server administrator to apply manually, exactly as required for system-path scripts (see "Writing to system paths"). Moving the destination from the plugin dir to uploads does not fix it; only removing the runtime write does. **Audit:** `grep -n "file_put_contents\|fwrite\|fputs\|\bcopy(\|\brename(\|move_uploaded_file" *.php includes/*.php` — inspect every hit. Hard violation if (i) the content argument contains `<?php` / assembles code tokens, **or** (ii) a `copy()`/`rename()`/`move_uploaded_file()` destination is a `.php`/`.sh` filename, *even when the source is a bundled static asset and the destination is inside `wp_upload_dir()`*.

- **`fopen()` on remote URLs** — Flagged by reviewers as equivalent to `file_get_contents()` on remote resources. Many hosts block PHP stream wrappers for remote access. Use `wp_remote_get()` / `wp_remote_post()` (WP HTTP API). Exception: third-party bundled vendor libraries you cannot modify — document this in your review reply.

- **Nonce value must use `sanitize_text_field( wp_unslash() )` before `wp_verify_nonce()`** — Because `wp_verify_nonce()` is a pluggable function, reviewers require its argument to be fully sanitised. Passing only `wp_unslash( $_POST['_wpnonce'] ?? '' )` is flagged. Correct pattern: `wp_verify_nonce( sanitize_text_field( wp_unslash( $_POST['_wpnonce'] ?? '' ) ), 'action_name' )`.

- **Logging unsanitized input before the sanitize call** — Even with a `// phpcs:ignore WordPress.Security.ValidatedSanitizedInput.InputNotSanitized` comment, passing a raw superglobal to a log function on any line *before* the `sanitize_*()` call is flagged by reviewers as processing unsanitized data. Fix: sanitize first, then log. If you genuinely need the raw value for diagnostic purposes, assign it separately, add a comment that this is intentional and logged-only, and never pass it to any other function.

- **`WordPress.WP.I18n.NonSingularStringLiteralDomain` — i18n domain passed as a variable** — every WordPress i18n function (`__()`, `_e()`, `esc_html__()`, `esc_attr__()`, `_x()`, `_n()`, and all variants) requires the `$domain` parameter to be a **single string literal**, not a variable. Passing `__( 'Text', $td )`, `esc_html__( 'Text', $text_domain )`, or `__( 'Text', $this->text_domain )` — even when the variable holds the correct slug — generates one PCP Critical error per call. A class with a shared `$td` constructor parameter and 25 translated strings generates 25 Critical errors, which produces a long rejection email that looks catastrophic but has a single fix. **Fix:** replace the variable argument with the literal text domain slug at every i18n call site — `__( 'Text', 'my-plugin-slug' )`. The variable itself can remain for other uses; only the i18n call sites need the literal. There is no `phpcs:ignore` that satisfies the automated checker — the literal string is the only fix. **Audit:** `grep -rn "__(\|_e(\|esc_html__(\|esc_attr__(\|_x(\|_n(\|esc_html_e(\|esc_attr_e(" includes/ *.php | grep '\$'` — every match where the final argument before `)` is a `$variable` is a violation. This applies equally to a licence class, notification class, or any shared utility class that conventionally stores the text domain in a property to avoid repetition — WordPress.org's automated checker has no mechanism to trace the variable's value, so it must be a literal at every call site, without exception.

- **`application_detected: Application files are not permitted`** — WordPress.org's automated scanner rejects plugins that include development tooling or build configuration files in the submitted zip. Common culprits: `phpcs.xml`, `phpcs.xml.dist`, `.phpcs.xml`, `.phpcs.xml.dist`, `phpunit.xml`, `phpunit.xml.dist`, `Gruntfile.js`, `webpack.config.js`, `package.json`, `composer.json`, `Makefile`, `.babelrc`, `.eslintrc`, `.editorconfig`. **Critical note:** the `.phpcs.xml.dist` workaround documented above for suppressing vendor PHPCS errors is the correct approach — but the file must **never** appear in the submitted zip. Create it for local/CI use and ensure it is listed in the rsync/zip exclusion rules that build the distribution package. The same applies to `docs/`, build scripts (`build.sh`, `generate-help-docs.js`), test configs, and CI definition files. **Fix:** add every tooling file path to the exclusion list used when building the distribution zip. **Audit:** `unzip -l plugin.zip | grep -E 'phpcs\.xml|phpunit|Gruntfile|webpack\.config|package\.json|composer\.json|Makefile|\.babelrc|\.eslintrc|\.editorconfig|generate-|build\.sh|/docs/'` — any hit is a rejection.

- **`_e()` and `_ex()` output unescaped translations** — `_e( 'text', 'slug' )` and `_ex( 'text', 'context', 'slug' )` echo the translated string with no escaping, creating an XSS vector. PCP flags every call. **Fix:** replace `_e(...)` with `esc_html_e(...)` or `esc_attr_e(...)` depending on context. Similarly, `echo __( 'text', 'slug' )` is flagged — wrap: `echo esc_html__( 'text', 'slug' )`. **Audit:** `grep -rn "\b_e(\|\b_ex(" --include=*.php` — every hit is a violation.

- **`esc_url_raw()` used as HTML output escaper** — `esc_url_raw()` is a *sanitiser* designed for storing URLs in the database or passing them to `wp_redirect()`. It does not encode `'` and `"`, making it unsafe for HTML attribute output. Using it in `echo`, template output, or `esc_attr()` substitution is flagged. **Fix:** use `esc_url()` for all URL output in HTML. The distinction: sanitise with `esc_url_raw()` when saving; escape with `esc_url()` when outputting.

- **`json_encode()` instead of `wp_json_encode()`** — PCP flags `WordPress.WP.AlternativeFunctions.json_encode_json_encode` on every native `json_encode()` call. `wp_json_encode()` adds WordPress-specific safety checks and returns `false` on failure rather than throwing. **Fix:** global replace `json_encode(` → `wp_json_encode(` — a one-line mechanical change with no behaviour difference for valid input.

- **PHP short tags (`<?` / `<?=`)** — Many production hosts set `short_open_tag = Off`, causing `<?` to render as literal text and breaking the plugin entirely. PCP also flags every short tag because PHPCS cannot trace escaping through short-echo syntax (`<?= $var ?>`). **Fix:** always use `<?php` and `<?php echo`. **Audit:** `grep -rn "^<?\b\|[^p]<?\b" --include=*.php` — any `<?` not followed immediately by `php` is a violation.

- **`ALLOW_UNFILTERED_UPLOADS` set to `true`** — Defining `ALLOW_UNFILTERED_UPLOADS` as `true`, or referencing it in any conditional that bypasses MIME validation, is an immediate hard-rejection. It permits uploading executable `.php` files, creating a remote code execution vector. There is no legitimate reason for a plugin to set this constant. **Fix:** remove all references. If specific file types are legitimately needed, add them to the `upload_mimes` filter instead.

- **`ini_set()` at global scope** — Calling `ini_set( 'memory_limit', '-1' )` or `ini_set( 'max_execution_time', 120 )` on `init` or at file load affects the entire site and overrides host restrictions. PCP flags this and reviewers require it to be scoped. **Fix:** move `ini_set()` calls inside the specific function that needs the limit (e.g. inside a large export/import function), not at plugin load time.

- **`date_default_timezone_set()` called anywhere** — WordPress stores all timestamps in UTC and converts them to local time via `wp_date()` and `get_option('timezone_string')`. Calling `date_default_timezone_set()` overrides the PHP timezone globally, breaking `get_post_time()`, `current_time()`, and any plugin that relies on UTC-based calculations. **Fix:** remove all calls. Use `gmdate()` for UTC output and `wp_date()` for site-local display; never manipulate the default timezone.

- **`error_reporting()` in committed code** — Calling `error_reporting(0)` or `error_reporting(E_ALL)` in plugin code prevents site operators from using `WP_DEBUG` reliably and is flagged by reviewers. **Fix:** delete every `error_reporting()` call — leave PHP error reporting entirely to the site's configuration.

- **HEREDOC / NOWDOC syntax in output** — While PHP HEREDOC (`<<<EOT`) and NOWDOC (`<<<'EOT'`) are valid PHP, PHPCS/PCP cannot trace variable escaping inside heredoc blocks. A reviewer cannot verify that output is escaped, so every file containing HEREDOC or NOWDOC for output is flagged. **Fix:** convert to standard string concatenation (`echo '<p>' . esc_html($var) . '</p>'`) or use `printf()` with escaped format strings. HEREDOC is also explicitly banned from WordPress.org-destined plugin code by the plugin review team.

- **`filter_var()` / `filter_input()` without a filter parameter** — Calling `filter_var( $val )` or `filter_input( INPUT_POST, 'key' )` with no filter argument defaults to `FILTER_DEFAULT`, which performs no sanitisation — it is equivalent to reading the raw value. PCP flags this as unsanitised input. **Fix:** always specify the filter: `filter_var( $val, FILTER_SANITIZE_NUMBER_INT )` or use WordPress-native `sanitize_text_field()`, `absint()`, etc.

- **Iterating over entire `$_POST` / `$_REQUEST` / `$_GET` superglobal** — Looping over `$_POST` as an array (e.g. `foreach ( $_POST as $key => $val )`) processes every key the request sends, not just the keys your handler needs. This increases attack surface and is flagged. **Fix:** access only the specific named keys your handler requires: `$field = sanitize_text_field( wp_unslash( $_POST['field_name'] ?? '' ) )`.

- **`esc_html()` applied to HTML content** — `esc_html()` escapes `<`, `>`, `&`, `"`, and `'` — it strips all HTML tags from the output. Using it on content that is meant to contain tags (post content, user-authored HTML, rich-text output) is both a display bug and a PCP flag. **Fix:** use `wp_kses_post( $content )` to allow the standard post-content tag allowlist, or `wp_kses( $content, $allowed )` with a custom allowlist for more restricted contexts.

- **Plugin activates or deactivates other plugins programmatically** — Calling `activate_plugin()`, `deactivate_plugins()`, or any equivalent outside of a user-initiated action or a dependency-failure guard is prohibited. It removes user control over their site. **Fix:** remove all such calls. If your plugin requires another plugin, use the WordPress 6.5+ Plugin Dependencies header (`Requires Plugins:`) to declare the dependency and let WordPress handle the UI.

- **Built-in custom update checker ("phones home")** — Including code that contacts your own server, GitHub, Bitbucket, or any external host to check for plugin updates is prohibited. WordPress.org provides the update service for directory-hosted plugins; external update checkers create a parallel channel that bypasses the directory. **Fix:** remove all custom update-check code (including common libraries like `plugin-update-checker` by YahnisElsts). If you need to notify users of a major update, use the standard WordPress plugin details page.

- **Slug naming restrictions** — The plugin slug (permanently fixed at approval) must satisfy all of: (a) only English letters and Arabic numerals — no accented characters, no Unicode; (b) cannot start with `wordpress` or `plugin` except in extreme circumstances; (c) cannot contain version numbers; (d) cannot contain vulgarities or slurs; (e) cannot start with a trademarked term you do not own; (f) cannot imply official status ("Official WooCommerce Connector" by a non-Automattic developer). Display name can be changed post-approval; slug cannot. Choose carefully before submitting.

- **Using "trunk" as the Stable tag** — Setting `Stable tag: trunk` in `readme.txt` means WordPress.org always serves the current trunk code to users as the stable release, making rollback impossible and complicating hotfixes. **Fix:** always set `Stable tag:` to an explicit version number matching a tag in your SVN `tags/` directory (e.g. `Stable tag: 1.2.3`). Never use `trunk`.

- **SVN tag naming must be numbers and periods only** — SVN tags must match the pattern `X.Y.Z` (numbers and periods). Tags like `my-release`, `v1.0`, or `release-2024` are not valid and break the zip generator. Use Semantic Versioning: `1.0.0`, `2.3.1`.

- **No compressed archives inside the plugin zip** — Including `.zip`, `.tar.gz`, `.gz`, or other compressed files inside the plugin package is prohibited. Remove all such files. If you are bundling a library that ships as a zip, extract it first.

- **Plugin zip must be under 10 MB** — The WordPress.org submission form rejects files over 10 MB. Before submitting, remove: test suites (`tests/`), documentation build output (`docs/`), `node_modules/`, `bower_components/`, full `vendor/` trees (include only what the plugin actually loads), large demo assets, and anything excluded from distribution anyway. Run `du -sh plugin.zip` before submitting.

- **Non-standard file types** — WordPress.org reviewers flag unusual file types in the plugin package. Permitted without question: `.php`, `.js`, `.css`, `.txt`, `.md`, `.png`, `.svg`, `.jpg`, `.jpeg`, `.gif`, `.webp`, `.json`, `.xml`, `.po`, `.mo`, `.pot`. Any other extension (`.sh`, `.rb`, `.py`, `.bin`, `.exe`, `.dmg`) requires a clear documented reason in your review reply, and executable non-PHP scripts (`.sh`, `.rb`) are almost always rejected.

- **`composer.json` must be present if plugin uses Composer** — If the plugin uses Composer to manage PHP dependencies, `composer.json` must be included in the plugin root. Reviewers and future developers need it to understand the dependency tree and to fork/audit the code. Even if Composer is only used during development (build step), the `composer.json` must ship. The `vendor/` directory should include only what the plugin actually loads (prune dev dependencies).

- **Changelog retention in readme.txt** — Keep the current version and one major version back in the `== Changelog ==` section of `readme.txt`. Move older versions to a `changelog.txt` file in the plugin root. `changelog.txt` is not shown in the Plugin Directory but remains accessible via SVN. Over-long changelogs bloat the readme and are flagged by reviewers.

- **Main plugin file must be named to match the slug** — The primary PHP file must be named `plugin-slug.php`, matching the plugin's WordPress.org slug (derived from `Plugin Name:` in the header). A mismatch (e.g. main file named `index.php` or `plugin.php`) violates WordPress convention and can break update detection. Every plugin has exactly one file with the full plugin header block; that file must carry the slug name.

- **Framework and library-only plugins not accepted** — A plugin that is purely a code library, utility framework, or "starter template" that other plugins are expected to require, copy from, or extend directly is not accepted. The Plugin Directory is for end-user plugins, not shared code bases. Bundle your shared utilities within each consumer plugin instead. If your plugin requires another plugin to function, that other plugin must itself be a fully functional end-user plugin (not a pure library).

- **100% duplicate plugin or pure core reimplementation** — Submitting a plugin that is an unmodified or minimally modified copy of an existing plugin, or that only replicates functionality already built into WordPress core (e.g. a plugin that adds a `the_content` filter that core already applies), is rejected. Your plugin must provide genuine new value. Forks of GPL plugins are legally permitted but must provide clear improvement; credit the original author per GPL requirements.

- **Unprotected WP-Cron callback (PHP-FPM crash loop)** — a cron callback registered directly via `add_action( 'hook', $callback )` with no `Throwable`-catching wrapper will kill the PHP-FPM worker if `$callback` throws. PHP 8 turned many previously-silent conditions (e.g. `fread($handle, 0)`, type coercions) into `ValueError`/`TypeError`. The crash flow: uncaught exception → worker SIGSEGV → PHP-FPM spawns a replacement → next page load triggers WP-Cron again → same exception → crash loop → all workers dead → 503. No log entry is written because the worker dies before any logger can flush. **Fix:** never register a cron hook directly. Always use a wrapper that catches `Throwable` and calls `error_log()`. For class-based plugins add a `private static function cron_action( string $hook, callable $callback ): void` helper; for procedural plugins add a prefixed standalone function with the same signature. Pattern: `add_action( $hook, static function () use ( $hook, $callback ): void { try { $callback(); } catch ( \Throwable $e ) { error_log( sprintf( '[plugin] cron "%s" exception (%s): %s in %s line %d', $hook, get_class($e), $e->getMessage(), $e->getFile(), $e->getLine() ) ); } } );`. Every cron registration in the plugin must go through this wrapper — audit with `grep -n "add_action.*cron\|wp_schedule_event" *.php includes/*.php` and verify each hook has a wrapper. **Critical structural requirement:** `try {` must be the **first executable statement** inside the callback body — any variable declaration, option fetch, or function call placed *before* `try` is unprotected and can crash the PHP-FPM worker just as surely as code inside an unwrapped callback. Move all initialisation inside the `try` block. A common failure mode is an inner `try/catch` that covers only part of the callback while setup code runs before it; the outer wrapper must enclose the entire body from the first line. **Audit rule:** after confirming each hook has a wrapper, open the callback and check that the first non-comment, non-blank line is `try {`. Logs appear in PHP error log / `docker logs` container output.

---

## Block Plugin additional requirements

Plugins submitted to the **Block Directory** (the subset of the Plugin Directory for editor blocks) must satisfy all general plugin guidelines **plus** these stricter requirements. Violations result in rejection from the Block Directory; the plugin may still qualify for the main Plugin Directory.

### Structural requirements

- Must include a valid `block.json` file with **all** of: `name`, `title`, and at least one of `script`/`editorScript` and at least one of `style`/`editorStyle`.
- Must register at least one new block. Style-only extensions (block variations, block styles without a new block) are not currently eligible for the Block Directory.
- Must be a standard WordPress plugin with a valid `readme.txt`.

### Scope and purpose requirements

- Block plugins are **single-purpose**: one top-level block per plugin in almost all cases. Multiple blocks are only permitted when they have a clear parent/child or container/content relationship (e.g. a list block plus a list-item block). Collections of unrelated blocks are rejected.
- **No UI outside the block editor**: no options pages, no `wp-admin` menus, no Dashboard widgets, no admin notices. The block plugin must operate entirely inside the post/site editor.
- Server-side code must be **minimal**. Prefer the WordPress REST API over custom PHP implementations. PHP is permitted only where genuinely necessary for performance, and must be clearly written and documented.

### Dependency and functionality requirements

- Must function **standalone** immediately upon installation — no additional setup steps, no account creation, no activation key, no manual service connections required.
- Must not require another plugin or theme to function.
- May use external APIs only where integration is seamless and does not require user authentication steps post-install.
- Must not rely on an external API for functionality that could reasonably be performed locally.

### Prohibited in block plugins (additional to general rules)

- Any advertisements, upsell prompts, or premium feature gates.
- Any notifications, dashboard alerts, or admin notices of any kind.
- Requiring payment for any functionality.
- Any UX surface outside the block editor.
- Trademark violations in block name or plugin name.

### Naming

- Block `name` in `block.json` must be unique and namespaced: `author-name/block-name` or `plugin-slug/block-name`. Cannot use reserved namespaces `core` or `wordpress`.
- Plugin title and block title must clearly describe their purpose. Generic titles like "Image Block" or "Button Block" that could be confused with core blocks are rejected.

### Audit checklist for block plugins

```bash
# Confirm block.json exists and has required fields
find . -name block.json | xargs grep -l '"name"' | xargs grep -l '"title"'

# Check for admin page registration (prohibited)
grep -rn "add_menu_page\|add_submenu_page\|add_options_page\|admin_notices" --include=*.php

# Check for dashboard widgets (prohibited)
grep -rn "wp_add_dashboard_widget\|add_meta_box" --include=*.php

# Check for upsell/premium gates
grep -rn "is_pro\|upgrade\|premium\|licence_key\|license_key" --include=*.php
```
