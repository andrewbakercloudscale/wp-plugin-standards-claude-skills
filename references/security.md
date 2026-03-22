# Security

## Nonce verification

Every form, AJAX handler, and REST endpoint must verify a nonce before processing.

```php
// Form — output
wp_nonce_field( 'my_plugin_action', 'my_plugin_nonce' );

// Form — verify
if ( ! isset( $_POST['my_plugin_nonce'] ) ||
     ! wp_verify_nonce( sanitize_text_field( wp_unslash( $_POST['my_plugin_nonce'] ) ), 'my_plugin_action' ) ) {
    wp_die( esc_html__( 'Security check failed.', 'plugin-slug' ) );
}

// AJAX handler
function my_plugin_ajax_handler() {
    check_ajax_referer( 'my_plugin_ajax_nonce', 'nonce' );
    wp_send_json_success( $data );
}

// REST endpoint
register_rest_route( 'plugin-slug/v1', '/endpoint', array(
    'methods'             => 'POST',
    'callback'            => 'my_plugin_rest_callback',
    'permission_callback' => function() {
        return current_user_can( 'manage_options' );
    },
) );
```

### Delegated nonce verification and PHPCS suppression

When a plugin centralises nonce + capability checks in a shared helper (e.g. `ajax_check()`), PHPCS cannot trace the call and raises `WordPress.Security.NonceVerification.Missing` on every `$_POST` access below the helper, even though the nonce was already verified. This is a false positive.

**Pattern — shared helper with per-handler suppression:**

```php
// Shared helper (called by all AJAX handlers):
private function ajax_check(): void {
    check_ajax_referer( 'my_plugin_nonce', 'nonce' );
    if ( ! current_user_can( 'manage_options' ) ) {
        wp_send_json_error( 'Forbidden', 403 );
    }
}

// AJAX handler:
public function ajax_my_action(): void {
    $this->ajax_check(); // verifies nonce and capability

    // phpcs:disable WordPress.Security.NonceVerification.Missing -- nonce checked via ajax_check()
    $post_id = (int) ( $_POST['post_id'] ?? 0 );
    $value   = sanitize_text_field( wp_unslash( $_POST['value'] ?? '' ) );
    // phpcs:enable WordPress.Security.NonceVerification.Missing
}
```

Rules:
- The suppression block must be as narrow as possible — open immediately before the first `$_POST` read and close after the last.
- The inline comment must name the helper (`ajax_check()`) so a reader can verify the nonce is genuinely checked.
- Do **not** suppress `NonceVerification.Missing` on a handler that does not call `check_ajax_referer()` or `wp_verify_nonce()` somewhere in the call stack — fix the missing check instead.

## Input sanitisation

Sanitise on the way in. Always unslash superglobals first.

```php
$value = sanitize_text_field( wp_unslash( $_POST['field'] ) );
```

| Data type      | Sanitiser                       |
|----------------|---------------------------------|
| Plain text     | `sanitize_text_field()`        |
| Textarea       | `sanitize_textarea_field()`    |
| Integer        | `absint()` / `intval()`        |
| Email          | `sanitize_email()`             |
| URL (storage)  | `esc_url_raw()`                |
| Rich HTML      | `wp_kses_post()`               |
| Filename       | `sanitize_file_name()`         |
| Key / slug     | `sanitize_key()`               |

## Output escaping

Escape at the point of output. Never escape and store.

```php
echo '<a href="' . esc_url( $url ) . '">' . esc_html( $label ) . '</a>';
```

| Context           | Escaper                                         |
|-------------------|-------------------------------------------------|
| HTML text         | `esc_html()`                                   |
| HTML attribute    | `esc_attr()`                                   |
| URL               | `esc_url()`                                    |
| JavaScript string | `esc_js()`                                     |
| Translated output | `esc_html__()`, `esc_attr__()`, `esc_html_e()` |
| Trusted HTML      | `wp_kses_post()`                               |

## Database queries

All dynamic queries use `$wpdb->prepare()`. No interpolated SQL.

```php
$results = $wpdb->get_results(
    $wpdb->prepare(
        "SELECT * FROM {$wpdb->prefix}my_table WHERE user_id = %d AND status = %s",
        absint( $user_id ),
        sanitize_key( $status )
    )
);
```

Placeholders: `%d` integer, `%s` string, `%f` float.

## Capability checks

Use the most specific capability available.

```php
if ( ! current_user_can( 'manage_options' ) ) {
    wp_die( esc_html__( 'You do not have permission to do this.', 'plugin-slug' ) );
}
```

