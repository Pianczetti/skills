---
name: theme-create-from-scratch
description: Scaffold a minimum viable PrestaShop 9 theme from an empty folder, with the strict directory layout, the required `config/theme.yml` keys and the activation flow. Use when the user wants full control over markup and assets and is OK starting blank instead of forking Hummingbird.
---

## Requirements

Ask the user:
* Theme technical name (lower_snake_case, MUST match the directory under `themes/`, e.g. `mytheme`).
* Display name shown in the Back Office (e.g. `My Theme`).
* Author name, optional email and URL.
* Version (`1.0.0` for a brand-new theme).
* Target PrestaShop version range (`9.0.0`, `9.1.0`, ...). The `meta.compatibility.from`/`to` keys gate activation.
* Whether the theme will register custom assets, image types or per-page layouts (drives the `theme.yml` payload).

If the user just wants the recommended starting point, redirect to `theme-create-from-hummingbird` instead. Building from scratch is mainly useful when you must control the full tree.

## Steps

1. Create the theme directory under your PrestaShop install at `themes/<name>/` and lay out the strict skeleton PrestaShop validates against:

    ```
    themes/mytheme/
    ├── config/
    │   └── theme.yml
    ├── assets/
    │   ├── css/theme.css
    │   └── js/theme.js
    ├── templates/
    │   ├── _partials/        # header, footer, breadcrumb, notifications fragments
    │   ├── catalog/          # product.tpl, listing/product-list.tpl
    │   ├── checkout/         # cart.tpl, cart-empty.tpl, checkout.tpl, order-confirmation.tpl
    │   ├── cms/              # page.tpl, category.tpl, sitemap.tpl, stores.tpl
    │   ├── customer/         # my-account.tpl, addresses.tpl, history.tpl, ...
    │   ├── errors/           # 404.tpl, forbidden.tpl
    │   ├── layouts/
    │   │   └── layout-full-width.tpl
    │   ├── contact.tpl
    │   └── index.tpl
    └── preview.png           # 500 x 746 PNG, displayed in the BO theme picker
    ```

    Template files can be empty - PrestaShop only requires that they exist for the theme to validate. Pages will render blank until you fill them in.

2. Write `config/theme.yml`. The mandatory top-level keys are `name`, `display_name`, `version`, `author`, `meta`, `global_settings` (image types are required) and `theme_settings` (must declare a `default_layout`):

    ```yaml
    name: mytheme              # MUST match the directory name
    display_name: My Theme
    version: 1.0.0
    author:
      name: "Your Name"
      email: "dev@example.com"
      url: "https://example.com"

    meta:
      compatibility:
        from: 9.0.0
        to: ~9.0
      available_layouts:
        layout-full-width:
          name: Full width
          description: Single column, no sidebars

    assets:
      use_parent_assets: false  # Only relevant for child themes

    global_settings:
      configuration:
        PS_QUICK_VIEW: false
      image_types:              # REQUIRED. Activation REPLACES the shop's image types.
        cart_default:    {width: 125, height: 125, scope: [products]}
        small_default:   {width: 98,  height: 98,  scope: [products, categories, manufacturers, suppliers]}
        medium_default:  {width: 452, height: 452, scope: [products, manufacturers, suppliers]}
        large_default:   {width: 800, height: 800, scope: [products, manufacturers, suppliers]}
        home_default:    {width: 250, height: 250, scope: [products]}
        category_default:{width: 180, height: 180, scope: [categories]}

    theme_settings:
      default_layout: layout-full-width
      layouts:                  # Per-page family overrides (optional)
        identity: layout-full-width
        order-confirmation: layout-full-width
    ```

3. Add a minimal layout at `templates/layouts/layout-full-width.tpl`. Layouts own the `<html>`, `<head>` and `<body>` shell. Page templates extend the layout via `{extends file=$layout}` and override its named blocks:

    ```smarty
    <!doctype html>
    <html lang="{$language.iso_code}">
    <head>
      {block name='head'}{include file='_partials/head.tpl'}{/block}
    </head>
    <body id="{$page.page_name}" class="{$page.body_classes|classnames}">
      {hook h='displayAfterBodyOpeningTag'}
      <header>{block name='header'}{include file='_partials/header.tpl'}{/block}</header>
      <main>{block name='content'}{/block}</main>
      <footer>{block name='footer'}{include file='_partials/footer.tpl'}{/block}</footer>
      {hook h='displayBeforeBodyClosingTag'}
    </body>
    </html>
    ```

4. Activate the theme. Two paths:

    * Back Office (preferred): zip the theme root and upload via Design > Theme & Logo > Add new theme, then click Use this theme. See `theme-package-zip` for the export rules.
    * CLI: `bin/console prestashop:theme:enable <name>` from the PrestaShop root.

    Activation REPLACES the shop's image types with whatever you declared under `global_settings.image_types`. Audit that block before activating on a live store.

## Do

- Keep the directory `name` and the `name:` key in `theme.yml` identical and lower_snake_case. PrestaShop refuses to load mismatched themes.
- Declare every image preset your templates reference under `global_settings.image_types`. Missing presets render broken images sitewide.
- Use `theme_settings.default_layout` plus per-page entries in `theme_settings.layouts` to wire layouts to pages instead of hard-coding `{extends}` paths.

## Don't

- Don't ship empty `image_types`. Activation REPLACES the existing presets - dropping any will break the shop's media.
- Don't omit any of the required directories under `templates/`. Validation requires them even if the files are empty.
- Don't edit compiled assets directly when you start using a bundler. Edit sources, rebuild, ship the rebuilt `assets/`.

## Canonical examples

- [devdocs - Creating a theme from scratch](https://devdocs.prestashop-project.org/9/themes/create-a-theme/from-scratch/) - the source for the minimum file tree and the `theme.yml` schema used here.
- [devdocs - Theme structure](https://devdocs.prestashop-project.org/9/themes/concepts/theme-structure/) - full `theme.yml` reference (compatibility, layouts, configuration, modules, hooks, image_types, dependencies).
- [devdocs - Quick start](https://devdocs.prestashop-project.org/9/themes/getting-started/quick-start/) - Back Office activation flow.
- [devdocs - `prestashop:theme:enable`](https://devdocs.prestashop-project.org/9/development/components/console/prestashop-theme-enable/) - the CLI activation command.
- [`PrestaShop/classic-theme` - `config/theme.yml`](https://github.com/PrestaShop/classic-theme/blob/develop/config/theme.yml) - real-world `theme.yml` shipped with PrestaShop, including layouts and hook routing.
- [`PrestaShop/classic-theme` - `templates/layouts/layout-full-width.tpl`](https://github.com/PrestaShop/classic-theme/blob/develop/templates/layouts/layout-full-width.tpl) - canonical full-width layout extending `layout-both-columns.tpl`.

## Related skills

- `theme-create-from-hummingbird` - the recommended starting point for most new themes.
- `theme-create-child-theme` - extend an existing theme instead of starting over.
- `theme-add-layout` - add additional layouts after the theme is live.
- `theme-override-template` - override a single front-office Smarty template.
