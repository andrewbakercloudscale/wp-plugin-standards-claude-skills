# Cyber Security

WordPress-specific attack surface, OWASP Top 10 mapped to WordPress, and checks
that go beyond PCP compliance. Read this alongside `references/security.md` for
every plugin that handles user input, performs privileged actions, or exposes
admin screens.

---

## Admin screen access control

This is the most common access control failure in WordPress plugins: admin pages
that are reachable without authentication, or reachable by low-privilege users.

### Admin menu capability

Every `add_menu_page()` and `add_submenu_page()` call must pass a capability that
is **not** granted to Subscribers or Customers.

```php
// CORRECT — only administrators can reach this page
add_menu_page(
    __( 'My Plugin', 'plugin-slug' ),
    __( 'My Plugin', 'plugin-slug' ),
    'manage_options',          // <-- minimum required capability
    'my-plugin',
    array( $this, 'render_page' )
);

// WRONG — 'read' is granted to Subscribers; anyone logged-in can visit this page
add_menu_page( ..., 'read', ... );
```

Minimum safe capabilities by role:

| Capability | Minimum role |
|-----------|--------------|
| `manage_options` | Administrator |
| `manage_woocommerce` | Shop Manager (WooCommerce) |
| `edit_posts` | Author |
| `publish_posts` | Author |
| `read` | Subscriber — **never use for admin pages** |

### Admin page render callback

The render callback registered with `add_menu_page()` is called by WordPress only
after the capability check passes. But if your plugin triggers the render callback
any other way (e.g. via a direct `include`), it must re-check the capability:

```php
public function render_page(): void {
    if ( ! current_user_can( 'manage_options' ) ) {
        wp_die( esc_html__( 'You do not have permission to view this page.', 'plugin-slug' ) );
    }
    // render...
}
```

### Admin partials / templates

Template files in `admin/partials/` are PHP files. Without an ABSPATH guard they
can be loaded directly via a browser URL:

```
https://example.com/wp-content/plugins/my-plugin/admin/partials/my-plugin-admin-display.php
```

Every file in `admin/partials/` **must** have an ABSPATH guard as its first
executable line AND a capability check:

```php
<?php
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}
if ( ! current_user_can( 'manage_options' ) ) {
    wp_die( esc_html__( 'Forbidden.', 'plugin-slug' ) );
}
```

### AJAX handlers — admin vs. public split

Use the correct hook pair. Never register an admin-only action on `wp_ajax_nopriv_`:

```php
// Admin-only AJAX (logged-in users)
add_action( 'wp_ajax_my_plugin_action', array( $this, 'handle_action' ) );

// Public AJAX (also logged-out users) — only add this when genuinely needed
// add_action( 'wp_ajax_nopriv_my_plugin_action', array( $this, 'handle_action' ) );
```

Always verify nonce AND capability inside every `wp_ajax_*` handler:

```php
public function handle_action(): void {
    check_ajax_referer( 'my_plugin_nonce', 'nonce' );
    if ( ! current_user_can( 'manage_options' ) ) {
        wp_send_json_error( array( 'message' => 'Forbidden' ), 403 );
    }
    // process...
}
```

### admin-post.php handlers

Handlers hooked to `admin_post_{action}` must verify nonce and capability:

```php
add_action( 'admin_post_my_plugin_export', array( $this, 'handle_export' ) );

public function handle_export(): void {
    check_admin_referer( 'my_plugin_export_nonce' );
    if ( ! current_user_can( 'manage_options' ) ) {
        wp_die( esc_html__( 'Forbidden.', 'plugin-slug' ) );
    }
    // process...
}
```

### REST API endpoints

Never use `'__return_true'` as a `permission_callback` in production — it makes
the endpoint publicly accessible with no authentication:

```php
// WRONG — world-readable endpoint
register_rest_route( 'plugin-slug/v1', '/data', array(
    'permission_callback' => '__return_true',
) );

// CORRECT — require authentication
register_rest_route( 'plugin-slug/v1', '/data', array(
    'permission_callback' => function() {
        return current_user_can( 'manage_options' );
    },
) );
```

---

## OWASP Top 10 — WordPress mapping

