---
name: migrate-theme-smarty-deprecated
description: Replace deprecated Smarty helpers, removed template variables, and legacy markup patterns with their PS 9 equivalents. Use when a theme triggers Smarty deprecation warnings or references variables removed in PrestaShop 9.
---

## Requirements

- Theme name and PS version being migrated to (9.0, 9.1).
- List of deprecation warnings from the debug toolbar or error log.
- Access to the "Changes in 9.x" devdocs page for the target version.

## Steps

1. Enable debug mode and collect deprecation notices:

    ```bash
    grep -rn 'deprecated\|Smarty.*removed' var/logs/dev.log | grep mytheme
    ```

2. Grep templates for known deprecated patterns:

    ```bash
    grep -rn '{$link->getPageLink\|{$link->getCMSLink\|{$cookie\.' themes/mytheme/templates/
    grep -rn 'display_column_left\|display_column_right' themes/mytheme/templates/
    grep -rn '{addJsDef\|{addJsDefL' themes/mytheme/templates/
    ```

3. Replace `{$link->getPageLink()}` with the `{url}` helper:

    ```smarty
    {* Before *}
    <a href="{$link->getPageLink('contact')}">{l s='Contact' d='Shop.Theme.Global'}</a>

    {* After *}
    <a href="{url entity='cms' params=['id_cms_category' => 3]}">{l s='Contact' d='Shop.Theme.Global'}</a>
    ```

4. Replace removed `$cookie` access with structured variables:

    ```smarty
    {* Before *}
    {$cookie->customer_firstname}

    {* After *}
    {$customer.firstname}
    ```

5. Replace deprecated `{display_column_left}` / `{display_column_right}` with hook calls in layouts:

    ```smarty
    {* Before *}
    {if $page.page_name != 'index'}{display_column_left}{/if}

    {* After *}
    {hook h='displayLeftColumn'}
    ```

6. Replace `{addJsDef}` / `{addJsDefL}` with structured `prestashop` JS object or `data-*` attributes:

    ```smarty
    {* Before *}
    {addJsDef baseDir=$urls.base_url}

    {* After - use prestashop.urls.base_url in JS, already available globally *}
    ```

7. Replace `{$link->getCMSLink()}`:

    ```smarty
    {* Before *}
    <a href="{$link->getCMSLink(3)}">{l s='Terms' d='Shop.Theme.Global'}</a>

    {* After *}
    <a href="{url entity='cms' id=3}">{l s='Terms' d='Shop.Theme.Global'}</a>
    ```

8. Replace deprecated `{$base_dir}` and similar globals:

    ```smarty
    {* Before *}
    <img src="{$base_dir}modules/mymodule/logo.png" />

    {* After *}
    <img src="{$urls.base_url}modules/mymodule/logo.png" />
    ```

9. After all replacements, clear the cache and verify no deprecation warnings remain:

    ```bash
    bin/console cache:clear
    # Browse all page families and check var/logs/dev.log
    ```

10. Run the theme validator to confirm structural compliance:

    ```bash
    bin/console prestashop:theme:export mytheme --dry-run
    ```

## Do

- Check the [Changes in 9.x devdocs](https://devdocs.prestashop-project.org/9/modules/core-updates/) for the exhaustive list of removed variables per version.
- Use `{url entity='...' params=[...]}` for all page links; it handles pretty URLs and multilang.
- Access customer data through `{$customer.*}`, cart through `{$cart.*}`, shop through `{$shop.*}`.

## Don't

- Don't suppress deprecation warnings without fixing the underlying template code.
- Don't access `$cookie` directly; use the structured template variables provided by `FrontController`.
- Don't use `{addJsDef}` for new variables; pass data via `data-*` attributes or the global `prestashop` JS object.
- Don't hardcode URL paths; always use `{url}` or `{$urls.*}` variables.

## Related skills

- `migrate-classic-to-hummingbird` - full theme architecture migration.
- `migrate-theme-to-child` - reducing a fork to only necessary overrides.
- `theme-validate` - validating theme structure after migration.
