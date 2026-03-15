# Performance

## Asset enqueuing

**This is the most common WordPress.org submission rejection.** Every `<script>` and `<style>` tag echoed directly into page output is a violation. Use WordPress API functions exclusively.

Never enqueue CSS or JS globally. Scope to the correct hook and screen. Version strings must be the plugin version constant — never `false` or a hardcoded string.

### Correct hooks

| Context | Script hook | Style hook |
|---------|-------------|------------|
| Frontend | `wp_enqueue_scripts` | `wp_enqueue_scripts` |
| Admin | `admin_enqueue_scripts` | `admin_enqueue_scripts` |
| Login page | `login_enqueue_scripts` | `login_enqueue_scripts` |

```php
public function enqueue_admin_assets( $hook ) {
    // Always gate to the specific admin screen — never load globally.
    if ( 'settings_page_my-plugin' !== $hook ) {
        return;
    }
    wp_enqueue_style(
        'my-plugin-admin',
        plugin_dir_url( __FILE__ ) . 'assets/css/admin.css',
        array(),
        MY_PLUGIN_VERSION
    );
    wp_enqueue_script(
        'my-plugin-admin',
        plugin_dir_url( __FILE__ ) . 'assets/js/admin.js',
        array( 'jquery' ),
        MY_PLUGIN_VERSION,
        true // Always load scripts in the footer unless there is a documented reason not to.
    );
}
add_action( 'admin_enqueue_scripts', array( $this, 'enqueue_admin_assets' ) );
```

### Inline JavaScript — never echo `<script>` tags

Wrong — this is the most common rejection reason on WordPress.org:

```php
// WRONG — never do this
echo '<script>var myData = ' . json_encode( $data ) . ';</script>';
echo '<script src="' . $url . '"></script>';
```

Correct — use `wp_localize_script()` for data, `wp_add_inline_script()` for code:

```php
// Pass PHP data to JS
wp_localize_script(
    'my-plugin-admin',
    'myPluginData',
    array(
        'ajaxUrl' => admin_url( 'admin-ajax.php' ),
        'nonce'   => wp_create_nonce( 'my_plugin_ajax_nonce' ),
        'strings' => array(
            'saved' => esc_html__( 'Settings saved.', 'my-plugin' ),
            'error' => esc_html__( 'An error occurred.', 'my-plugin' ),
        ),
    )
);

// Add inline JS that depends on an already-registered script
wp_add_inline_script( 'my-plugin-admin', 'myPlugin.init();', 'after' );
```

### Inline CSS — never echo `<style>` tags

Wrong:

```php
// WRONG — never do this
echo '<style>.my-plugin-box { color: red; }</style>';
```

Correct — use `wp_add_inline_style()`:

```php
// Attach inline CSS to a registered stylesheet
$custom_css = '.my-plugin-box { color: ' . sanitize_hex_color( $colour ) . '; }';
wp_add_inline_style( 'my-plugin-admin', $custom_css );
```

### defer and async attributes (WordPress 6.3+)

As of WordPress 6.3, pass `defer` or `async` via the `$args` array instead of filter hacks:

```php
wp_enqueue_script(
    'my-plugin-deferred',
    plugin_dir_url( __FILE__ ) . 'assets/js/deferred.js',
    array(),
    MY_PLUGIN_VERSION,
    array(
        'in_footer' => true,
        'strategy'  => 'defer', // or 'async'
    )
);
```

Reference: https://make.wordpress.org/core/2023/07/14/registering-scripts-with-async-and-defer-attributes-in-wordpress-6-3/

### Custom script attributes (WordPress 5.7+)

For attributes beyond `defer`/`async` (e.g. `type="module"`, `crossorigin`), use the `script_loader_tag` filter:

