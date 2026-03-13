# Plugin Check (PCP) Compliance Checklist

Run every item before finalising any plugin file. The [WordPress Plugin Check plugin](https://github.com/WordPress/plugin-check) runs these checks automatically on WordPress.org submission. Failures mean rejection.

## Plugin header

- [ ] `Plugin Name` present and unique
- [ ] `Version` matches `VERSION` constant and `readme.txt` `Stable tag`
- [ ] `Requires at least` present with a valid WP version
- [ ] `Tested up to` is the current latest stable WP release
- [ ] `Requires PHP` present, minimum 7.4 (8.0 recommended)
- [ ] `License` is `GPLv2 or later` or a compatible licence
- [ ] `License URI` is correct
- [ ] `Text Domain` matches the plugin directory slug exactly
- [ ] `Domain Path` present if `.pot` files are included
- [ ] No trailing whitespace or BOM in the main plugin file

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
- [ ] No closing `?>` at end of PHP-only files
- [ ] No short open tags `<?`
- [ ] No deprecated WordPress functions
- [ ] No direct DB table creation without first checking whether the table exists
- [ ] No hardcoded `http://` URLs — use `https://` or `plugin_dir_url()`
- [ ] No `set_time_limit()` calls — flagged as discouraged by PCP (`Squiz.PHP.DiscouragedFunctions`)
- [ ] All variables in `uninstall.php` global scope are prefixed with the plugin slug (e.g. `$cs_seo_meta_keys` not `$meta_keys`) — PCP flags `NonPrefixedVariableFound`
- [ ] All classes in global scope are prefixed with the plugin slug — PCP flags `NonPrefixedClassFound`; add `phpcs:ignore` with explanation if the name genuinely includes the prefix

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

- [ ] `readme.txt` present in plugin root
- [ ] `uninstall.php` present and removes all plugin data
- [ ] No `.git`, `.svn`, `node_modules`, or `vendor` in the deployed plugin
- [ ] No `package.json`, `composer.json`, `.env`, or build config files in the deployed root
- [ ] All PHP files in `includes/`, `admin/`, `public/` have the ABSPATH guard
- [ ] File names use lowercase with hyphens: `class-my-plugin-admin.php`

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
- [ ] `Plugin URI` and `Author URI` domains are ones you demonstrably own or represent
- [ ] If submitting on behalf of an organisation, your WordPress.org email is under the organisation's domain — or a DNS TXT verification record has been added to the owner's domain

**Ownership mismatch is an immediate rejection.** The reviewer checks that the submitting WordPress.org username appears in the `Contributors:` field and that the email domain relates to the plugin's declared URLs. If your account username differs from what is in `Contributors:`, or your email domain does not match the plugin's domain, the submission will be held until you resolve one of the following:

1. Add a DNS TXT record at the owner's domain root with the value the reviewer provides (e.g. `wordpressorg-USERNAME-verification`)
2. Update your WordPress.org profile email to one under the owner's domain
3. Ask the reviewer to transfer the submission to the correct WordPress.org account

## Pre-submission final check

1. Run Plugin Check plugin locally — confirm zero errors and zero warnings
2. Confirm `Version`, `VERSION` constant, and `Stable tag` all match
3. Confirm `CHANGELOG.md` has an entry for this version
4. Confirm `readme.txt` changelog section has an entry for this version
5. Confirm `Tested up to` is the current latest stable WordPress release
6. Confirm no development artefacts are present in the deployed plugin
