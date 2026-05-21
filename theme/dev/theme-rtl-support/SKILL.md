---
name: theme-rtl-support
description: Add Right-to-Left language support to a PrestaShop 9 theme - rely on PrestaShop's automatic `*_rtl.css` discovery, ship a `rtl.css` override, fine-tune with `.rtlfix` files, and prefer logical CSS properties so layouts mirror without per-rule overrides. Use when the theme must serve Arabic, Hebrew, Persian, Urdu or any language whose `is_rtl` flag is on.
---

## Requirements

Ask the user:
* Which RTL languages must be supported (Arabic, Hebrew, Persian, Urdu, ...). Each needs to be installed in International > Localization > Languages with the `is_rtl` flag enabled (default for those packs).
* Whether the theme uses plain CSS files in `assets/css/` (Classic style) or compiles them from a SCSS / PostCSS pipeline (Hummingbird style). The two approaches differ in how RTL stylesheets are produced.
* Whether layouts should be mirrored automatically via logical CSS properties (preferred) or via dedicated RTL overrides (legacy approach).

## Steps

1. Understand the runtime contract. When the active language has `is_rtl=1`, PrestaShop:
    * Adds the `lang-rtl` class on `<body>` so any `body.lang-rtl ...` selector activates.
    * Loads `assets/css/rtl.css` automatically with ID `theme-rtl` and priority `900` (right after the main stylesheets, before `custom.css`). Use it for theme-wide RTL overrides.
    * For every registered stylesheet (`theme.css`, `product.css`, ...), looks for a `*_rtl.css` sibling and loads it instead of (or after) the original. The `needRtl: false` option on a stylesheet entry opts out.

    No `assets.rtl_css` key exists in `theme.yml`; the discovery is purely filename-based.

2. For plain-CSS themes, generate RTL siblings from the Back Office. The BO walks `assets/css/`, runs each file through an RTL converter, and writes `<name>_rtl.css` next to it:

    1. Install at least one RTL language (Arabic is the canonical fixture).
    2. Design > Theme & Logo > "Adaptation to right-to-left languages" (only visible when an RTL language is installed).
    3. Pick the theme, toggle "Generate RTL stylesheet" to Yes, save.

    The generator never overwrites an existing `*_rtl.css`. To regenerate, delete the file first.

3. For SCSS / PostCSS pipelines (Hummingbird and any modern theme), generate the RTL bundle as a build step rather than letting the BO do it. Two well-supported plugins:

    ```bash
    # Option A: PostCSS pipeline
    npm install --save-dev postcss-rtlcss

    # Option B: standalone CLI
    npm install --save-dev rtlcss
    ```

    Wire it into the build so `npm run build` produces both `assets/css/theme.css` and `assets/css/theme_rtl.css`:

    ```js
    // postcss.config.js
    module.exports = {
      plugins: [
        require('postcss-rtlcss')({ mode: 'override', source: 'ltr' }),
        // ...
      ],
    };
    ```

    Commit the generated `*_rtl.css` files alongside the LTR ones; PrestaShop ships from compiled assets, not from sources.

4. Fine-tune the auto-generated output with `.rtlfix` files when the converter mishandles a rule. Do NOT edit the generated `*_rtl.css` directly - the next regeneration will wipe your changes. Create a sibling `<name>.rtlfix` (e.g. `assets/css/theme.rtlfix`); PrestaShop appends its contents to `<name>_rtl.css` after every regeneration:

    ```css
    /* assets/css/theme.rtlfix */
    .product-thumbnails {
      transform: scaleX(-1); /* keep arrows mirrored, the converter flipped them twice */
    }
    ```

