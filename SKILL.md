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
| Critical | Echoed `<script>` or `<style>` tags, missing nonce, unescaped output, raw SQL, WordPress.org ownership or contributor mismatch, hidden files (dot-files) present in the plugin directory |
| High | Missing capability check, unsanitised input, bare `die()`, hardcoded URLs, missing ABSPATH guard |
| Medium | Duplicate helper functions, missing DocBlocks, version string mismatch, missing CHANGELOG entry, global asset enqueue |
| Low | Naming convention violations, missing inline comments, non-autoloaded options, minor i18n issues, PHPCS false-positive suppressions missing for delegated-nonce handlers (`NonceVerification.Missing`) or WordPress core hook names (`NonPrefixedHooknameFound`) |

**Do not proceed to Step 1 until the user replies with confirmation.**

### Step 1 — Check Utils

Before writing any new function, read `includes/class-SLUG-utils.php` and confirm
the helper does not already exist there (see `references/reuse.md`).

### Step 2 — Apply fixes

Apply fixes for the severity levels the user confirmed. Use `references/security.md`,
`references/coding-standards.md`, `references/performance.md`, and
`references/accessibility.md` as you go.

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
- Text domain must match the plugin slug and appear on every translatable string
- All enqueued assets versioned with the plugin version constant
- WordPress.org `Contributors:` field must match the submitting username; email domain must relate to the plugin's declared URLs

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
- **`NonceVerification.Missing` false positives via helper delegation** — PHPCS cannot trace nonce verification through a shared helper such as `ajax_check()`. Every `$_POST` access below the helper call is flagged even though the nonce was checked. Fix: wrap the affected `$_POST` reads in a `// phpcs:disable` / `// phpcs:enable` block with an explanation comment naming the helper. Example:
  ```php
  $this->ajax_check(); // verifies nonce
  // phpcs:disable WordPress.Security.NonceVerification.Missing -- nonce checked via ajax_check()
  $post_id = (int) ( $_POST['post_id'] ?? 0 );
  // phpcs:enable WordPress.Security.NonceVerification.Missing
  ```
  Review every AJAX handler that calls a shared nonce-check helper rather than `check_ajax_referer()` directly. See `references/security.md` §Nonce verification for the full pattern.
- **`NonPrefixedHooknameFound` for WordPress core hooks** — when a plugin calls `apply_filters()` or `do_action()` using a WordPress core hook name (e.g. `the_content`, `https_local_ssl_verify`, `robots_txt`), PHPCS warns that the hook name does not start with the plugin prefix. This is a false positive — the plugin is *invoking* a core hook, not *registering* its own. Suppress inline: `// phpcs:ignore WordPress.NamingConventions.PrefixAllGlobals.NonPrefixedHooknameFound -- [hook-name] is a WordPress core filter`. See `references/pcp-checklist.md` §Code quality.