### A01 — Broken Access Control

- Every admin page, AJAX handler, REST endpoint, and `admin-post.php` handler must
  verify the caller has the required capability before processing anything.
- Never trust client-supplied role or capability data (`$_POST['role']` etc.).
- Check the specific object capability, not just the generic one:
  `current_user_can( 'edit_post', $post_id )` not just `current_user_can( 'edit_posts' )`.
- IDOR (Insecure Direct Object Reference): when fetching posts, users, or options
  by a user-supplied ID, verify access to that specific object:

```php
// WRONG — verifies generic capability but not access to this specific post
if ( ! current_user_can( 'edit_posts' ) ) { wp_die(...); }
$post = get_post( absint( $_GET['post_id'] ) );

// CORRECT — verifies access to the specific post
$post_id = absint( $_GET['post_id'] ?? 0 );
if ( ! current_user_can( 'edit_post', $post_id ) ) {
    wp_die( esc_html__( 'You cannot edit this post.', 'plugin-slug' ) );
}
$post = get_post( $post_id );
```

### A02 — Cryptographic Failures

- Never use `md5()` or `sha1()` to hash passwords — use `wp_hash_password()` /
  `wp_check_password()`.
- API keys and secrets must not be stored in plain text in `wp_options`. Store in
  `wp-config.php` constants or environment variables and read via `defined()`.
- Never hardcode credentials, tokens, or API keys in plugin files.
- All external API calls must use HTTPS (`https://` scheme enforced in `esc_url_raw()`
  calls and `wp_remote_get()` args).

```php
// Storing a credential — use a constant, not wp_options
if ( ! defined( 'MY_PLUGIN_API_KEY' ) ) {
    // Remind user to set this in wp-config.php
    return new WP_Error( 'missing_key', 'API key not configured.' );
}
$key = MY_PLUGIN_API_KEY;
```

### A03 — Injection

**SQL injection** — all dynamic queries use `$wpdb->prepare()`. See `security.md`.

**PHP object injection** — never call `unserialize()` on user-supplied data or any
value read from an HTTP request, cookie, or URL parameter. Use `json_decode()`:

```php
// WRONG — arbitrary object injection if $data is user-controlled
$obj = unserialize( base64_decode( $_POST['data'] ) );

// CORRECT
$obj = json_decode( wp_unslash( $_POST['data'] ), true );
if ( ! is_array( $obj ) ) {
    wp_send_json_error( 'Invalid data.' );
}
```

**Shell injection** — never pass user input to shell execution functions:

```php
// These functions must NEVER receive user-controlled input
exec(), system(), passthru(), shell_exec(), proc_open(), popen()
```

If shell execution is unavoidable, whitelist every argument:

```php
$allowed_commands = array( 'ls', 'du' );
if ( ! in_array( $command, $allowed_commands, true ) ) {
    wp_die( esc_html__( 'Invalid command.', 'plugin-slug' ) );
}
```

**XSS (stored)** — escape every database value on output, even values your plugin
stored itself. Stored values can be poisoned by direct DB write or plugin conflict.

### A04 — Insecure Design / File Upload

- Never trust `$_FILES['type']` — it is client-supplied and trivially spoofed.
- Validate MIME type server-side using `wp_check_filetype_and_ext()`:

```php
$file     = $_FILES['upload'];
$filetype = wp_check_filetype_and_ext( $file['tmp_name'], $file['name'] );
if ( ! $filetype['type'] || ! $filetype['ext'] ) {
    wp_die( esc_html__( 'Invalid file type.', 'plugin-slug' ) );
}
// Block executable extensions
$blocked = array( 'php', 'php3', 'php4', 'php5', 'phtml', 'phar', 'pl', 'py', 'sh', 'cgi' );
if ( in_array( strtolower( $filetype['ext'] ), $blocked, true ) ) {
    wp_die( esc_html__( 'This file type is not permitted.', 'plugin-slug' ) );
}
// Use WordPress upload handler
$upload = wp_handle_upload( $file, array( 'test_form' => false ) );
```

- Use `wp_handle_upload()` for all file uploads — it enforces allowed types and
  stores files in the uploads directory.
