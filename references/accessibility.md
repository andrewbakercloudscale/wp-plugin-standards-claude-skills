# Accessibility

All plugin UI must meet WCAG 2.1 AA. These rules apply to every admin screen, form, notice, and modal the plugin renders.

## Labels and ARIA

Every form input must have an associated `<label>` with matching `for` and `id`. Never use placeholder text as the only label.

```php
echo '<label for="my-field">' . esc_html__( 'Field Label', 'plugin-slug' ) . '</label>';
echo '<input type="text" id="my-field" name="my_field" value="' . esc_attr( $value ) . '" />';
```

Use ARIA attributes for dynamic content:

```html
<div role="alert" aria-live="polite" id="my-plugin-notice"></div>
<button aria-expanded="false" aria-controls="my-panel">Toggle</button>
<div id="my-panel" aria-hidden="true">...</div>
```

## Keyboard navigation

- All interactive elements must be keyboard operable
- Never remove focus outlines with `outline: none` unless a visible custom focus style replaces them
- Tab order must follow visual reading order
- Never use `tabindex` values greater than 0

## Colour and contrast

- Minimum contrast ratio: 4.5:1 for normal text, 3:1 for large text
- Never use colour alone to convey information — always pair with an icon, label, or pattern

## Screen reader support

Decorative images use `alt=""`. Meaningful images use descriptive alt text.

Admin notices must include a visually hidden prefix:

```php
echo '<div class="notice notice-error"><p>';
echo '<span class="screen-reader-text">' . esc_html__( 'Error: ', 'plugin-slug' ) . '</span>';
echo esc_html( $message ) . '</p></div>';
```

## Modals

- Trap focus while open; return focus to the trigger element on close
- Use `role="dialog"`, `aria-modal="true"`, and `aria-labelledby` pointing to the dialog title
