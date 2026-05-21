---
name: theme-bootstrap-compatibility
description: Pin a PrestaShop 9 theme to a single supported Bootstrap version and decide how (or whether) to ship the Bootstrap Compatibility Layer that bridges Bootstrap 4 modules into a Bootstrap 5 theme. Use when the user starts a new PS 9 theme, upgrades a Classic-derived theme to Hummingbird, or hits broken modals / dropdowns coming from a third-party module.
---

## Requirements

Ask the user:
* Which theme are we targeting:
  * A new PS 9 theme or a Hummingbird fork: target Bootstrap 5.3.x, no jQuery on the theme side.
  * The legacy `classic` theme on PS 9: stays on Bootstrap 4 for backward compatibility with the existing module ecosystem.
  * A custom theme already in production: read its `package.json` to confirm the current Bootstrap major.
* Which third-party modules the merchant ships, and whether any of them rely on Bootstrap 4 patterns (renamed utility classes like `.ml-*`, jQuery plugins like `$('#x').modal()`, or non-namespaced data attributes like `data-toggle`). Modal, tooltip, dropdown and popover are the components most likely to break.
* Whether the priority is "ship now with mixed-version content" (warrants the Compatibility Layer as a stopgap) or "migrate cleanly" (warrants updating the modules' markup).

## Steps

1. Pin the Bootstrap version explicitly. PrestaShop 9 themes target Bootstrap 5.3.x; the legacy Classic theme targets Bootstrap 4. Mixing utility classes from both majors silently breaks layout in unpredictable ways.

    ```jsonc
    // package.json (Hummingbird-style PS 9 theme)
    {
      "dependencies": {
        "bootstrap": "5.3.3",
        "@popperjs/core": "2.11.8"
      }
    }
    ```

    Lock to a specific patch version (no `^`, no `~`) until the theme has a CI matrix that catches Bootstrap regressions automatically.

2. Use `bootstrap.bundle.min.js` (which embeds Popper) instead of separate `bootstrap.js` + `popper.js` includes, and load it without jQuery. PS 9 storefront no longer ships jQuery for new themes; `core.js` keeps it loaded only for backward compatibility with legacy modules:

    ```js
    // assets/js/theme.js (or your bundler entry)
    import 'bootstrap/dist/js/bootstrap.bundle.min.js';
    ```

    Initialise components via the BS5 JS API (`new bootstrap.Modal(el)`, `new bootstrap.Tooltip(el)`), not via jQuery shortcuts (`$(el).modal()`).

3. Use Bootstrap 5 markup and data attributes everywhere in the theme. Common renames to watch for during a migration:

    | Bootstrap 4 (legacy) | Bootstrap 5.3 (PS 9 target) |
    |---|---|
    | `data-toggle="modal"` / `data-target="#m"` | `data-bs-toggle="modal"` / `data-bs-target="#m"` |
    | `.ml-2`, `.mr-3`, `.pl-1` | `.ms-2`, `.me-3`, `.ps-1` (start / end logical sides) |
    | `.float-left`, `.float-right` | `.float-start`, `.float-end` |
    | `.text-left`, `.text-right` | `.text-start`, `.text-end` |
    | `.sr-only` | `.visually-hidden` |
    | `.no-gutters` | `.g-0` |
    | `.form-row`, `.form-group` | grid (`.row` + `.col-*`), removed |
    | `.custom-select`, `.custom-control` | `.form-select`, `.form-check` |
    | `.badge-primary`, `.badge-pill` | `.badge.text-bg-primary`, `.rounded-pill` |
    | `.close` | `.btn-close` |

    The full breaking-change list lives in the [Bootstrap 5.3 migration guide](https://getbootstrap.com/docs/5.3/migration/).

4. If the merchant ships modules that still emit Bootstrap 4 markup (modals refusing to open, dropdowns inert), adopt the Bootstrap Compatibility Layer as a temporary shim. It is a standalone runtime helper, NOT part of Hummingbird; module developers (or the merchant) include it from their own module:

    ```php
    // From a module: hooks/setMedia.php
    public function hookActionFrontControllerSetMedia(): void
    {
        $this->context->controller->addCss('https://unpkg.com/bootstrap-compatibility-layer@1/dist/bootstrap-compatibility-layer.min.css');
        $this->context->controller->addJs('https://unpkg.com/bootstrap-compatibility-layer@1');
    }
    ```

    The layer covers the three biggest BS4 -> BS5 breakages: `data-bs-*` attribute namespacing, renamed utility classes, and jQuery plugin bridges (`.modal()`, `.tooltip()`, ...). Treat it as a stopgap: each module that depends on it should have a tracked migration to Bootstrap 5 markup.

5. For Hummingbird forks, do not rip out the SCSS variable overrides Hummingbird ships (custom-properties layer, `--bs-*` overrides). They are how the theme reaches its visual identity without recompiling Bootstrap. Override them in your own SCSS partial that imports last:

    ```scss
    // assets/scss/_variables.scss
    @use 'bootstrap/scss/functions' as *;
    @use 'bootstrap/scss/variables' as *;

    $primary: #1a73e8;
    $border-radius: 0.5rem;
    ```

6. Document the targeted Bootstrap version in `README.md` and in the theme's `composer.json` / `package.json` description so module authors know which markup the theme expects. Treat any Bootstrap-major bump as a breaking change for downstream modules.

7. Verify:
    * `npm ls bootstrap` shows the single pinned version, no duplicates.
    * Open the BO debug toolbar with Developer Mode on; the Symfony "Performance" panel lists every loaded JS asset. Confirm `bootstrap.bundle.min.js` loads exactly once.
    * Open a page that uses every interactive Bootstrap component the theme relies on (modal, dropdown, offcanvas, tooltip, accordion, carousel, form-select). All must function without console errors.

## Do

- Pin Bootstrap (and Popper) to a specific patch version. Bootstrap minor releases have introduced regressions that broke PrestaShop themes in the past.
- Use `bootstrap.bundle.min.js` and the BS5 JS API (no jQuery shortcut plugins).
- Migrate module markup to BS5 rather than living on the Compatibility Layer indefinitely. The layer is documented as a stopgap, not a long-term solution.
- Re-export SCSS / CSS custom properties (`--bs-primary`, `--bs-border-radius`, ...) from the theme so child themes can rebrand without rebuilding Bootstrap.

## Don't

- Don't mix `.ml-*` / `.ms-*`, `data-toggle` / `data-bs-toggle`, or BS4 / BS5 component markup in the same theme. The result silently looks broken on edge cases.
- Don't add jQuery to a new PS 9 theme just because a module needs it. If jQuery is unavoidable for a module, scope it to that module's bundle, not the theme.
- Don't load Bootstrap from a CDN in production while also bundling it locally. You will ship two majors and waste KB on every page.
- Don't bump the Classic theme to Bootstrap 5. Classic exists precisely to keep BS4 module compatibility on PS 9.

## Canonical examples

- [devdocs - Hummingbird Bootstrap compatibility](https://devdocs.prestashop-project.org/9/themes/hummingbird/bootstrap-compatibility/) - the layer rationale, what it shims (`data-bs-*` attributes, renamed classes, jQuery bridges), and the `addCss` / `addJs` registration pattern.
- [devdocs - Hummingbird CSS conventions](https://devdocs.prestashop-project.org/9/themes/hummingbird/css-conventions/) - how Hummingbird overrides Bootstrap variables and exposes its own custom-properties layer.
- [devdocs - Browser compatibility (themes)](https://devdocs.prestashop-project.org/9/themes/guidelines/browser-compatibility/) - PS 9 theme browser baseline, which feeds into Bootstrap version choice.
- [Bootstrap 5.3 migration guide](https://getbootstrap.com/docs/5.3/migration/) - definitive list of breaking changes from BS4 to BS5.
- [`PrestaShop/bootstrap-compatibility-layer`](https://github.com/PrestaShop/bootstrap-compatibility-layer) - source of the runtime shim, with installation and scope documentation.
- [`PrestaShop/hummingbird` - `package.json`](https://github.com/PrestaShop/hummingbird/blob/develop/package.json) - reference Bootstrap and Popper pinning for a PS 9 theme.

## Related skills

- `theme-create-from-hummingbird` - scaffolds a theme that already targets Bootstrap 5.3 with the right pipeline.
- `theme-asset-pipeline` - where to drop the Bootstrap import (`bootstrap.bundle.min.js`, SCSS) inside the theme bundler.
- `module-bump-compatibility` - module-side counterpart for upgrading a BS4 module to BS5 markup, removing the Compatibility Layer dependency.