- Do not allow uploaded files to be stored in a directory that permits PHP execution.

### A05 — Security Misconfiguration

- No `phpinfo()` calls anywhere in the plugin.
- No stack traces or verbose error messages exposed to the browser — use
  `WP_DEBUG_LOG` with `WP_DEBUG_DISPLAY` off.
- Remove all development files (`.env`, `phpunit.xml`, `composer.json`, test
  fixtures) from the deployed plugin directory.
- ABSPATH guard on every PHP file that is not the main plugin file.
- No sensitive data in code comments (API keys, passwords, tokens, internal URLs).

```php
// WRONG — logs to browser in any environment
error_log( 'Debug: ' . print_r( $data, true ) );

// CORRECT — only logs to file when WP_DEBUG_LOG is on
if ( defined( 'WP_DEBUG_LOG' ) && WP_DEBUG_LOG ) {
    // phpcs:ignore WordPress.PHP.DevelopmentFunctions.error_log_error_log
    error_log( 'my-plugin: ' . wp_json_encode( $data ) );
}
```

### A06 — Vulnerable and Outdated Components

- All bundled third-party libraries must be GPL-compatible (WordPress.org requirement)
  and must not have known unpatched CVEs.
- Use WordPress HTTP API (`wp_remote_get()`, `wp_remote_post()`) instead of bundling
  a custom HTTP client — WordPress handles TLS verification, redirects, and timeouts.
- Keep bundled libraries as thin as possible — each added dependency is an attack surface.
- Document all bundled libraries in `readme.txt` under `== Credits ==`.

### A07 — Identification and Authentication Failures

- Never call `wp_set_auth_cookie()` without first validating credentials through the
  WordPress authentication stack (`wp_authenticate()` or `wp_signon()`).
- Never bypass the `authenticate` filter chain.
- WordPress nonces expire (default 12 hours). Do not persist nonces in the DB or
  extend their lifetime to work around expiry — regenerate them instead.
- For sensitive admin operations (data deletion, export, privilege changes), require
  re-confirmation in the same request (nonce + capability check), not just a URL
  containing a previously-generated token.
- Rate-limit login/authentication attempts if the plugin implements any login flow.

### A08 — Software and Data Integrity Failures

- Never call `unserialize()` on any externally-sourced value including: `$_COOKIE`,
  `$_GET`/`$_POST`, option values set by other plugins, or REST request bodies.
- If the plugin implements a custom update mechanism, verify package signatures or
  checksums before applying any update.
- `maybe_unserialize()` is safe only for values your own plugin serialised and
  stored — never use it on user-supplied or third-party data.

### A09 — Security Logging and Monitoring

- Log authentication failures, nonce failures, and capability denials to
  `WP_DEBUG_LOG` (gated on `defined('WP_DEBUG_LOG') && WP_DEBUG_LOG`).
- Do **not** log passwords, nonces, session tokens, or API keys.
- Log entries should include: timestamp, action attempted, user ID (not username),
  and outcome. Never log PII.

### A10 — Server-Side Request Forgery (SSRF)

Never pass a user-supplied URL directly to `wp_remote_get()` or `wp_remote_post()`:

```php
// WRONG — SSRF: attacker can scan internal network
$url = sanitize_url( $_POST['webhook_url'] );
wp_remote_post( $url, $args );

// CORRECT — validate against an allowlist or use wp_http_validate_url()
$url = esc_url_raw( wp_unslash( $_POST['webhook_url'] ?? '' ) );
if ( ! wp_http_validate_url( $url ) ) {
    wp_send_json_error( 'Invalid URL.' );
}
// Additionally block private/internal IPs
$parsed = wp_parse_url( $url );
if ( in_array( $parsed['host'], array( 'localhost', '127.0.0.1', '::1' ), true ) ) {
    wp_send_json_error( 'Internal URLs are not allowed.' );
}
wp_remote_post( $url, $args );
```

WordPress 5.9+ blocks requests to private IP ranges by default via the
`http_request_host_is_external` filter. Do **not** override this filter to
re-enable internal requests.

---

## WordPress-specific attack vectors

### Open redirect

