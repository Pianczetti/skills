---
name: theme-add-layout
description: Add a new page layout to a PrestaShop 9 theme - the `.tpl` shell, the `meta.available_layouts` declaration and the per-page assignment in `theme_settings`. Use when the user wants a new column arrangement, a checkout-specific shell or a marketing landing layout.
---

## Requirements

Ask the user:
* Layout technical name. By convention it starts with `layout-` (e.g. `layout-checkout`, `layout-landing`). The file under `templates/layouts/` MUST be named exactly the same.
* The structural intent: full-width, one column, two columns (left/right), three columns, content-only, error.
* Which page family should default to it (`product`, `category`, `cms`, `checkout`, `order-confirmation`, `index`, ...). The full list maps to the `php_self` of each front controller.
* Whether merchants should be able to switch the assignment from Design > Theme & Logo > Choose layouts (always yes - the BO writes the override to `settings_<shopId>.yml`).

## Steps

1. Create the layout template at `templates/layouts/<layout-name>.tpl`. Two patterns:

    * **Extend an existing layout** to reuse most of the shell and only redefine the columns or the wrapper:

        ```smarty
        {* templates/layouts/layout-landing.tpl *}
        {extends file='layouts/layout-full-width.tpl'}

        {block name='content_wrapper'}
          <div id="content-wrapper" class="container-fluid landing">
            {block name='content'}{/block}
          </div>
        {/block}

        {block name='header'}{/block}    {* hide the default header *}
        {block name='footer'}{include file='_partials/footer-minimal.tpl'}{/block}
        ```

    * **Compose a fresh shell** when you need full control of `<html>`, `<head>` and `<body>`. The layout owns the document and exposes named `{block}` regions for page templates to fill via `{extends file=$layout}`:

        ```smarty
        <!doctype html>
        <html lang="{$language.iso_code}">
        <head>
          {block name='head'}{include file='_partials/head.tpl'}{/block}
        </head>
        <body id="{$page.page_name}" class="{$page.body_classes|classnames}">
          {hook h='displayAfterBodyOpeningTag'}
          <main>
            {block name='content_wrapper'}
              <div id="content-wrapper">
                {block name='content'}{/block}
              </div>
            {/block}
          </main>
          {hook h='displayBeforeBodyClosingTag'}
          {block name='javascript_bottom'}{include file='_partials/javascript.tpl' javascript=$javascript.bottom}{/block}
        </body>
        </html>
        ```

    Define as many `{block}` regions as you can: child themes and overrides extend layouts the same way page templates do.

2. Declare the layout in `config/theme.yml` under `meta.available_layouts`. The key MUST equal the file's basename without `.tpl`. The `name` and `description` are surfaced in the BO layout picker:

    ```yaml
    meta:
      available_layouts:
        layout-full-width:
          name: Full width
          description: No side columns
        layout-landing:
          name: Landing page
          description: Marketing pages with no header and a minimal footer
    ```

3. Wire the layout to pages. `theme_settings.default_layout` is the shop-wide fallback; `theme_settings.layouts` overrides it per page family. The page key matches the controller's `php_self`:

    ```yaml
    theme_settings:
      default_layout: layout-full-width
      layouts:
        index: layout-landing
        cms: layout-full-width
        order-confirmation: layout-full-width
    ```

    Once the theme is reactivated (or reinstalled), merchants can override these per-page in Design > Theme & Logo > Choose layouts. The BO writes overrides to `themes/<name>/config/settings_<shopId>.yml` and leaves `theme.yml` untouched.

4. Page templates pick up the layout automatically - they extend the special `$layout` variable that the front controller resolves from `theme_settings.layouts`:

    ```smarty
    {* templates/cms/page.tpl *}
    {extends file=$layout}

    {block name='content'}
      {* page-specific markup *}
    {/block}
    ```

    To force a specific layout from a single template (rare, prefer `theme_settings`), extend it directly:

    ```smarty
    {extends file='layouts/layout-landing.tpl'}
    ```

5. Verify. Run `bin/console prestashop:theme:enable <name>` to reactivate (or use the BO), open the page, view-source. Developer Mode wraps templates with HTML comments showing which layout file rendered:

    ```html
    <!-- begin /var/www/html/themes/<name>/templates/layouts/layout-landing.tpl -->
    ```

## Do

- Name files `layout-<purpose>.tpl` and use the same key in `meta.available_layouts`. PrestaShop matches by filename basename - mismatched names silently disappear from the BO picker.
- Extend an existing layout instead of duplicating its `<head>`, asset hooks (`displayAfterBodyOpeningTag`, `displayBeforeBodyClosingTag`) and `_partials/javascript.tpl` includes. Forgetting any of those breaks core JS, modules and CSS loading.
- Define rich `{block}` regions (`head`, `header`, `breadcrumb`, `content_wrapper`, `content`, `footer`, `javascript_bottom`). Child themes and per-template overrides depend on them.
- Use `theme_settings.layouts` to bind layouts to page families. Per-template `{extends file='layouts/...'}` is a hard override and bypasses the BO layout picker.

## Don't

- Don't drop `{hook h='displayAfterBodyOpeningTag'}` and `{hook h='displayBeforeBodyClosingTag'}` from a fresh layout shell. Many modules (cookie banners, GTM, accessibility helpers) depend on them.
- Don't omit `{block name='javascript_bottom'}{include file='_partials/javascript.tpl' javascript=$javascript.bottom}{/block}`. Without it, all bottom-position scripts registered via `registerJavascript` go missing.
- Don't define a layout key in `available_layouts` without shipping the matching `.tpl` file. PrestaShop will error when a page is assigned to it.
- Don't edit `settings_<shopId>.yml` by hand. It is maintained by the BO layout picker.

## Canonical examples

- [devdocs - Templates and layouts](https://devdocs.prestashop-project.org/9/themes/concepts/templates/templates-and-layouts/) - source for the layout vs template split, the `$layout` variable and the resolution chain.
- [devdocs - Theme structure](https://devdocs.prestashop-project.org/9/themes/concepts/theme-structure/) - full `meta.available_layouts` and `theme_settings.{default_layout,layouts}` reference.
- [devdocs - Template inheritance](https://devdocs.prestashop-project.org/9/themes/concepts/templates/template-inheritance/) - `{extends}`, `{block}`, `prepend`/`append`.
- [`PrestaShop/classic-theme` - `config/theme.yml`](https://github.com/PrestaShop/classic-theme/blob/develop/config/theme.yml) - `available_layouts` declaring four layouts (full-width, both-columns, left-column, right-column).
- [`PrestaShop/classic-theme` - `templates/layouts/layout-full-width.tpl`](https://github.com/PrestaShop/classic-theme/blob/develop/templates/layouts/layout-full-width.tpl) - layout extending `layout-both-columns.tpl` and emptying the `left_column`/`right_column` blocks.

## Related skills

- `theme-create-from-scratch` - bootstrap the theme that will own the new layout.
- `theme-create-child-theme` - add a layout from a child theme without forking.
- `theme-override-template` - override a single page template instead of adding a layout.
