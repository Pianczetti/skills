---
name: migrate-classic-to-hummingbird
description: Rebuild a Classic-derived theme using Hummingbird architecture (Bootstrap 5, BEM SCSS, TypeScript, data-ps-* bindings, no jQuery). Use when a shop runs a customized Classic theme and must move to the PS 9 Hummingbird-based stack.
---

## Requirements

- Identify the current theme directory and customization scope (templates, CSS, JS, assets).
- Confirm target PS version (9.x) and Node version (20.x for Hummingbird toolchain).
- List pages with heavy custom markup that cannot be auto-migrated.
- Decide: child theme of Hummingbird (less work, limited customization) vs full fork (more control). See `theme-create-child-theme` for the child approach.

## Steps

1. **Audit the Classic-based theme:**

    ```bash
    # Count custom templates vs stock Classic
    diff -rq themes/mytheme/templates themes/classic/templates | grep 'differ\|Only in mytheme'
    # Count JS files and jQuery usage
    grep -rn '\$(' themes/mytheme/assets/js/ | wc -l
    # Identify Smarty variables and hooks used
    grep -rn '{hook' themes/mytheme/templates/
    ```

2. **Clone Hummingbird** as the new base (see `theme-create-from-hummingbird`):

    ```bash
    cd themes/
    git clone https://github.com/PrestaShop/hummingbird.git mytheme-v2
    cd mytheme-v2 && rm -rf .git && git init
    ```

3. **Map Classic templates to Hummingbird equivalents.** Key structural differences:

    | Classic | Hummingbird |
    |---------|-------------|
    | `templates/layout/layout-both-columns.tpl` | `templates/layouts/layout-both-columns.tpl` |
    | `assets/css/theme.css` (flat) | `src/scss/` (BEM partials, compiled) |
    | `assets/js/theme.js` (jQuery) | `src/js/` (TypeScript modules, no jQuery) |
    | Bootstrap 4 classes | Bootstrap 5 classes |
    | `{hook h='displayNav'}` | Same hooks, but markup uses `data-ps-*` |

4. **Migrate custom templates.** For each customized `.tpl`:
   - Copy to the equivalent Hummingbird path.
   - Replace Bootstrap 4 classes with Bootstrap 5 equivalents (`ml-` to `ms-`, `mr-` to `me-`, `float-left` to `float-start`, etc.).
   - Replace jQuery-dependent markup with `data-ps-*` attributes.
   - Use Hummingbird's partial structure (one block per file).

5. **Migrate CSS to BEM SCSS** (see `migrate-theme-css-to-bem-scss`):
   - Create one `_block-name.scss` partial per visual component.
   - Use CSS custom properties for design tokens.
   - Import partials in `src/scss/main.scss`.

6. **Migrate JS to TypeScript** (see `migrate-theme-jquery-to-vanilla`):
   - Replace `$.ajax` with `fetch`.
   - Replace `$(selector).on(event)` with `addEventListener` via `data-ps-*` bindings.
   - Remove jQuery from `package.json`.

7. **Build and validate:**

    ```bash
    npm ci
    npm run build
    npm run lint
    ```

8. **Activate and smoke-test:**

    ```bash
    bin/console prestashop:theme:enable mytheme-v2
    ```

    Verify: homepage, category, product page, cart, checkout, my-account. Check responsive at 320px, 768px, 1200px.

## Do

- Start from a fresh Hummingbird clone and bring customizations in; do not try to upgrade Classic in-place.
- Use `data-ps-*` attributes for JS hooks; they are the stable contract between markup and behaviour.
- Run `npm run lint` frequently; Hummingbird ships ESLint + Stylelint configs.

## Don't

- Don't keep jQuery as a dependency; Hummingbird is jQuery-free by design.
- Don't edit files in `assets/`; edit `src/` and rebuild.
- Don't copy Classic's flat CSS directly; refactor into BEM partials.
- Don't remove accessibility primitives (ARIA, focus management, skip links) from Hummingbird templates.

## Canonical examples

- [devdocs - Hummingbird](https://devdocs.prestashop-project.org/9/themes/hummingbird/)
- [devdocs - Starting from Hummingbird](https://devdocs.prestashop-project.org/9/themes/create-a-theme/from-hummingbird/)

## Related skills

- `theme-create-from-hummingbird` - full setup of a Hummingbird-based theme.
- `theme-create-child-theme` - lighter approach when customizations are minimal.
- `migrate-theme-jquery-to-vanilla` - JS migration details.
- `migrate-theme-css-to-bem-scss` - CSS migration details.
- `theme-asset-pipeline` - build tooling configuration.
