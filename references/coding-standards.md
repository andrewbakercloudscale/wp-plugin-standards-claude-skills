# Coding Standards

Follows [WordPress PHP Coding Standards](https://developer.wordpress.org/coding-standards/wordpress-coding-standards/php/).

## Naming

| Element         | Convention                          | Example                        |
|-----------------|-------------------------------------|--------------------------------|
| Functions       | `plugin_slug_lowercase_underscores` | `my_plugin_get_settings()`    |
| Variables       | `lowercase_underscores`             | `$user_count`                 |
| Classes         | `Title_Case_Underscores`            | `My_Plugin_Admin`             |
| Class filenames | `class-hyphen-case.php`             | `class-my-plugin-admin.php`   |
| Constants       | `UPPER_CASE_UNDERSCORES`            | `MY_PLUGIN_VERSION`           |
| Hooks           | `plugin_slug_lowercase_underscores` | `my_plugin_after_save`        |

All functions, hooks, and constants must be prefixed with the plugin slug.

## File formatting

- `<?php` on line 1, no preceding whitespace
- No closing `?>` at the end of PHP-only files
- Indentation: tabs, not spaces
- Line endings: Unix LF
- Max line length: 100 characters for code, 80 for comments
- One class per file; one responsibility per class

## DocBlocks

Every file, class, method, and function requires a DocBlock. Mandatory.

```php
<?php
/**
 * Admin functionality for My Plugin.
 *
 * @package   My_Plugin
 * @subpackage My_Plugin/Admin
 * @since      1.0.0
 */

if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

/**
 * Handles admin page rendering and settings.
 *
 * @since 1.0.0
 */
class My_Plugin_Admin {

    /**
     * Retrieves plugin settings with defaults applied.
     *
     * @since  1.0.0
     * @since  1.3.0 Added validation for the timeout field.
     * @param  bool  $use_cache Whether to use cached value. Default true.
     * @return array            Associative array of settings.
     */
    public function get_settings( $use_cache = true ) {
```

Required tags: `@since` on everything. `@param` per parameter. `@return` if a value is returned. Add a new `@since` line for each non-trivial behavioural change.

Inline comments explain the *why*, not the *what*:

```php
// Use absint — negative IDs are invalid and cause a DB error.
$post_id = absint( $_GET['post_id'] );

// Remove and re-add to avoid infinite recursion during save.
remove_action( 'save_post', array( $this, 'on_save_post' ) );
wp_update_post( $post_data );
add_action( 'save_post', array( $this, 'on_save_post' ) );
```

## Class structure order

1. Constants
2. Static properties
3. Instance properties
4. `__construct()`
5. Public methods (alphabetical)
6. Protected methods (alphabetical)
7. Private methods (alphabetical)
8. Static methods

## Yoda conditions

```php
// Correct
if ( true === $is_active ) {}
if ( 'publish' === $post->post_status ) {}

// Wrong
if ( $is_active === true ) {}
```

## Spacing and braces

```php
// Correct
if ( $condition ) {
    my_function( $arg );
}

// Wrong — missing spaces, missing braces
if($condition) my_function ($arg);
```

Always use braces. Spaces inside parentheses for control structures. No spaces inside for function calls.

## Internationalisation

Every user-facing string must use an i18n function with the plugin text domain as a plain string literal.

```php
esc_html__( 'Settings saved.', 'my-plugin' )

sprintf(
    /* translators: %s: post title */
    esc_html__( 'Post "%s" was updated.', 'my-plugin' ),
    esc_html( $post_title )
)
```

Load the text domain on `init` via `load_plugin_textdomain()`.

## Error handling

Never suppress errors with `@`. Return `WP_Error` for recoverable failures.

```php
$result = file_get_contents( $path );
if ( false === $result ) {
    return new WP_Error( 'file_read_failed', esc_html__( 'Could not read file.', 'my-plugin' ) );
}
```

## JavaScript async error handling

Every `async` function called from an `onclick` attribute or event listener **must** wrap its
body in `try/catch`. Without this, any runtime error (e.g. calling `.style` on a non-existent
element, a failed `fetch`, or a non-JSON response) causes the Promise to reject silently —
no error surface, no user feedback, the operation just stops. This is the root cause of
"button does nothing" bugs.

**Required pattern:**

```js
// Error helper — logs to console and shows message in a status element
function myPluginErr(label, err) {
    console.error('[my-plugin] ' + label, err);
    const st = document.getElementById('my-status');
    if (st) st.textContent = 'Error: ' + (err ? err.message : label);
}

async function myPluginLoad() {
    try {
        const fd = new FormData();
        fd.append('action', 'my_plugin_action');
        fd.append('nonce', myPluginData.nonce);
        const r = await fetch(myPluginData.ajaxUrl, { method: 'POST', body: fd });
        const d = await r.json();
        if (!d.success) { myPluginErr('Server returned error', null); return; }
        // ... handle success ...
    } catch (err) {
        myPluginErr('myPluginLoad', err);
    }
}
```

**Rules:**
- Every `async` function gets a `try/catch` — no exceptions.
- The `catch` block **must** call `console.error()` (developer visibility) AND update a
  visible status element (user visibility). Silent `catch` blocks are not acceptable.
- Loops that call `async` functions (e.g. batch processing) should use `try/catch` per-call
  so a single failure does not abort the whole batch, and use `finally` to re-enable
  disabled buttons.
- All `document.getElementById()` calls must be null-checked before accessing properties
  on the result. An unchecked null will throw inside an async function and kill it silently
  if not wrapped in try/catch.

**Audit checklist:**
- [ ] Every `async function` has `try { ... } catch(err) { ... }` wrapping its body
- [ ] Every `catch` logs with `console.error()` and surfaces a message to the user
- [ ] Buttons disabled at the start of an operation are re-enabled in `finally {}`
- [ ] All `getElementById` results are null-checked before property access