```php
add_filter( 'script_loader_tag', 'my_plugin_add_module_type', 10, 3 );

function my_plugin_add_module_type( $tag, $handle, $src ) {
    if ( 'my-plugin-module' !== $handle ) {
        return $tag;
    }
    return '<script type="module" src="' . esc_url( $src ) . '"></script>' . "\n";
}
```

Reference: https://make.wordpress.org/core/2021/02/23/introducing-script-attributes-related-functions-in-wordpress-5-7/

### Quick reference — which function for which job

| Job | Correct function |
|-----|-----------------|
| Load a JS file | `wp_enqueue_script()` |
| Register a JS file without loading | `wp_register_script()` |
| Add inline JS after a registered script | `wp_add_inline_script()` |
| Pass PHP data to JS | `wp_localize_script()` |
| Load a CSS file | `wp_enqueue_style()` |
| Register a CSS file without loading | `wp_register_style()` |
| Add inline CSS after a registered style | `wp_add_inline_style()` |
| Add `defer` / `async` (WP 6.3+) | `strategy` arg in `wp_enqueue_script()` |
| Add other script attributes | `script_loader_tag` filter |

## WP_Query / get_posts exclusion parameters

`post__not_in` (and `author__not_in`, `tag__not_in`, `category__not_in`) force MySQL to perform
a full table scan with a `NOT IN` clause. On large sites this can be significantly slower than
an equivalent positive query. PCP flags these with `WordPressVIPMinimum.Performance.WPQueryParams.PostNotIn_post__not_in`.

**When it is acceptable:**
- Excluding a small fixed list (1–3 IDs) where the `posts_per_page` cap limits the query scope
- No practical alternative exists (e.g. excluding the current post from a related-posts pool)

**Required:** Add an inline `phpcs:ignore` comment with a short justification on the same line
as the `post__not_in` key.

```php
// Acceptable — excluding current post from a pool capped at 20; no alternative
$args = [
    'posts_per_page' => 20,
    'post__not_in'   => [ $post_id ], // phpcs:ignore WordPressVIPMinimum.Performance.WPQueryParams.PostNotIn_post__not_in -- pool capped at 20; current post exclusion required
];
```

**When to refactor instead:**
- The exclusion list is dynamic and could grow large (dozens of IDs)
- The query is run on every page load without caching
- An alternative query structure exists (e.g. `meta_query`, `tax_query`, or a direct `$wpdb` query with a `NOT IN` on a pre-fetched small list)

## Database queries

Never run queries inside loops. Fetch all data before iterating.

```php
// Wrong — N queries
foreach ( $post_ids as $id ) {
    $meta = get_post_meta( $id, 'my_key', true );
}

// Correct — prime cache first
update_postmeta_cache( $post_ids );
foreach ( $post_ids as $id ) {
    $meta = get_post_meta( $id, 'my_key', true );
}
```

## Transients

Cache expensive computations and external API calls. Delete transients on deactivation and when source data changes.

```php
public function get_cached_data() {
    $data = get_transient( 'my_plugin_expensive_data' );
    if ( false === $data ) {
        $data = $this->compute_expensive_data();
        set_transient( 'my_plugin_expensive_data', $data, HOUR_IN_SECONDS );
    }
    return $data;
}
```

## Options API

Use one serialised array rather than many individual options.

```php
// Wrong
update_option( 'my_plugin_setting_a', $a );
update_option( 'my_plugin_setting_b', $b );

// Correct
update_option( 'my_plugin_settings', array( 'setting_a' => $a, 'setting_b' => $b ) );
```

Prevent unnecessary autoloading for data not needed on every page:

```php
add_option( 'my_plugin_large_data', $data, '', false );
```

## Background processing

Long-running tasks use WP Cron or Action Scheduler — never block execution.

```php
if ( ! wp_next_scheduled( 'my_plugin_process_task' ) ) {
    wp_schedule_single_event( time(), 'my_plugin_process_task' );
}
add_action( 'my_plugin_process_task', array( $this, 'process_task' ) );
```
