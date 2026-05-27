---
name: migrate-theme-css-to-bem-scss
description: Convert flat or nested CSS to BEM SCSS with one block per file, no deep descendant selectors, modifiers via --modifier, and design tokens via CSS custom properties. Use when a theme has unstructured CSS that needs to follow the Hummingbird BEM component pattern.
---

## Requirements

- Theme name and location.
- List of CSS/SCSS files to migrate.
- Design tokens already defined (or to be created): colors, spacing, typography, breakpoints.
- Block names for each visual component (product-card, header, footer, breadcrumb, etc.).

## Steps

1. Identify visual blocks by auditing the templates. Each distinct, reusable UI piece becomes a BEM block:

    ```bash
    grep -rn 'class="' themes/mytheme/templates/ | sed 's/.*class="//;s/".*//' | tr ' ' '\n' | sort -u
    ```

2. Create the design tokens file `assets/scss/_settings.scss`:

    ```scss
    :root {
      --ps-color-primary: #25b9d7;
      --ps-color-text: #212b36;
      --ps-spacing-sm: 0.5rem;
      --ps-spacing-md: 1rem;
      --ps-spacing-lg: 1.5rem;
      --ps-radius-sm: 4px;
      --ps-font-family: system-ui, sans-serif;
    }
    ```

3. For each block, create `assets/scss/components/_block-name.scss`. Follow BEM strictly:

    ```scss
    // _product-card.scss
    .product-card {
      display: flex;
      flex-direction: column;
      border-radius: var(--ps-radius-sm);
      padding: var(--ps-spacing-md);

      &__image {
        width: 100%;
        aspect-ratio: 1;
        object-fit: cover;
      }

      &__title {
        font-size: 1rem;
        color: var(--ps-color-text);
      }

      &__price {
        font-weight: 700;
      }

      &--featured {
        border: 2px solid var(--ps-color-primary);
      }
    }
    ```

4. Create the components index `assets/scss/_components.scss`:

    ```scss
    @forward 'components/product-card';
    @forward 'components/header';
    @forward 'components/footer';
    @forward 'components/breadcrumb';
    ```

5. Wire the main entry point `assets/scss/theme.scss`:

    ```scss
    @use 'settings';
    @use 'components';
    ```

6. Update templates to use BEM class names:

    ```html
    <!-- Before -->
    <div class="product-container featured">
      <img class="product-img" />
      <h3 class="product-name">...</h3>
    </div>

    <!-- After -->
    <div class="product-card product-card--featured">
      <img class="product-card__image" />
      <h3 class="product-card__title">...</h3>
    </div>
    ```

7. Remove the old CSS files and update `config/theme.yml` to reference only the compiled output.

8. Build and verify:

    ```bash
    npm run build --prefix themes/mytheme
    ```

9. Visually inspect each page family for regressions. Use browser DevTools to confirm no orphan class selectors remain in the stylesheet.

## Do

- One block per `_block-name.scss` file; never put two blocks in one file.
- Use `&__element` and `&--modifier` nesting only one level deep.
- Reference design tokens via `var(--ps-*)` instead of hardcoded values.
- Use `@forward` (not `@import`) in the components index for proper Sass module scoping.

## Don't

- Don't nest selectors deeper than Block > Element (e.g. `.block__el__sub` is wrong; flatten to `.block__sub`).
- Don't use descendant selectors (`.block .element`); use BEM `&__element`.
- Don't use `!important`; if specificity conflicts arise, refactor the block boundary.
- Don't mix Bootstrap utility classes inside BEM blocks; keep them in templates only.

## Related skills

- `theme-add-bem-component` - adding a new BEM component from scratch.
- `migrate-classic-to-hummingbird` - the broader theme migration context.
- `migrate-theme-jquery-to-vanilla` - JS bindings for these BEM blocks.
