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
