---
name: migrate-classic-to-hummingbird
description: Rebuild a Classic-derived theme using Hummingbird architecture with Webpack/Vite pipeline, BEM SCSS layers, vanilla TypeScript with data-ps-* bindings, and updated theme.yml asset entries. Use when a forked Classic theme needs to adopt the PS 9 Hummingbird conventions.
---

## Requirements

- Current theme name and parent (usually `classic`).
- Whether the theme has a custom asset pipeline or relies on Classic's built-in CSS/JS.
- List of custom templates, overrides, and JS components.
- Target bundler: Webpack (Hummingbird default) or Vite.

## Steps

1. Clone the Hummingbird scaffold as a reference:

    ```bash
    git clone --depth 1 https://github.com/PrestaShop/hummingbird.git /tmp/hummingbird-ref
    ```

2. Create the new asset pipeline in the theme root. For Webpack (Hummingbird default):

    ```bash
    cp /tmp/hummingbird-ref/webpack.config.js themes/mytheme/webpack.config.js
    cp /tmp/hummingbird-ref/package.json themes/mytheme/package.json
    ```

    For Vite, create a `vite.config.ts` with SCSS and TypeScript support.

3. Restructure assets into Hummingbird's layer pattern:

    ```
    assets/
    ├── scss/
    │   ├── _settings.scss        # design tokens (--ps-*, --bs-*)
    │   ├── _components.scss      # @forward all BEM component partials
    │   ├── components/
    │   │   ├── _product-card.scss
    │   │   └── _header.scss
    │   └── theme.scss            # main entry, imports layers in order
    └── js/
        ├── components/
        │   ├── ProductCard.ts
        │   └── Header.ts
        └── theme.ts              # main entry, auto-discovers [data-ps-component]
    ```

4. Convert CSS to BEM SCSS: one block per file, no descendant selectors deeper than element level, modifiers via `--modifier`, design tokens via custom properties. See `migrate-theme-css-to-bem-scss`.

5. Replace jQuery with vanilla TypeScript + `data-ps-*` bindings. See `migrate-theme-jquery-to-vanilla`. Wire components via `data-ps-component` attribute:

    ```typescript
    // assets/js/theme.ts
    import { initComponents } from './core/component-loader';
    document.addEventListener('DOMContentLoaded', () => initComponents());
    ```

6. Update `config/theme.yml` to declare the new compiled assets:

    ```yaml
    assets:
      css:
        all:
          - id: theme-main
            path: assets/css/theme.css
            priority: 100
      js:
        all:
          - id: theme-main
            path: assets/js/theme.js
            priority: 100
    ```

7. Remove old Classic asset includes (`<link>` / `<script>` tags in templates) that are now handled by the pipeline.

8. Build and verify:

    ```bash
    npm install --prefix themes/mytheme
    npm run build --prefix themes/mytheme
    bin/console prestashop:theme:enable mytheme
    ```

9. Visually verify all major page families (home, category, product, cart, checkout, account) render correctly.

## Do

- Keep all design tokens as CSS custom properties (`--ps-*` for theme, `--bs-*` for Bootstrap).
- Use `data-ps-component` / `data-ps-element` for JS bindings instead of CSS class selectors.
- Import Bootstrap via the SCSS entry point to share variables and mixins.

## Don't

- Don't keep jQuery as a dependency unless a third-party module absolutely requires it.
- Don't mix inline `<style>` or `<script>` tags with the bundled pipeline.
- Don't ship `node_modules/` or source `.ts`/`.scss` files in the distributable theme zip.

## Related skills

- `migrate-theme-jquery-to-vanilla` - systematic jQuery removal.
- `migrate-theme-css-to-bem-scss` - CSS to BEM conversion.
- `theme-asset-pipeline` - bundler configuration details.
- `theme-create-from-hummingbird` - starting fresh from Hummingbird.
