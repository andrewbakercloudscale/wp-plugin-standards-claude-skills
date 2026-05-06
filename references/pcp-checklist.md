# Plugin Check (PCP) Compliance Checklist

Run every item before finalising any plugin file. The [WordPress Plugin Check plugin](https://github.com/WordPress/plugin-check) runs these checks automatically on WordPress.org submission. Failures mean rejection.

## Plugin header

- [ ] `Plugin Name` present and unique
- [ ] `Version` matches `VERSION` constant and `readme.txt` `Stable tag`
- [ ] `Requires at least` present with a valid WP version
- [ ] `Tested up to` is ≥ the current WordPress stable release — WordPress.org automated scanning rejects submissions where this value is **lower** than the current stable (`outdated_tested_upto_header`). **Never downgrade this value during a review.** If the existing value looks like an unreleased version, verify against wordpress.org/news before changing it — it may simply be ahead of your knowledge cutoff. The safe rule: only raise this value, never lower it.
- [ ] `Requires PHP` present, minimum 7.4 (8.0 recommended)
- [ ] `License` is `GPLv2 or later` or a compatible licence
- [ ] `License URI` is correct
- [ ] `Text Domain` matches the WordPress.org plugin slug exactly. **Critical:** WordPress.org derives the slug from the plugin *name*, not the folder name. "CloudScale Free Backup and Restore" → slug `cloudscale-free-backup-and-restore`. PCP reports `textdomain_mismatch` and `WordPress.WP.I18n.TextDomainMismatch` on every translatable string if the header `Text Domain:` does not match. Verify: slugify the plugin name (lowercase, hyphens) and confirm it matches `Text Domain:` in the header.
- [ ] `Domain Path` present if `.pot` files are included
- [ ] No trailing whitespace or BOM in the main plugin file

## Cyber security

These checks go beyond PCP compliance. See `references/cyber-security.md` for
code patterns and explanations.

### Admin access control

- [ ] Every `add_menu_page()` / `add_submenu_page()` uses a capability that is **not** `'read'` — `'read'` is granted to Subscribers; use `'manage_options'` or more specific capabilities
- [ ] Every admin page render callback starts with `current_user_can()` — WordPress only enforces the menu capability when reached via the admin menu; direct URL access bypasses it
- [ ] Every file in `admin/partials/` has both an ABSPATH guard (`exit;`) and a `current_user_can()` check
- [ ] No admin-only AJAX action registered on `wp_ajax_nopriv_` — that hook fires for logged-out users
- [ ] All `admin_post_{action}` handlers call `check_admin_referer()` and `current_user_can()`
- [ ] No REST endpoint uses `'__return_true'` as `permission_callback`
- [ ] `is_admin()` is not used as an access-control check — it checks whether the admin area is loaded, not whether the user is an administrator; use `current_user_can()` instead

### Injection and deserialisation

- [ ] No `unserialize()` on user-supplied, cookie, or URL data — use `json_decode()`
- [ ] No `maybe_unserialize()` on externally-sourced or third-party option values
- [ ] No `exec()`, `system()`, `passthru()`, `shell_exec()`, `proc_open()`, or `popen()` with user-controlled input
- [ ] XML parsed with `LIBXML_NONET | LIBXML_NOENT` flags and `libxml_disable_entity_loader(true)` (PHP < 8)

### File upload

- [ ] File type validated server-side via `wp_check_filetype_and_ext()` — `$_FILES['type']` is client-supplied and not trusted
- [ ] Executable file extensions explicitly blocked: `.php`, `.php3`, `.php4`, `.php5`, `.phtml`, `.phar`, `.pl`, `.py`, `.sh`, `.cgi`
- [ ] Uploads handled via `wp_handle_upload()`, not manual `move_uploaded_file()`

### Redirects and SSRF

- [ ] Redirects to user-supplied URLs use `wp_safe_redirect()` not `wp_redirect()`
- [ ] Any cross-domain redirect validated against an explicit domain allowlist
- [ ] User-supplied URLs are not passed directly to `wp_remote_get()` / `wp_remote_post()` — validate via `wp_http_validate_url()` first
- [ ] `http_request_host_is_external` filter is not overridden to re-allow internal/private IP requests

### Path traversal

- [ ] No file paths built from user input without `validate_file()` returning `0` and a `realpath()` boundary check
- [ ] `sanitize_file_name()` applied to any filename sourced from user input

### Privilege escalation and IDOR

- [ ] No `role` or capability data accepted from `$_POST` / `$_GET` and applied to `wp_update_user()` without allowlist validation and `current_user_can( 'promote_users' )` check
- [ ] Object-level capability checks used: `current_user_can( 'edit_post', $post_id )` not just `current_user_can( 'edit_posts' )` when fetching posts by a user-supplied ID

### Cryptographic / credential hygiene

- [ ] No `md5()` or `sha1()` for password hashing — use `wp_hash_password()` / `wp_check_password()`
- [ ] No API keys, tokens, or passwords hardcoded in plugin files — use `wp-config.php` constants
- [ ] All external API calls use HTTPS

### Information disclosure

- [ ] No `phpinfo()` calls
- [ ] `$wpdb->last_error` not output to the browser
- [ ] No credentials, tokens, or internal paths in code comments or error messages exposed to users

## Security

- [ ] All superglobal values sanitised before use
- [ ] All DB queries use `$wpdb->prepare()`
- [ ] All output escaped with context-specific escaper
- [ ] All AJAX handlers call `check_ajax_referer()` or `wp_verify_nonce()`
- [ ] All REST endpoints have a non-trivial `permission_callback`
- [ ] All privileged actions gated with `current_user_can()`
- [ ] No `eval()`
- [ ] No `base64_decode()` on untrusted input
- [ ] No `$_REQUEST` where `$_POST` or `$_GET` can be used specifically

## Code quality

- [ ] No `error_log()` outside the Utils logger (gated on `WP_DEBUG_LOG`)
- [ ] No `var_dump()`, `print_r()`, `debug_print_backtrace()` in committed code
- [ ] No bare `die()` or `exit()` — use `wp_die()` in HTTP contexts
- [ ] Every `wp_die()` call passes an **escaped** string — `wp_die( esc_html__( 'Forbidden', 'text-domain' ) )` not `wp_die( 'Forbidden' )`. PCP flags `WordPress.Security.EscapeOutput.OutputNotEscaped` on unescaped `wp_die` arguments.
- [ ] No `date()` — use `gmdate()` for UTC timestamps or `wp_date()` for timezone-localised display. PCP flags `WordPress.DateTime.RestrictedFunctions.date_date` on every `date()` call.
- [ ] No `unlink()` — use `wp_delete_file()`. PCP flags `WordPress.WP.AlternativeFunctions.unlink_unlink`.
- [ ] No `rmdir()` — use WP Filesystem API (`$wp_filesystem->rmdir()`). PCP flags `WordPress.WP.AlternativeFunctions.file_system_operations_rmdir`.
- [ ] No `readfile()` — use WP Filesystem API. PCP flags `WordPress.WP.AlternativeFunctions.file_system_operations_readfile`.
- [ ] No direct cURL (`curl_init`, `curl_exec`, etc.) in plugin-authored code — **hard WordPress.org rejection**; `phpcs:ignore` suppresses PHPCS/PCP but human reviewers will still reject it. Replace with `wp_remote_get()` / `wp_remote_post()`. This applies to all own-code API integrations (Dropbox, Google Drive, AWS S3, OneDrive, OAuth flows, etc.) — there is no exception for "it's just an HTTP call". **Third-party vendor libraries** containing cURL are explicitly permitted by reviewers. Sole exception in own code: sub-second connect timeout requirement where `wp_remote_get()` genuinely cannot substitute (e.g. AWS IMDS); suppress with `// phpcs:ignore WordPress.WP.AlternativeFunctions.curl_curl_init, WordPress.WP.AlternativeFunctions.curl_curl_setopt_array, WordPress.WP.AlternativeFunctions.curl_curl_exec, WordPress.WP.AlternativeFunctions.curl_curl_getinfo, WordPress.WP.AlternativeFunctions.curl_curl_close -- wp_remote_get() does not support sub-second connect timeouts required for IMDS` but be aware human reviewers may still flag it.
- [ ] No `set_time_limit()` without a `phpcs:ignore` — PCP flags `Squiz.PHP.DiscouragedFunctions.Discouraged`. Suppress with `// phpcs:ignore Squiz.PHP.DiscouragedFunctions.Discouraged -- required to prevent PHP timeout on large backups`.
- [ ] All superglobal arrays (`$_POST`, `$_GET`, `$_FILES`, `$_SERVER`) run through `wp_unslash()` before any sanitisation function. PCP flags `WordPress.Security.ValidatedSanitizedInput.MissingUnslash` otherwise. Pattern: `sanitize_text_field( wp_unslash( $_POST['field'] ?? '' ) )`.
- [ ] **All** variables echoed into HTML — including intermediate variables that only hold safe values like `'checked'`, hex colour codes, or pre-escaped attribute strings — must be wrapped in the appropriate `esc_*()` function. Variables inside `onclick="..."` JavaScript attributes must use `esc_js()`. A variable already assigned via `esc_attr()` earlier still requires wrapping at the point of output — escape at output, not at assignment.
- [ ] `InputNotSanitized` — when array input is validated via `array_map('intval', ...)` or `array_intersect()` against a whitelist rather than a named `sanitize_*()` function, add `// phpcs:ignore WordPress.Security.ValidatedSanitizedInput.InputNotSanitized -- sanitised via [method]`.
- [ ] Schema introspection queries (`SHOW CREATE TABLE`, `DESCRIBE`) require `WordPress.DB.DirectDatabaseQuery.SchemaChange` in the phpcs:ignore list in addition to `DirectQuery` and `NoCaching`.
- [ ] Multi-line `$wpdb->prepare()` calls containing interpolated table names — use `phpcs:disable` / `phpcs:enable` blocks rather than `phpcs:ignore` so the suppression covers all lines of the statement. PCP's `WordPress.Security.EscapeOutput.OutputNotEscaped` fires on any unescaped variable regardless of its contents. Specifically audit: closures that return attribute strings (e.g. `$mc('key', true)` → `echo $mc(...)` must be `echo esc_attr( $mc(...) )`), inline style variables containing hex colours, AMI table row ID variables used in `id=` or `onclick=` attributes.
- [ ] Direct DB queries (`$wpdb->query()`, `$wpdb->get_results()`, etc.) suppress `WordPress.DB.DirectDatabaseQuery.DirectQuery` and `WordPress.DB.DirectDatabaseQuery.NoCaching`. For single-line queries use `// phpcs:ignore` on the **same line** as the query. For blocks of queries (e.g. a restore loop with `SET FOREIGN_KEY_CHECKS` + replay loop), use a `phpcs:disable` / `phpcs:enable` block — inline `phpcs:ignore` is not reliably respected by Plugin Check on `$wpdb->query()` calls inside loops. Always include `PluginCheck.Security.DirectDB.UnescapedDBParameter` in the suppression list when the query argument is a variable.
- [ ] Interpolated table names in `$wpdb->prepare()` — suppress `WordPress.DB.PreparedSQL.InterpolatedNotPrepared` with `// phpcs:ignore WordPress.DB.PreparedSQL.InterpolatedNotPrepared -- $safe_table is sanitised via esc_sql()`.
- [ ] `NonceVerification.Missing` — every AJAX handler that reads `$_POST`/`$_GET`/`$_FILES` must call `check_ajax_referer()`, `wp_verify_nonce()`, or `check_admin_referer()` **directly in the same function/closure body**. A shared helper wrapper (e.g. `ajax_check()`, `cs_verify_nonce()`) is not traced by PHPCS or Plugin Check, and `phpcs:disable NonceVerification.Missing` does not satisfy the check either. Grep for `add_action( 'wp_ajax_` and confirm a direct call appears in each handler — not via a delegated helper. If a helper wrapper exists, replace every AJAX call site with the direct `check_ajax_referer()` call.
- [ ] No `set_time_limit()` calls without suppression — add `// phpcs:ignore Squiz.PHP.DiscouragedFunctions.Discouraged -- required to prevent PHP timeout on large backups` on every instance.
- [ ] **🚨 NO `exec()`, `shell_exec()`, `system()`, `passthru()`, `proc_open()`, or `popen()` calls** — WordPress.org reviewers require **complete removal**. `phpcs:ignore` and `escapeshellarg()` do **not** satisfy a human review; the plugin will be rejected regardless. PCP automated checks flag these as discouraged, but reviewers go further and block submission. Rewrite using PHP/WP equivalents: `ZipArchive` (zip), `$wpdb->query()` (DB), WP Filesystem API (files), `wp_remote_*` (HTTP).
- [ ] `file_put_contents()` / `file_get_contents()` — add `// phpcs:ignore WordPress.WP.AlternativeFunctions.file_system_operations_file_put_contents, WordPress.WP.AlternativeFunctions.file_system_operations_file_get_contents -- [reason: WP Filesystem API unavailable in this context]` on each.
- [ ] No `fopen()` on remote URLs — use `wp_remote_get()` / `wp_remote_post()`. Many hosts block PHP stream wrappers for remote access. `fopen()` for local filesystem operations is also discouraged; use WP Filesystem API. Exception: third-party vendor libraries you cannot modify (document in review reply).
- [ ] `mt_rand()` — replace with `wp_rand()`. PCP flags `WordPress.WP.AlternativeFunctions.rand_mt_rand` as an error.
- [ ] No closing `?>` at end of PHP-only files
- [ ] No short open tags `<?`
- [ ] No deprecated WordPress functions
- [ ] No direct DB table creation without first checking whether the table exists
- [ ] No hardcoded `http://` URLs — use `https://` or `plugin_dir_url()`
- [ ] No `set_time_limit()` calls — flagged as discouraged by PCP (`Squiz.PHP.DiscouragedFunctions`)
- [ ] All variables in `uninstall.php` global scope are prefixed with the plugin slug (e.g. `$cs_seo_meta_keys` not `$meta_keys`) — PCP flags `NonPrefixedVariableFound`
- [ ] All classes in global scope are prefixed with the plugin slug — PCP flags `NonPrefixedClassFound`; add `phpcs:ignore` with explanation if the name genuinely includes the prefix
- [ ] All hook names *registered* by the plugin (`add_filter`, `add_action`, `do_action`, `apply_filters` where the plugin owns the hook) start with the plugin slug prefix — PCP flags `NonPrefixedHooknameFound`
- [ ] **🚨 CRITICAL — Prefix is at least 4 characters long** — WordPress.org will **reject the submission outright** if any prefix is 2 or 3 characters (e.g. `cs_`, `my_`, `ab_`). Count only characters before the first underscore (`cs_` = 2 characters, not 3). Applies to every globally scoped name: function names, class names, `define()` constants, `update_option()` / `set_transient()` / `update_post_meta()` keys, `add_shortcode()` tags, `register_setting()` keys, AJAX action names (`wp_ajax_{action}`), and `wp_register_script()` / `wp_register_style()` handles. Prefixes `wp_`, `_`, and `__` are reserved for WordPress core and are also prohibited. Derive the prefix from the plugin name (e.g. `csbr_` for "CloudScale Backup & Restore").
- [ ] WordPress core hook names *invoked* via `apply_filters()` or `do_action()` are suppressed with an inline `// phpcs:ignore WordPress.NamingConventions.PrefixAllGlobals.NonPrefixedHooknameFound -- [hook-name] is a WordPress core filter/action` comment. Common examples: `the_content`, `robots_txt`, `https_local_ssl_verify`, `script_loader_tag`, `style_loader_tag`. This is a false positive — the plugin is calling a core hook, not declaring its own.

## Code reuse

- [ ] No helper function duplicated across files — all shared helpers in Utils class
- [ ] Utils class exists at `includes/class-SLUG-utils.php`
- [ ] No copy-pasted blocks of logic — DRY throughout

## Documentation

- [ ] Every file has a file-level DocBlock with `@package` and `@since`
- [ ] Every class has a DocBlock with `@since`
- [ ] Every method and function has a DocBlock with `@since`, `@param`, `@return`
- [ ] Inline comments explain the *why*, not the *what*

## Internationalisation

- [ ] Every user-facing string wrapped in an i18n function
- [ ] Correct text domain on every i18n call — plain string literal, never a variable
- [ ] Do **not** call `load_plugin_textdomain()` — it has been discouraged since WP 4.6; WordPress.org auto-loads translations. Remove any existing call and its hook registration. PCP flags `PluginCheck.CodeAnalysis.DiscouragedFunctions.load_plugin_textdomainFound`.
- [ ] Every `printf()` / `sprintf()` call whose format string is wrapped in an i18n function **must** have a `/* translators: %s: description */` comment on the line immediately above. PCP flags `WordPress.WP.I18n.MissingTranslatorsComment` without it.

## Assets

This section is the most common source of WordPress.org submission rejections. Grep the entire codebase for `<script` and `<style` before submitting. Every hit is a violation.

- [ ] **Zero** echoed `<script>` tags anywhere in the codebase — use `wp_enqueue_script()` or `wp_add_inline_script()`
- [ ] **Zero** echoed `<style>` tags anywhere in the codebase — use `wp_enqueue_style()` or `wp_add_inline_style()`
- [ ] **Zero** direct `<script src=...>` tags — always use `wp_enqueue_script()`
- [ ] No scripts or styles enqueued via `wp_head` / `wp_footer` actions directly
- [ ] No scripts or styles enqueued via `admin_print_scripts` / `admin_print_styles` directly — use `admin_enqueue_scripts`
- [ ] Every enqueued script and style includes a version string (the plugin version constant)
- [ ] PHP data passed to JS via `wp_localize_script()`, not echoed JSON
- [ ] jQuery-dependent scripts declare `array( 'jquery' )` as a dependency
- [ ] Scripts loaded in footer (`true` or `array( 'in_footer' => true )`) unless there is a documented reason for the head
- [ ] Assets only enqueued on the specific screen where they are needed — gated by `$hook` in admin, by conditional tags on frontend
- [ ] `defer` / `async` applied via the `strategy` argument in `wp_enqueue_script()` (WP 6.3+), not via raw attribute injection

## File and folder structure

- [ ] `readme.txt` — `Tags:` line has **5 tags maximum** — PCP flags `readme_parser_warnings_too_many_tags` if exceeded
- [ ] `readme.txt` — short description (the single line below the header block) is **150 characters maximum** — PCP flags `readme_parser_warnings_trimmed_short_description` if longer
- [ ] **🚨 CRITICAL — `readme.txt` `== Changelog ==` section is 5,000 words maximum** — WordPress.org truncates it on import and warns _"The Changelog section is too long and was truncated."_ Trim or summarise older entries; keep only the most recent releases in full detail
- [ ] `readme.txt` present in plugin root
- [ ] `uninstall.php` present and removes all plugin data
- [ ] No `.git`, `.svn`, `node_modules`, or `vendor` in the deployed plugin
- [ ] No `package.json`, `composer.json`, `.env`, or build config files in the deployed root
- [ ] **No unexpected markdown files in the plugin root** — only `README.md` and `CHANGELOG.md` are permitted at the root level. Dev, planning, and audit files (`UX-AUDIT.md`, `TODO.md`, `NOTES.md`, `DECISIONS.md`, etc.) must be deleted or excluded from the distribution zip. PCP flags `unexpected_markdown_file` (warning) for each unexpected `.md` found. Verify with `unzip -l plugin.zip | grep '\.md$'`.
- [ ] All PHP files in `includes/`, `admin/`, `public/` have the ABSPATH guard
- [ ] File names use lowercase with hyphens: `class-my-plugin-admin.php`
- [ ] **No files written to the plugin directory** — never write to any path under `plugin_dir_path()` or `WP_PLUGIN_DIR`. Plugin directories are deleted on upgrade; files stored there are lost and publicly accessible. Use `wp_upload_dir()['basedir'] . '/plugin-slug/'` for persistent data storage. Applies to: `copy()`, `file_put_contents()`, `fwrite()`, and any WP Filesystem write whose destination resolves to the plugin folder.
- [ ] **No raw `WP_CONTENT_DIR` for writable storage** — using `WP_CONTENT_DIR . '/plugin-slug/'` as a storage root is rejected. All writable directories (backup dirs, staging dirs, temp/chunk dirs) must use `wp_upload_dir()['basedir'] . '/plugin-slug/'`. Call `wp_upload_dir()` at runtime inside a function — never at file-load time.
- [ ] **Own plugin file paths use `plugin_dir_path()`/`plugin_dir_url()`** — referencing your own plugin's files via `WP_PLUGIN_DIR . '/' . $your_slug . '/'` is flagged. Define constants at plugin boot: `define( 'MYPLUGIN_DIR', plugin_dir_path( __FILE__ ) )`. Use `WP_LANG_DIR` instead of `WP_CONTENT_DIR . '/languages'` and `WPMU_PLUGIN_DIR` directly instead of a `defined()` conditional falling back to `WP_CONTENT_DIR . '/mu-plugins'`.
- [ ] **No remote asset offloading** — JS, CSS, images, and other assets that are not part of WordPress core must be bundled locally in the plugin and served via `wp_enqueue_script()` / `wp_enqueue_style()`. Loading assets from your own domain, S3 bucket, or CDN is prohibited. Download links to your own hosted zip inside shipped help HTML files also trigger this violation. Exclude all `docs/`, `help-page.html`, and similar dev/marketing files from the distribution zip.

## Performance

- [ ] No DB queries on every page load without caching
- [ ] No queries inside loops
- [ ] Transients used for expensive or external data
- [ ] Non-autoloaded options used for data not needed on every page load

## Accessibility

- [ ] All form inputs have associated `<label>` elements
- [ ] No `tabindex` values greater than 0
- [ ] Interactive elements are keyboard operable
- [ ] Colour contrast meets WCAG 2.1 AA minimum
- [ ] Images have `alt` attributes
- [ ] Admin notices include screen-reader-only context prefix

## Licensing

- [ ] All bundled third-party libraries are GPL-compatible
- [ ] Bundled library licences documented in `readme.txt` under `== Credits ==`

## WordPress.org submission requirements

These are administrative checks that the reviewer performs manually and that automated tools do not catch.

- [ ] `Contributors:` field in `readme.txt` lists your exact WordPress.org username
- [ ] Your WordPress.org account email matches or is clearly related to the `Author URI` domain
- [ ] `Author:` in the plugin header matches the name on your WordPress.org profile
- [ ] **🚨 `Author URI` must not be a placeholder domain** — WordPress.org automated scanning hard-rejects any plugin where `Author URI:` contains `example.com`, `example.org`, `example.net`, or any other RFC 2606 reserved domain. Error: `plugin_header_invalid_author_uri_domain`. Replace with your real website URL before uploading.
- [ ] `Plugin URI` and `Author URI` domains are ones you demonstrably own or represent
- [ ] If submitting on behalf of an organisation, your WordPress.org email is under the organisation's domain — or a DNS TXT verification record has been added to the owner's domain

**Ownership mismatch is an immediate rejection.** The reviewer checks that the submitting WordPress.org username appears in the `Contributors:` field and that the email domain relates to the plugin's declared URLs. If your account username differs from what is in `Contributors:`, or your email domain does not match the plugin's domain, the submission will be held until you resolve one of the following:

1. Add a DNS TXT record at the owner's domain root with the value the reviewer provides (e.g. `wordpressorg-USERNAME-verification`)
2. Update your WordPress.org profile email to one under the owner's domain
3. Ask the reviewer to transfer the submission to the correct WordPress.org account

## onclick refactor checklist

Removing inline `onclick` attributes from PHP-rendered buttons is the correct PCP fix, but it must be completed end-to-end or the button silently breaks.

**For every `onclick` attribute removed from a PHP-rendered `<button>` or `<a>`:**

- [ ] The element has a unique `id` attribute added at the same time the `onclick` is removed
- [ ] A corresponding `addEventListener('click', fn)` call is wired to that `id` in the JS setup block (e.g. inside `DOMContentLoaded`)
- [ ] **No** `querySelector('[onclick="fnName()"]')` or `querySelector('[onclick*="fnName"]')` remains in the codebase — this selector returns `null` the moment the `onclick` attribute is gone and is therefore self-defeating
- [ ] The new event binding is tested by clicking the button after deployment

**Pattern — correct replacement:**

```php
<!-- PHP: was onclick="myFunction()" — add an id instead -->
<button type="button" id="my-action-btn" class="button">Do thing</button>
```

```javascript
// JS: bind by id inside DOMContentLoaded
document.addEventListener('DOMContentLoaded', function() {
    var btn = document.getElementById('my-action-btn');
    if (btn) btn.addEventListener('click', function() { myFunction(); });
});
```

**Anti-pattern — do NOT do this:**

```javascript
// BROKEN: querySelector finds nothing once onclick is removed
var btn = document.querySelector('[onclick="myFunction()"]');
if (btn) {
    btn.removeAttribute('onclick');          // onclick is already gone
    btn.addEventListener('click', myFunction); // never reached
}
```

**Grep to catch leftovers before submitting:**

```bash
# Find any remaining onclick-as-selector patterns in JS
grep -r "querySelector.*\[onclick" includes/
# Find PHP-rendered buttons with no id (review manually)
grep -n "<button" includes/*.php | grep -v ' id='
```

Note: `onclick` attributes inside JavaScript **string literals** (e.g. `innerHTML` assignments, template literals building table rows) are not a PCP violation — they are JS code, not PHP-rendered HTML attributes. Only `onclick` attributes written directly in PHP `echo`/heredoc output require removal.

## Pre-submission final check

1. Run Plugin Check plugin locally — confirm zero errors and zero warnings
2. Confirm `Version`, `VERSION` constant, and `Stable tag` all match
3. Confirm `CHANGELOG.md` has an entry for this version
4. Confirm `readme.txt` changelog section has an entry for this version
5. Confirm `Tested up to` is the current latest stable WordPress release
6. Confirm no development artefacts are present in the deployed plugin
7. **Confirm no hidden files in the zip** — run `unzip -l plugin.zip | grep '/\.'` and verify zero results. WordPress.org automated scanning rejects any plugin zip containing dot-files (`.distignore`, `.gitignore`, `.env`, `.DS_Store`, etc.) with the error `hidden_files: Hidden files are not permitted.` Ensure your build script excludes all dot-files from the distribution zip.
8. **Confirm `Author URI` is not a placeholder** — run `grep -i "Author URI" plugin-slug.php` and verify it points to a real URL you own. `example.com`, `example.org`, or `example.net` cause an automated hard-reject (`plugin_header_invalid_author_uri_domain`) before the submission reaches human review.
