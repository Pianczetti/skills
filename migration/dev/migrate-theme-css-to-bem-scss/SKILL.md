---
name: migrate-theme-css-to-bem-scss
description: Convert flat or unstructured CSS/SCSS into BEM component architecture with the layered SCSS structure Hummingbird uses. Use when a theme has monolithic stylesheets and must adopt component-level organization.
---

## Requirements

- Identify the current CSS/SCSS entry point and total file count.
- Confirm the theme uses a build tool (Webpack/Vite) that compiles SCSS.
- Review Hummingbird's SCSS layer structure as the target architecture.

## Steps

1. **Understand the target layer structure** (Hummingbird convention):

    ```
    src/scss/
    ├── _settings.scss        # CSS custom properties (design tokens)
    ├── _tools.scss           # Mixins, functions
    ├── _generic.scss         # Resets, box-sizing
    ├── _elements.scss        # Base HTML elements (h1, p, a)
    ├── _objects.scss         # Layout primitives (container, grid)
    ├── components/           # BEM blocks (one partial per block)
    │   ├── _product-card.scss
    │   ├── _header.scss
    │   └── _cart-summary.scss
    ├── _utilities.scss       # Single-purpose helpers
    └── main.scss             # Imports all layers in order
    ```

2. **Audit existing styles.** Identify visual blocks (header, product-card, footer, cart, etc.):

    ```bash
    grep -rn '^\.' src/scss/ | sed 's/:.*//' | sort -u  # list all class selectors
    ```

3. **Define design tokens** as CSS custom properties in `_settings.scss`:

    ```scss
    // _settings.scss
    :root {
      --ps-color-primary: #25b9d7;
      --ps-color-text: #232b30;
      --ps-spacing-sm: 0.5rem;
      --ps-spacing-md: 1rem;
      --ps-border-radius: 4px;
      --ps-font-family: system-ui, sans-serif;
    }
    ```

4. **Refactor each visual block to BEM.** One partial per Block:

    ```scss
    // components/_product-card.scss
    .product-card {
      border: 1px solid var(--ps-color-border);
      border-radius: var(--ps-border-radius);
      padding: var(--ps-spacing-md);

      &__image {
        width: 100%;
        aspect-ratio: 1;
        object-fit: cover;
      }

      &__title {
        font-size: 1rem;
        margin-top: var(--ps-spacing-sm);
      }

      &__price {
        font-weight: 700;
        color: var(--ps-color-primary);
      }

      &--highlighted {
        border-color: var(--ps-color-primary);
      }
    }
    ```

    Naming rules:
    - **Block**: standalone component name (`product-card`)
    - **Element**: part of the block, separated by `__` (`product-card__title`)
    - **Modifier**: variant, separated by `--` (`product-card--highlighted`)

5. **Update templates** to use BEM classes and map to `data-ps-*` attributes:

    ```html
    <article class="product-card product-card--highlighted" data-ps-component="product-card">
      <img class="product-card__image" src="..." alt="...">
      <h3 class="product-card__title">{$product.name}</h3>
      <span class="product-card__price">{$product.price}</span>
    </article>
    ```

6. **Wire imports** in `main.scss` following the layer order:

    ```scss
    // main.scss
    @import 'settings';
    @import 'tools';
    @import 'generic';
    @import 'elements';
    @import 'objects';
    @import 'components/product-card';
    @import 'components/header';
    @import 'components/cart-summary';
    @import 'utilities';
    ```

7. **Build and validate:**

    ```bash
    npm run build
    npm run lint        # Stylelint with BEM plugin catches naming violations
    ```

## Do

- One partial per Block; never mix multiple blocks in one file.
- Use CSS custom properties for all magic values (colors, spacing, radii).
- Keep specificity flat: avoid nesting deeper than Block > Element.
- Use Stylelint with `stylelint-selector-bem-pattern` to enforce naming.

## Don't

- Don't nest more than one level deep inside a block (no `.block .block__el .block__sub`).
- Don't use IDs for styling; BEM classes provide sufficient specificity.
- Don't create Elements of Elements (`.block__el__sub`); flatten to `.block__sub`.
- Don't put layout rules (grid, flex container) inside a component; use Object partials (`_objects.scss`).

## Canonical examples

- [devdocs - Hummingbird](https://devdocs.prestashop-project.org/9/themes/hummingbird/)
- [`PrestaShop/hummingbird` src/scss/](https://github.com/PrestaShop/hummingbird/tree/develop/src/scss) - reference BEM structure.

## Related skills

- `theme-add-bem-component` - creating a single new BEM component.
- `migrate-classic-to-hummingbird` - broader theme migration workflow.
- `theme-asset-pipeline` - build configuration for SCSS compilation.