Do not use `manage_options` as a blanket — use `edit_posts`, `publish_posts`, etc. where appropriate.

## Filesystem access

Never use PHP file functions directly. Use the WP Filesystem API.

```php
global $wp_filesystem;
WP_Filesystem();
$contents = $wp_filesystem->get_contents( $file_path );
$wp_filesystem->put_contents( $file_path, $contents, FS_CHMOD_FILE );
```

## Direct access guard

Every PHP file except the main plugin file must begin with this immediately after `<?php`:

```php
if ( ! defined( 'ABSPATH' ) ) {
    exit;
}
```

## Options and transients

Sanitise before saving. Escape on output. Treat stored values as untrusted.

```php
update_option( 'my_plugin_setting', sanitize_text_field( $value ) );
echo esc_html( get_option( 'my_plugin_setting', '' ) );
```

## Admin access control

Admin pages must never be publicly reachable. Three layers are required:

**1. Correct capability on registration** — never use `'read'` (granted to Subscribers):

```php
add_menu_page( ..., 'manage_options', 'my-plugin', array( $this, 'render_page' ) );
```

**2. Capability re-check in the render callback:**

```php
public function render_page(): void {
    if ( ! current_user_can( 'manage_options' ) ) {
        wp_die( esc_html__( 'You do not have permission to view this page.', 'plugin-slug' ) );
    }
}
```

**3. ABSPATH guard + capability check in every partial template** (`admin/partials/*.php`):

```php
if ( ! defined( 'ABSPATH' ) ) { exit; }
if ( ! current_user_can( 'manage_options' ) ) {
    wp_die( esc_html__( 'Forbidden.', 'plugin-slug' ) );
}
```

`is_admin()` checks whether the admin area is loaded — not whether the user is an
administrator. **Never use `is_admin()` as an access control check.**

## Object injection

Never call `unserialize()` on user-supplied data or any externally-sourced value.
Use `json_decode()` instead:

```php
// WRONG
$data = unserialize( base64_decode( $_POST['payload'] ) );

// CORRECT
$data = json_decode( wp_unslash( $_POST['payload'] ?? '' ), true );
if ( ! is_array( $data ) ) {
    wp_send_json_error( 'Invalid data.' );
}
```

`maybe_unserialize()` is safe only for values your own plugin previously serialised
and stored. Never call it on user input or third-party option values.

## Open redirect

Use `wp_safe_redirect()` for any redirect that follows a user-supplied URL.
`wp_safe_redirect()` blocks cross-domain redirects:

```php
$url = esc_url_raw( wp_unslash( $_GET['redirect'] ?? '' ) );
wp_safe_redirect( $url );
exit;
```

For legitimate cross-domain redirects, use an explicit domain allowlist rather
than `wp_redirect()` directly.

## Path traversal

Validate every file path built from user input:

```php
$file     = sanitize_file_name( wp_unslash( $_GET['file'] ?? '' ) );
$base_dir = realpath( plugin_dir_path( __FILE__ ) . 'templates/' );
$full     = realpath( $base_dir . DIRECTORY_SEPARATOR . $file );

if ( validate_file( $file ) !== 0 || ! $full || strpos( $full, $base_dir ) !== 0 ) {
    wp_die( esc_html__( 'Invalid file.', 'plugin-slug' ) );
}
```

`validate_file()` returns `0` for safe paths, non-zero for traversal attempts.

## SSRF (Server-Side Request Forgery)

Never pass a user-supplied URL directly to `wp_remote_get()` / `wp_remote_post()`:

```php
$url = esc_url_raw( wp_unslash( $_POST['url'] ?? '' ) );
if ( ! wp_http_validate_url( $url ) ) {
    wp_send_json_error( 'Invalid URL.' );
}
wp_remote_get( $url );
```

WordPress 5.9+ blocks private IP ranges by default. Do not override
`http_request_host_is_external` to re-allow internal requests.

## IDOR (Insecure Direct Object Reference)

Check the specific object capability, not just the generic one:

```php
// WRONG — any editor can read any post ID
if ( ! current_user_can( 'edit_posts' ) ) { wp_die(...); }

// CORRECT — only editors with access to this specific post
if ( ! current_user_can( 'edit_post', absint( $_GET['post_id'] ) ) ) { wp_die(...); }
```

## Full cyber security reference

See `references/cyber-security.md` for the complete OWASP Top 10 WordPress mapping,
file upload safety, privilege escalation patterns, XXE, race conditions, and the
full audit checklist.
