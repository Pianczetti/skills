---
name: theme-override-template
description: Override a single template from your active theme - a front office Smarty `.tpl` or a back office Symfony Twig view shipped by a module. Use whenever the user wants to change rendered markup without forking the upstream theme or core.
---

## Requirements

Ask the user:
* Which template to override and where it currently lives:
  * A theme template (`themes/<parent>/templates/<path>.tpl`).
  * A module front-office template (`modules/<modulename>/views/templates/front/...tpl` or `hook/...tpl`).
  * A back-office Twig view (`@PrestaShop/Admin/...html.twig` shipped by the core or by a module-rendered admin form).
* Whether they want to replace the file outright or only override one or two `{block}` regions.
* Whether the override belongs in the active theme (correct for theme-level overrides) or in a module (correct for module-rendered admin views).

Run with Developer Mode enabled (`_PS_MODE_DEV_ = true`) - PrestaShop emits HTML comments showing the source path of every rendered template, which is the fastest way to confirm your override is being picked up.

## Steps

### Front office Smarty templates

1. Locate the source. Most front-office templates live under `themes/<parent_theme>/templates/<path>.tpl`. Module front templates live under `modules/<modulename>/views/templates/{front,hook}/...tpl`.

2. Re-create the same path inside the active theme. PrestaShop's resolver always prefers the active theme's path over the parent theme's or the module's:

    ```
    themes/<your_theme>/templates/<path>.tpl                                        # theme template override
    themes/<your_theme>/modules/<modulename>/views/templates/front/<file>.tpl       # module template override
    themes/<your_theme>/modules/<modulename>/views/templates/hook/<file>.tpl
    ```

3. Pick a block strategy. Replacing the file outright is the simplest path; extending and overriding `{block}` regions keeps the diff small and survives parent updates better. To override only specific blocks of a template that lives in the parent theme, extend it with the `parent:` Smarty resource prefix (without it `{extends file='...'}` resolves back to the child and loops):

    ```smarty
    {extends file='parent:catalog/listing/category.tpl'}

    {block name='product_list_header'}
      <div class="custom-header">{$listing.label}</div>
    {/block}
    ```

    Use `{block name='foo' append}` / `{block name='foo' prepend}` (or `{$smarty.block.parent}`) to keep the parent's content and add to it.

4. Module template overrides have one trap: includes resolved with a relative path (`{include file='./_partials/card.tpl'}`) load from the module directory and ignore your override of the partial. Includes prefixed with `module:` (`{include file='module:ps_featuredproducts/views/templates/front/_partials/card.tpl'}`) go through the override chain and pick your file up. If the module uses relative includes you must override every included partial alongside the main template.

5. Verify. With Developer Mode on, view-source on the rendered page; PrestaShop wraps each template with HTML comments:

    ```html
    <!-- begin /var/www/html/themes/<your_theme>/templates/catalog/listing/category.tpl -->
    ...
    <!-- end /var/www/html/themes/<your_theme>/templates/catalog/listing/category.tpl -->
    ```

### Back office Twig views (Symfony admin pages)

1. Identify the template. Open the admin page in Developer Mode, click the Symfony Debug toolbar, then "Twig metrics". The list shows every Twig template rendered for the page (e.g. `@PrestaShop/Admin/Product/CatalogPage/catalog.html.twig`).

2. Re-create the same path inside a module's `views/PrestaShop/...` tree. Back office Twig overrides are owned by modules, NOT by the active theme - the BO Twig loader maps `modules/<modulename>/views/PrestaShop/Admin/...` over the core templates:

    ```
    modules/<modulename>/views/PrestaShop/Admin/Product/CatalogPage/catalog.html.twig
    modules/<modulename>/views/PrestaShop/Admin/Product/CatalogPage/Lists/list.html.twig
    ```

3. Extend the original namespace with the override prefix. Use `@!PrestaShop` (or, on PS 8.0+, `@PrestaShopCore`) so Twig knows you mean the core file, not your override:

    ```twig
    {% extends '@!PrestaShop/Admin/Product/CatalogPage/catalog.html.twig' %}

    {% block product_catalog_filters %}
      {# your custom markup #}
    {% endblock %}
    ```

    Without the `!`, `{% extends '@PrestaShop/...' %}` resolves to your override and infinite-loops.

4. Override Twig views ship with the module - register the override module as a regular module (`bin/console prestashop:module install <modulename>`). Twig caches admin views aggressively; clear the cache after each change with `bin/console cache:clear --env=dev` (or `--env=prod` on a live site).

## Do

- Always work in Developer Mode while building an override (Advanced Parameters > Performance > Force compilation: Yes; Cache: No). The HTML comments and Twig metrics tell you which file actually rendered.
- Use `{extends file='parent:...'}` + block overrides for theme templates and `{% extends '@!PrestaShop/...' %}` for back-office Twig. They keep your override surface tiny and make parent upgrades safer.
- Override module templates via the active theme's `modules/<modulename>/...` directory. Override admin Twig templates via a module's `views/PrestaShop/...` directory. They are different mechanisms with different ownership.
- Document every override in your theme/module README so future maintainers can spot conflicts when two extensions override the same file.

## Don't

- Don't edit `vendor/`, the core source tree, or the parent theme directly. Updates and `composer install` will wipe your changes.
- Don't override a back-office Twig template from a child theme. The BO Twig loader does not look inside `themes/`; only modules can override admin views.
- Don't forget the `parent:` (Smarty) or `@!PrestaShop` / `@PrestaShopCore` (Twig) prefix when extending a same-named template - omitting it self-references the override and infinite-loops.
- Don't override a module template just to drop it. Use `unregisterStylesheet` / `unregisterJavascript` from a hook (or an empty asset override file) instead.

## Canonical examples

- [devdocs - Theme overrides](https://devdocs.prestashop-project.org/9/themes/concepts/overrides/) - module template/asset override resolution from the active theme, including the `module:` include caveat.
- [devdocs - Template inheritance](https://devdocs.prestashop-project.org/9/themes/concepts/templates/template-inheritance/) - `{extends}`, `{block}`, `prepend`/`append`, `{$smarty.block.parent}`, `parent:` prefix.
- [devdocs - How to override Back Office views](https://devdocs.prestashop-project.org/9/modules/concepts/templating/admin-views/) - module-owned Twig overrides and the `@!PrestaShop` / `@PrestaShopCore` namespaces.
- [devdocs - Templates and layouts](https://devdocs.prestashop-project.org/9/themes/concepts/templates/templates-and-layouts/) - locale and entity-ID resolution chain (`en-US/catalog/product-3.tpl` -> `catalog/product.tpl`).
- [`PrestaShop/classic-theme`](https://github.com/PrestaShop/classic-theme) - reference Smarty templates to override.
- [`PrestaShop/hummingbird`](https://github.com/PrestaShop/hummingbird) - reference Smarty templates for PS 9.

## Related skills

- `theme-create-child-theme` - the preferred container for front-office template overrides.
- `theme-add-layout` - register a brand-new layout instead of overriding a template.
- `module-add-config-page-modern` - the modern admin pages whose Twig views you might override.