Use `wp_safe_redirect()` instead of `wp_redirect()` when redirecting to a
user-supplied or untrusted URL. `wp_safe_redirect()` blocks redirects to external
domains by default:

```php
// WRONG — attacker supplies ?redirect=https://evil.com
$url = sanitize_url( $_GET['redirect'] );
wp_redirect( $url );

// CORRECT — blocks cross-domain redirects
$url = esc_url_raw( wp_unslash( $_GET['redirect'] ?? '' ) );
wp_safe_redirect( $url );
exit;
```

If the plugin legitimately needs to redirect to a known external domain, whitelist
the allowed domains explicitly rather than using `wp_redirect()` broadly:

```php
$allowed_hosts = array( 'example.com', 'api.example.com' );
$host = wp_parse_url( $url, PHP_URL_HOST );
if ( in_array( $host, $allowed_hosts, true ) ) {
    wp_redirect( $url );
    exit;
}
wp_safe_redirect( home_url() );
exit;
```

### Path traversal

Never build file paths from user input without strict validation:

```php
// WRONG — path traversal: ../../wp-config.php
$file = $_GET['template'];
include plugin_dir_path( __FILE__ ) . 'templates/' . $file;

// CORRECT — validate_file() returns 0 only for safe relative paths
$file     = sanitize_file_name( wp_unslash( $_GET['template'] ?? '' ) );
$base_dir = plugin_dir_path( __FILE__ ) . 'templates/';
$full_path = realpath( $base_dir . $file );

if ( validate_file( $file ) !== 0 || ! $full_path || strpos( $full_path, realpath( $base_dir ) ) !== 0 ) {
    wp_die( esc_html__( 'Invalid template.', 'plugin-slug' ) );
}
include $full_path;
```

`validate_file()` returns:
- `0` — safe
- `1` — `..` traversal detected
- `2` — colon in path (Windows drive specifier)
- `3` — Windows drive path

### Privilege escalation

Never accept role or capability data from the request:

```php
// WRONG — user supplies their own role
$role = sanitize_text_field( $_POST['role'] );
wp_update_user( array( 'ID' => $user_id, 'role' => $role ) );

// CORRECT — role is hard-coded or comes from a validated allowlist
$allowed_roles = array( 'editor', 'author' );
$role = sanitize_key( $_POST['role'] ?? '' );
if ( ! in_array( $role, $allowed_roles, true ) ) {
    wp_send_json_error( 'Invalid role.' );
}
// Only administrators may promote users
if ( ! current_user_can( 'promote_users' ) ) {
    wp_send_json_error( 'Forbidden.', 403 );
}
wp_update_user( array( 'ID' => absint( $user_id ), 'role' => $role ) );
```

### XML external entity (XXE) injection

If the plugin parses XML, disable external entity loading:

```php
// Before parsing any XML from an external or user-supplied source
if ( PHP_VERSION_ID < 80000 ) {
    // libxml_disable_entity_loader() deprecated in PHP 8 (safe by default)
    // phpcs:ignore PHPCompatibility.FunctionUse.RemovedFunctions.libxml_disable_entity_loaderDeprecated
    libxml_disable_entity_loader( true );
}
$xml = simplexml_load_string(
    wp_kses( $xml_string, array() ),
    'SimpleXMLElement',
    LIBXML_NONET | LIBXML_NOENT
);
```

### Information disclosure

- Do not reveal WordPress version, PHP version, or plugin version numbers in
  public-facing output or HTTP headers.
- Never output database errors to the browser — WordPress suppresses `$wpdb->last_error`
  by default; ensure your code does not re-display it.
- `is_admin()` checks whether the admin area is being loaded, not whether the
  current user is an administrator. Do not use `is_admin()` as an access control check.

```php
// WRONG — is_admin() is true for AJAX requests from the frontend too
if ( is_admin() ) {
    // This runs for any AJAX call, not just admins
}

// CORRECT — check capability
if ( current_user_can( 'manage_options' ) ) {
    // Only actual admins
}
```

### Race conditions

For operations that must be atomic (decrementing a counter, marking a record as
processed, allocating a unique ID), use DB-level atomicity — do not read then write
in separate queries:

