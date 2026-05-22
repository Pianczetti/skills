---
name: migrate-theme-jquery-to-vanilla
description: Systematically remove jQuery from a PrestaShop 9 theme by replacing selectors, AJAX calls, events, and DOM ready patterns with vanilla TypeScript and data-ps-* bindings. Use when a theme depends on jQuery and needs to adopt Hummingbird's zero-jQuery architecture.
---

## Requirements

- Theme name and location.
- List of JS files containing jQuery usage.
- Whether third-party modules loaded in the theme depend on jQuery (if so, keep a compatibility shim or lazy-load jQuery only for those modules).

## Steps

1. Audit jQuery usage:

    ```bash
    grep -rn '\$(' themes/mytheme/assets/js/
    grep -rn 'jQuery\|\\$.ajax\|\\$.get\|\\$.post' themes/mytheme/assets/js/
    ```

2. Replace selectors:

    ```typescript
    // Before
    $('.product-card')
    $('#main-header')

    // After
    document.querySelectorAll('[data-ps-component="product-card"]')
    document.querySelector('[data-ps-element="main-header"]')
    ```

3. Replace AJAX calls:

    ```typescript
    // Before
    $.ajax({ url: '/api/cart', method: 'POST', data: payload, success: cb });

    // After
    const response = await fetch('/api/cart', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', 'X-Requested-With': 'XMLHttpRequest' },
      body: JSON.stringify(payload),
    });
    const data = await response.json();
    ```

4. Replace event bindings:

    ```typescript
    // Before
    $(document).on('click', '.btn-add', handler);

    // After
    document.addEventListener('click', (e) => {
      const target = (e.target as HTMLElement).closest('[data-ps-action="add"]');
      if (target) handler(e);
    });
    ```

5. Replace `$(document).ready`:

    ```typescript
    // Before
    $(document).ready(function() { /* ... */ });

    // After
    document.addEventListener('DOMContentLoaded', () => { /* ... */ });
    // Or use a module pattern with top-level await if bundler supports it
    ```

6. Replace DOM manipulation:

    ```typescript
    // Before
    $el.addClass('active').show().html(content);

    // After
    el.classList.add('active');
    el.style.display = '';
    el.innerHTML = content;
    ```

7. Replace animations:

    ```typescript
    // Before
    $el.fadeIn(300);

    // After (CSS transition)
    el.classList.add('is-visible');
    // With CSS: .is-visible { opacity: 1; transition: opacity 300ms; }
    ```

8. Wire all interactive components via `data-ps-component` so the component loader auto-initializes them:

    ```html
    <div data-ps-component="quantity-selector" data-ps-min="1" data-ps-max="99">
    ```

    ```typescript
    // components/QuantitySelector.ts
    export class QuantitySelector {
      constructor(private el: HTMLElement) {
        this.el.querySelector('[data-ps-action="increment"]')
          ?.addEventListener('click', () => this.increment());
      }
    }
    ```

9. Remove jQuery from `package.json` dependencies and `config/theme.yml` asset declarations. If a module still needs jQuery, lazy-load it:

    ```typescript
    async function loadjQuery(): Promise<void> {
      if (!window.jQuery) {
        await import('jquery');
      }
    }
    ```

10. Rebuild and test all interactive features (add to cart, quantity selector, mobile menu, modals, dropdowns).

## Do

- Use `data-ps-*` attributes for all JS hooks instead of CSS classes.
- Prefer CSS transitions/animations over JS-driven animation.
- Use event delegation on `document` for dynamically inserted elements.

## Don't

- Don't remove jQuery if third-party modules break without a compatibility path.
- Don't use `innerHTML` with user-supplied data; use `textContent` or sanitize.
- Don't bind events to CSS class selectors (`.btn-add`); use `data-ps-action` attributes.

## Related skills

- `migrate-classic-to-hummingbird` - the broader theme migration.
- `theme-data-ps-bindings` - the data-ps-* attribute convention.
- `theme-tdd-component` - testing migrated TypeScript components.
