---
name: migrate-theme-jquery-to-vanilla
description: Systematically remove jQuery from a theme's JavaScript, replacing with vanilla TypeScript and Hummingbird's `data-ps-*` binding pattern. Use when a theme still depends on jQuery and must adopt the PS 9 Hummingbird-style JS architecture.
---

## Requirements

- Confirm the theme has a build toolchain with TypeScript support (Webpack/Vite + `tsconfig.json`).
- Inventory jQuery usage: `grep -rn '\$(' src/js/` or `grep -rn 'jQuery' src/js/`.
- Confirm no third-party scripts hard-depend on `window.jQuery` (or plan a shim for those).

## Steps

1. **Inventory jQuery patterns.** Common categories:

    | jQuery pattern | Vanilla equivalent |
    |---------------|-------------------|
    | `$(selector)` | `document.querySelector(selector)` / `querySelectorAll` |
    | `$(el).on('click', fn)` | `el.addEventListener('click', fn)` |
    | `$(document).ready(fn)` | `document.addEventListener('DOMContentLoaded', fn)` |
    | `$.ajax({ url, ... })` | `fetch(url, { ... })` |
    | `$(el).addClass('x')` | `el.classList.add('x')` |
    | `$(el).hide()` / `.show()` | `el.hidden = true` / `el.hidden = false` |
    | `$(el).val()` | `(el as HTMLInputElement).value` |
    | `$.each(arr, fn)` | `for (const item of arr) { ... }` |
    | `$(el).data('key')` | `el.dataset.key` |
    | `$(el).trigger('event')` | `el.dispatchEvent(new Event('event'))` |

2. **Adopt the `data-ps-*` binding pattern** from Hummingbird. Instead of querying by CSS class, use data attributes as the JS contract:

    ```html
    <!-- Template -->
    <button data-ps-action="add-to-cart" data-ps-product-id="{{ product.id }}">
      Add to cart
    </button>
    ```

    ```ts
    // src/js/components/add-to-cart.ts
    const buttons = document.querySelectorAll<HTMLButtonElement>('[data-ps-action="add-to-cart"]');
    buttons.forEach((btn) => {
      btn.addEventListener('click', async () => {
        const productId = btn.dataset.psProductId;
        const response = await fetch(`/cart/add/${productId}`, { method: 'POST' });
        // handle response
      });
    });
    ```

3. **Replace `$.ajax` with `fetch`:**

    ```ts
    // BEFORE
    $.ajax({
      url: prestashop.urls.base_url + 'module/mymodule/ajax',
      type: 'POST',
      data: { action: 'refresh', id: itemId },
      success(data) { updateUI(data); },
      error(xhr) { console.error(xhr.responseText); }
    });

    // AFTER
    const response = await fetch(prestashop.urls.base_url + 'module/mymodule/ajax', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ action: 'refresh', id: itemId }),
    });
    if (!response.ok) throw new Error(`HTTP ${response.status}`);
    const data = await response.json();
    updateUI(data);
    ```

4. **Replace jQuery event delegation:**

    ```ts
    // BEFORE
    $(document).on('click', '.product-miniature', function() { ... });

    // AFTER
    document.addEventListener('click', (e) => {
      const target = (e.target as HTMLElement).closest('[data-ps-component="product-miniature"]');
      if (!target) return;
      // handle
    });
    ```

5. **Remove jQuery from the build:**

    ```bash
    npm uninstall jquery @types/jquery
    ```

    Remove any `ProvidePlugin` entry for `$` or `jQuery` in `webpack.config.js`.

6. **Handle third-party scripts** that require `window.jQuery`:
   - If possible, replace the library (e.g. Slick -> Swiper, Select2 -> Choices.js).
   - If not, load jQuery as an external only for that script, isolated from your main bundle.

7. **Validate:**

    ```bash
    npm run build
    npm run lint
    # Manual test: all interactive elements (cart, search, filters, modals)
    ```

## Do

- Use `data-ps-*` attributes as the binding contract; CSS classes are for styling only.
- Type DOM queries with generics: `querySelector<HTMLInputElement>(...)`.
- Use event delegation on `document` for dynamically inserted elements.

## Don't

- Don't replace jQuery with another DOM library (Zepto, Cash); go vanilla.
- Don't bind JS behaviour to CSS classes; markup refactors will silently break JS.
- Don't leave `window.$` or `window.jQuery` globals if no script needs them.

## Canonical examples

- [devdocs - Hummingbird](https://devdocs.prestashop-project.org/9/themes/hummingbird/)
- [`PrestaShop/hummingbird` src/js/](https://github.com/PrestaShop/hummingbird/tree/develop/src/js) - reference vanilla TS with data-ps-* bindings.

## Related skills

- `theme-data-ps-bindings` - full explanation of the data-ps-* pattern.
- `migrate-classic-to-hummingbird` - broader theme migration orchestration.
- `theme-asset-pipeline` - build tooling (Webpack/Vite) configuration.