```php
// WRONG — read/modify/write race condition
$count = (int) get_option( 'my_plugin_count', 0 );
update_option( 'my_plugin_count', $count + 1 );

// CORRECT — atomic increment at the DB level
global $wpdb;
$wpdb->query(
    $wpdb->prepare(
        // phpcs:ignore WordPress.DB.DirectDatabaseQuery.DirectQuery, WordPress.DB.DirectDatabaseQuery.NoCaching -- atomic increment requires direct query
        "UPDATE {$wpdb->options} SET option_value = option_value + 1 WHERE option_name = %s",
        'my_plugin_count'
    )
);
```

### WordPress XML-RPC

If the plugin does not use XML-RPC, do not enable or extend it. If the plugin
disables XML-RPC, do so only for non-authenticated contexts and document the reason.
Blanket XML-RPC disablement can break Jetpack and other legitimate integrations.

### nonce ≠ CSRF token for logged-out users

WordPress nonces are tied to the current user. For logged-out users, all nonces
share the same user context (user ID 0), making them weaker. For public-facing
forms that modify data, implement additional protections:

- Require user login before the form is submitted, **or**
- Use a short nonce lifetime (`add_filter( 'nonce_life', function() { return 30 * MINUTE_IN_SECONDS; } )`), **or**
- Add a honeypot field and server-side validation.

---

## Audit checklist — cyber security

Use this list in addition to `references/pcp-checklist.md §Security`.

### Admin access control

- [ ] Every `add_menu_page()` / `add_submenu_page()` capability is not `'read'`
- [ ] Every `add_menu_page()` / `add_submenu_page()` capability matches the minimum role needed
- [ ] All render callbacks re-check `current_user_can()` at the top of the function
- [ ] All files in `admin/partials/` have both ABSPATH guard and `current_user_can()` check
- [ ] No admin-only AJAX action is registered on `wp_ajax_nopriv_`
- [ ] All `admin_post_{action}` handlers verify nonce and capability
- [ ] No REST endpoint uses `'__return_true'` as `permission_callback`

### Injection

- [ ] No `unserialize()` on user-supplied, cookie, or URL data — use `json_decode()`
- [ ] No `maybe_unserialize()` on externally-sourced values
- [ ] No `exec()`, `system()`, `passthru()`, `shell_exec()`, `proc_open()` with user input
- [ ] No XML parsed without `LIBXML_NONET | LIBXML_NOENT` flags and entity loading disabled

### File upload

- [ ] File type validated server-side via `wp_check_filetype_and_ext()`
- [ ] `$_FILES['type']` never trusted as proof of MIME type
- [ ] Executable extensions (`.php`, `.phar`, `.phtml`, `.sh`, etc.) explicitly blocked
- [ ] File uploads handled via `wp_handle_upload()`, not manual `move_uploaded_file()`

### Redirects and SSRF

- [ ] All redirects to user-supplied URLs use `wp_safe_redirect()` not `wp_redirect()`
- [ ] Any external domain redirect is against an explicit allowlist
- [ ] User-supplied URLs are not passed directly to `wp_remote_get()` / `wp_remote_post()`
- [ ] `wp_http_validate_url()` called before any programmatic outbound HTTP request with a dynamic URL

### Path traversal

- [ ] No file paths built from user input without `validate_file()` + `realpath()` boundary check
- [ ] `sanitize_file_name()` applied before using any filename from user input

### Cryptographic / credential

- [ ] No `md5()` or `sha1()` used for password hashing — use `wp_hash_password()`
- [ ] No API keys or secrets hardcoded — use constants in `wp-config.php` or env vars
- [ ] All API keys read via `defined()` check before use

### Privilege escalation

- [ ] No `role` or `capabilities` accepted from POST/GET and applied to `wp_update_user()`
- [ ] Any role change gated on `current_user_can( 'promote_users' )`
- [ ] IDOR: object-level `current_user_can( 'edit_post', $id )` used, not just generic capability

### Information disclosure

- [ ] No `phpinfo()` calls
- [ ] `is_admin()` not used as an access-control check (use `current_user_can()`)
- [ ] `$wpdb->last_error` not output to the browser
- [ ] No credentials, tokens, or internal paths in code comments or debug output
