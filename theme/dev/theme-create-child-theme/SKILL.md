---
name: theme-create-child-theme
description: Create a child theme that inherits everything from a parent (usually `classic` or `hummingbird`) and only ships the overrides. Use this when the user wants small tweaks (a few templates, a stylesheet, a script) and wants to keep pulling parent updates.
---

## Requirements

Ask the user:
* Parent theme name. The parent MUST be present in `themes/` before the child can be activated. Common parents: `hummingbird` (recommended for PS 9), `classic`.
* Child theme name (lower_snake_case, MUST match the directory).
* What needs to change: which templates, which assets, whether the child reuses the parent's compiled assets or ships its own.
* Whether the parent ships compatible versioning - read the parent's `meta.compatibility` block first.

If the user wants deep markup or build-tooling changes, redirect to `theme-create-from-hummingbird` (fork) instead. Child themes are the right answer when overrides fit in a handful of files.

## Steps

1. Create a minimal child layout. A child only needs `config/theme.yml` and `preview.png`; everything else falls back to the parent through PrestaShop's template inheritance:

    ```
    themes/my-child-theme/
    ├── config/
    │   └── theme.yml
    └── preview.png            # 500 x 746 PNG
    ```

2. Write `config/theme.yml`. The `parent:` key is what wires inheritance. Match `name:` to the directory exactly:

    ```yaml
    parent: hummingbird           # MUST match an installed parent theme name
    name: my-child-theme          # MUST match the child directory
    display_name: My Child Theme
    version: 1.0.0
    author:
      name: "Your Name"

    assets:
      use_parent_assets: true     # Load parent's CSS/JS first, then append child's
    ```

    With `use_parent_assets: true` the parent's compiled `theme.css` and `theme.js` are loaded first, then any extra assets the child registers are appended. With `false` the child's `assets/theme.css` and `assets/theme.js` replace the parent's entirely (use when you want full control and ship your own bundle).

3. Add or override assets. PrestaShop auto-loads `assets/theme.css` and `assets/theme.js` from the active theme - drop them into the child to replace the parent's. To register additional files, declare them in `theme.yml` under `global_settings.assets`:

    ```
    themes/my-child-theme/
    └── assets/
        ├── css/custom.css
        └── js/custom.js
    ```

    ```yaml
    global_settings:
      assets:
        css:
          custom:
            path: assets/css/custom.css
            media: all
        js:
          custom:
            path: assets/js/custom.js
            position: bottom
    ```

4. Override a template by re-creating it at the same path under `templates/`. Two strategies:

    * **Replace from a lower-level base.** When you want to keep the parent's listing logic but rewrite the surrounding template, extend a generic base instead of duplicating:

        ```smarty
        {* themes/my-child-theme/templates/catalog/listing/category.tpl *}
        {extends file='catalog/listing/product-list.tpl'}

        {block name='product_list_header'}
          {include file='catalog/_partials/category-header.tpl' listing=$listing category=$category}
        {/block}
        ```

    * **Override specific blocks.** When you only need to tweak one or two blocks of the parent's same-named template, use the `parent:` Smarty resource prefix to extend the parent version (without it `{extends file='catalog/listing/category.tpl'}` would point at the child itself and infinite-loop):

        ```smarty
        {extends file='parent:catalog/listing/category.tpl'}

        {block name='product_list_header'}
          {include file='catalog/_partials/category-header.tpl' listing=$listing category=$category}
        {/block}
        ```

    Use `{block name='foo' append}` / `{block name='foo' prepend}` (or `{$smarty.block.parent}`) to keep the parent block content and add to it instead of replacing it.

5. Override a module template the same way. Mirror the module path under the child's `modules/` directory:

    ```
    themes/my-child-theme/modules/ps_featuredproducts/views/templates/front/ps_featuredproducts.tpl
    ```

    See `theme-override-template` for the full resolution chain (theme module override -> parent theme module override -> module's own file).

6. Activate the child:

    ```bash
    bin/console prestashop:theme:enable my-child-theme
    ```

    Or via Back Office at Design > Theme & Logo. The parent theme MUST be installed first (its directory must be present under `themes/`); activation fails otherwise.

## Do

- Prefer a child theme over forking when you only need small tweaks. It keeps parent updates one `git merge` away and avoids the maintenance cost of carrying a full theme.
- Use `{extends file='parent:...'}` plus block overrides when you change one section of an existing template. Lower diff vs. parent = easier upgrades.
- Use `{block name='foo' append}` / `{block name='foo' prepend}` when you only need to add to a parent block, not replace it.
- Define as many `{block}` regions as possible in your own templates so future child themes have granular extension points without duplicating large sections.

## Don't

- Don't duplicate the entire parent template tree into the child "just in case". Every duplicated file is a maintenance liability when the parent changes.
- Don't override a template only to change CSS - register a stylesheet under `global_settings.assets.css` instead.
- Don't ship a child without `parent:` in `theme.yml`. PrestaShop will treat it as a standalone theme and skip inheritance.
- Don't forget to activate the parent first. Without it, the child's templates resolve to nothing.

## Canonical examples

- [devdocs - Creating a child theme](https://devdocs.prestashop-project.org/9/themes/create-a-theme/child-theme/) - the source for the minimal `theme.yml` and the `parent:` block override pattern.
- [devdocs - Template inheritance](https://devdocs.prestashop-project.org/9/themes/concepts/templates/template-inheritance/) - `{extends}`, `{block}`, `{block ... prepend|append}`, `{$smarty.block.parent}`, `{$smarty.block.child}`.
- [devdocs - Theme structure](https://devdocs.prestashop-project.org/9/themes/concepts/theme-structure/#parentchild-settings) - `parent:` key, `assets.use_parent_assets` semantics and the `child_theme_assets` URL helpers.
- [devdocs - Asset management](https://devdocs.prestashop-project.org/9/themes/concepts/asset-management/) - `global_settings.assets` registration syntax for css/js per page.
- [devdocs - Theme overrides](https://devdocs.prestashop-project.org/9/themes/concepts/overrides/) - module template overrides under `themes/<name>/modules/...`.
- [`PrestaShop/hummingbird`](https://github.com/PrestaShop/hummingbird) - canonical PS 9 parent theme.
- [`PrestaShop/classic-theme`](https://github.com/PrestaShop/classic-theme) - the historical reference parent theme; still maintained.

## Related skills

- `theme-create-from-hummingbird` - fork instead, when overrides exceed a handful of files.
- `theme-override-template` - the override mechanics for a single template (front office Smarty and back office Twig).
- `theme-add-layout` - layouts registered by the child apply on top of the parent's.
- `theme-create-from-scratch` - skip inheritance entirely.
