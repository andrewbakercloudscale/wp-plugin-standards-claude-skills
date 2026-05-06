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
// ⚠️ wp_verify_nonce() is pluggable — always pass sanitize_text_field( wp_unslash( ... ) ),
//    not just wp_unslash(). WordPress.org reviewers flag bare wp_unslash() on nonce values.

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

### Delegated nonce verification — always inline the call

**Rule: call `check_ajax_referer()` (or `wp_verify_nonce()` / `check_admin_referer()`) directly in every handler that reads `$_POST`, `$_GET`, `$_REQUEST`, or `$_FILES`.** PHPCS and the WordPress.org Plugin Check only recognise nonce verification when one of those three functions is called in the same function scope as the superglobal access. A shared helper that wraps the call is not traced.

**Correct — direct call in scope (Plugin Check passes):**

```php
// AJAX handler:
public function ajax_my_action(): void {
    if ( ! current_user_can( 'manage_options' ) ) {
        wp_send_json_error( 'Forbidden', 403 );
    }
    check_ajax_referer( 'my_plugin_nonce', 'nonce' );

    $post_id = (int) ( $_POST['post_id'] ?? 0 );
    $value   = sanitize_text_field( wp_unslash( $_POST['value'] ?? '' ) );
    wp_send_json_success( $data );
}
```

**Incorrect — helper wrapper (Plugin Check still fails):**

```php
// DO NOT DO THIS — Plugin Check flags every $_POST access below as NonceVerification.Missing
public function ajax_my_action(): void {
    $this->ajax_check(); // wraps check_ajax_referer() internally — not visible to PHPCS

    $post_id = (int) ( $_POST['post_id'] ?? 0 ); // WARNING: NonceVerification.Missing
}
```

If a helper wrapper already exists (e.g. `cs_verify_nonce()`, `ajax_check()`), replace every call site with the direct `check_ajax_referer()` call. The helper function itself can remain; just stop delegating through it from handlers.

Rules:
- Never suppress `WordPress.Security.NonceVerification.Missing` with `phpcs:disable/ignore` as a substitute for the direct call — the Plugin Check will still flag it.
- If you must suppress other `ValidatedSanitizedInput` rules on a `$_POST` line, that is fine; but `NonceVerification.Missing` must be resolved by the direct call, not suppression.
- Do **not** suppress `NonceVerification.Missing` on a handler that does not call `check_ajax_referer()` or `wp_verify_nonce()` directly in scope — fix the call site instead.

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

### Every `$_POST` / `$_GET` value must be sanitised — no exceptions (`InputNotSanitized`)

PHPCS flags any superglobal access that is not wrapped in a sanitiser, including parameters that look like internal options or numeric window values. Even if you know the value should be an integer, apply `absint()` (not a bare cast) so PHPCS can trace the sanitisation call.

```php
// WRONG — PHPCS: WordPress.Security.ValidatedSanitizedInput.InputNotSanitized
$window = $_POST['par_window'];
$window = (int) $_POST['par_window'];  // bare cast is not recognised by PHPCS

// CORRECT
$window = absint( wp_unslash( $_POST['par_window'] ?? 0 ) );
```

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

### Computed variables still require escaping (`OutputNotEscaped`)

PHPCS flags **every** variable echo'd without an escaping call, including internally-computed values such as counts, formatted strings, or concatenated arrays. "This variable wasn't from user input" is not a defence — escape at the point of output regardless.

```php
// WRONG — PHPCS: WordPress.Security.EscapeOutput.OutputNotEscaped on $total and $parts
echo '<p>Total: ' . $total . ' (' . $parts . ')</p>';

// CORRECT — escape every variable, even computed ones
echo '<p>Total: ' . esc_html( $total ) . ' (' . esc_html( $parts ) . ')</p>';
```

For arrays rendered as a list, escape each element before joining:

```php
// WRONG
echo implode( ', ', $parts );

// CORRECT
echo implode( ', ', array_map( 'esc_html', $parts ) );
```

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
