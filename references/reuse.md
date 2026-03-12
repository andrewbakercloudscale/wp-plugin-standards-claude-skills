# Code Reuse and Documentation

## 1. The Utils class

Every plugin must have `includes/class-PLUGINSLUG-utils.php`. This is the only place shared helper functions live. No helper function may be duplicated across files.

**Before writing any new function:**
1. Open `includes/class-my-plugin-utils.php`
2. Search for a function that does what you need — if it exists, call it
3. If it does not exist but will be needed in more than one place, add it to Utils
4. If later needed elsewhere, move it to Utils — never copy it

```php
<?php
/**
 * Shared utility functions for My Plugin.
 *
 * All helpers used in more than one class belong here.
 * Call My_Plugin_Utils::method_name() from any class that needs them.
 *
 * @package My_Plugin
 * @since   1.0.0
 */

if ( ! defined( 'ABSPATH' ) ) {
    exit;
}

/**
 * Utility helpers for My Plugin.
 *
 * @since 1.0.0
 */
class My_Plugin_Utils {

    /**
     * Retrieves a plugin option from the stored settings array.
     *
     * @since  1.0.0
     * @param  string $key     Option key.
     * @param  mixed  $default Default if key is absent.
     * @return mixed
     */
    public static function get_option( $key, $default = false ) {
        $settings = get_option( 'my_plugin_settings', array() );
        return isset( $settings[ $key ] ) ? $settings[ $key ] : $default;
    }

    /**
     * Logs a debug message when WP_DEBUG_LOG is enabled.
     *
     * Never call error_log() directly elsewhere — use this method.
     *
     * @since  1.0.0
     * @param  string $message The message to log.
     * @param  mixed  $context Optional context data.
     * @return void
     */
    public static function log( $message, $context = null ) {
        if ( ! defined( 'WP_DEBUG_LOG' ) || ! WP_DEBUG_LOG ) {
            return;
        }
        $entry = '[My Plugin] ' . $message;
        if ( null !== $context ) {
            $entry .= ' | Context: ' . wp_json_encode( $context );
        }
        error_log( $entry ); // phpcs:ignore WordPress.PHP.DevelopmentFunctions
    }

    /**
     * Returns a sanitised integer from a superglobal array.
     *
     * @since  1.0.0
     * @param  array  $source  Superglobal ($_POST, $_GET, etc.).
     * @param  string $key     Key to retrieve.
     * @param  int    $default Default if key is absent.
     * @return int
     */
    public static function get_int( $source, $key, $default = 0 ) {
        return isset( $source[ $key ] ) ? absint( $source[ $key ] ) : $default;
    }

    /**
     * Returns a sanitised text string from a superglobal array.
     *
     * @since  1.0.0
     * @param  array  $source  Superglobal ($_POST, $_GET, etc.).
     * @param  string $key     Key to retrieve.
     * @param  string $default Default if key is absent.
     * @return string
     */
    public static function get_text( $source, $key, $default = '' ) {
        return isset( $source[ $key ] )
            ? sanitize_text_field( wp_unslash( $source[ $key ] ) )
            : $default;
    }
}
```

## 2. Version tracking

Three locations must always hold the same version string. Update all three on every release.

```php
// Location 1 — Plugin header (main plugin file)
 * Version: 1.4.0

// Location 2 — VERSION constant (immediately after the header block)
define( 'MY_PLUGIN_VERSION', '1.4.0' );
```

```
// Location 3 — readme.txt
Stable tag: 1.4.0
```

Use [Semantic Versioning](https://semver.org/): `MAJOR.MINOR.PATCH`.

## 3. CHANGELOG.md

Present in every plugin root. Updated on every commit that changes behaviour. [Keep a Changelog](https://keepachangelog.com/) format.

```markdown
# Changelog

All notable changes to My Plugin are documented here.

## [Unreleased]

## [1.4.0] - 2025-03-12
### Added
- Transient caching for the settings retrieval method.

### Changed
- Moved shared sanitisation helpers to Utils class.

### Fixed
- Nonce verification now runs before any POST data is read in the AJAX handler.

## [1.0.0] - 2025-01-15
### Added
- Initial release.
```

Valid sections: `Added`, `Changed`, `Deprecated`, `Removed`, `Fixed`, `Security`.

## 4. readme.txt format

```
=== My Plugin ===
Contributors:      your-wp-username
Tags:              tag1, tag2
Requires at least: 6.4
Tested up to:      6.7
Requires PHP:      8.0
Stable tag:        1.4.0
License:           GPLv2 or later
License URI:       https://www.gnu.org/licenses/gpl-2.0.html

Short description under 150 characters. No markup.

== Description ==

Full description.

== Installation ==

1. Step one.
2. Step two.

== Changelog ==

= 1.4.0 =
* Added transient caching.
* Fixed nonce verification order.

= 1.0.0 =
* Initial release.

== Upgrade Notice ==

= 1.4.0 =
Security fix included — update immediately.
```

## 5. Uninstall cleanup

`uninstall.php` must remove all plugin data — options, transients, custom tables, cron events.

```php
<?php
if ( ! defined( 'WP_UNINSTALL_PLUGIN' ) ) {
    exit;
}

delete_option( 'my_plugin_settings' );
delete_transient( 'my_plugin_expensive_data' );

global $wpdb;
$wpdb->query( "DROP TABLE IF EXISTS {$wpdb->prefix}my_plugin_table" ); // phpcs:ignore
```