5. Prefer CSS logical properties for new code so most rules mirror without an override. Modern browsers (PS 9 baseline) support them universally:

    | Physical (avoid) | Logical (prefer) |
    |---|---|
    | `margin-left` | `margin-inline-start` |
    | `margin-right` | `margin-inline-end` |
    | `padding-top` | `padding-block-start` |
    | `text-align: left` | `text-align: start` |
    | `left: 0` | `inset-inline-start: 0` |
    | `border-radius: 0 4px 4px 0` | `border-start-end-radius: 4px; border-end-end-radius: 4px` |

    Logical properties make the LTR and RTL bundles structurally identical for everything except icons, decorative images and free-form transforms.

6. Mirror non-CSS assets that have a direction (chevrons, breadcrumb arrows, decorative ribbons). Either ship a separate RTL-flipped image and switch via `body.lang-rtl .my-arrow { background-image: url('arrow-rtl.svg'); }`, or apply `transform: scaleX(-1)` from `rtl.css`.

7. Verify:
    * Switch the front office to Arabic (or any RTL language). The `<body>` tag should carry the `lang-rtl` class and `dir="rtl"`.
    * View-source: confirm `theme.css`, `theme_rtl.css` and `rtl.css` are all loaded.
    * Walk every page family (home, category, product, cart, checkout, my account). Pay special attention to: navigation chevrons, breadcrumb separators, form alignment, product image carousels, cart quantity controls, modals.
    * Run a Lighthouse / axe pass with the RTL language active to catch contrast and keyboard-focus regressions introduced by the mirroring.

## Do

- Rely on PrestaShop's `*_rtl.css` auto-discovery and the `lang-rtl` body class instead of reinventing language switches.
- Use logical CSS properties for new code; keep `rtl.css` for the few rules that genuinely need a directional override.
- Use `.rtlfix` for hand-tuned corrections so regenerations preserve them.
- Test with the BO running in an RTL language too. Some admin embeds (the BO module preview, the live search) inherit `dir="rtl"` from the storefront and break in subtle ways.

## Don't

- Don't add an `assets.rtl_css` key in `config/theme.yml`. PrestaShop does not read it; the discovery is filename-based.
- Don't hand-edit generated `*_rtl.css` files. They are overwritten on every regeneration and on every fresh build. Use `.rtlfix` (BO generation) or commit the source change in your SCSS / PostCSS pipeline.
- Don't try to mirror with `direction: rtl` alone. Layout, padding, icons, animations and JavaScript-driven scroll positions all have direction baked in.
- Don't ship only `rtl.css` and call it done. PrestaShop loads `*_rtl.css` siblings automatically for every registered stylesheet; the granular per-file siblings are what give clean overrides for product, listing, checkout, etc.

## Canonical examples

- [devdocs - RTL support](https://devdocs.prestashop-project.org/9/themes/concepts/rtl/) - the `lang-rtl` body class, `rtl.css` priority, the `_rtl` suffix discovery, BO auto-generation and `.rtlfix` fine-tuning.
- [devdocs - Asset management](https://devdocs.prestashop-project.org/9/themes/concepts/asset-management/) - `needRtl` option on every registered stylesheet entry, asset priority chart.
- [devdocs - Theme structure](https://devdocs.prestashop-project.org/9/themes/concepts/theme-structure/) - the `assets:` block in `theme.yml` and how stylesheets resolve.
- [`MohammadYounes/rtlcss`](https://github.com/MohammadYounes/rtlcss) and [`rtlcss.com`](https://rtlcss.com/) - reference RTL CSS converter, used by the BO generator under the hood.
- [`elchininet/postcss-rtlcss`](https://github.com/elchininet/postcss-rtlcss) - PostCSS plugin to bake RTL output into the same build that produces the LTR bundle.

## Related skills

- `theme-asset-pipeline` - where to insert `postcss-rtlcss` (or an `rtlcss` step) inside the Vite / webpack pipeline.
- `theme-create-from-hummingbird` - Hummingbird's PostCSS config is the canonical RTL-ready starting point.
- `theme-accessibility-audit` - mirror tests should run as part of the audit, since RTL changes focus order and reading direction.
