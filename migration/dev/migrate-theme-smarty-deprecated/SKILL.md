---
name: migrate-theme-smarty-deprecated
description: Replace deprecated Smarty helpers, removed variables, and changed syntax with PS 9 equivalents. Use when a theme triggers deprecation warnings or breaks due to removed Smarty features after upgrading to PrestaShop 9.
---

## Requirements

- Identify the PS version being upgraded from and to.
- Run the theme in debug mode to surface deprecation notices.
- Grep templates for known deprecated patterns (listed below).

## Steps

1. **Audit for deprecated patterns:**

    ```bash
    # Deprecated {l} without domain
    grep -rn '{l s=' themes/mytheme/templates/ | grep -v 'd='
    # Removed {counter}, {cycle}, {math}
    grep -rn '{counter\|{cycle\|{math' themes/mytheme/templates/
    # Deprecated {displayBlock}
    grep -rn '{displayBlock' themes/mytheme/templates/
    # Old hook syntax
    grep -rn "{hook h='" themes/mytheme/templates/
    # Removed variables
    grep -rn '\$link->\|\$cookie->\|\$smarty\.now' themes/mytheme/templates/
    ```

2. **Fix `{l}` calls - domain is now required:**

    ```smarty
    {* BEFORE (deprecated - missing domain) *}
    {l s='Add to cart'}

    {* AFTER *}
    {l s='Add to cart' d='Shop.Theme.Actions'}
    ```

    Common domains: `Shop.Theme.Global`, `Shop.Theme.Actions`, `Shop.Theme.Catalog`, `Shop.Theme.Checkout`, `Shop.Theme.Customeraccount`.

3. **Replace removed Smarty plugins:**

    | Removed | Replacement |
    |---------|-------------|
    | `{counter}` | Use a Smarty `{assign}` + `{math}` or move logic to controller |
    | `{cycle}` | Use CSS `:nth-child()` or `{if $index % 2}` |
    | `{math}` | Compute in PHP controller and assign result to template |
    | `{displayBlock}` | Use `{block name='...'}...{/block}` |

    ```smarty
    {* BEFORE *}
    {counter name='row' assign='row_num'}

    {* AFTER *}
    {assign var='row_num' value=$row_num+1}
    ```

4. **Update changed hook syntax:**

    ```smarty
    {* BEFORE (string params) *}
    {hook h='displayProductActions' product=$product}

    {* AFTER (same syntax is valid, but ensure the hook name matches PS 9 registry) *}
    {hook h='displayProductActions' product=$product}
    ```

    Check renamed hooks in PS 9: run `bin/console prestashop:update:hooks-documentation` and compare with your usage.

5. **Replace removed/renamed template variables:**

    | Removed/Changed | PS 9 equivalent |
    |----------------|-----------------|
    | `$link->getPageLink(...)` | `{$urls.pages.contact}` (use `$urls` array) |
    | `$cookie->id_lang` | `{$language.id}` |
    | `$smarty.now` | `{$smarty.now|date_format:'%Y'}` (still works) or use `{$currentDate}` |
    | `$product.description_short` (raw) | `{$product.description_short nofilter}` |
    | `$category->id` | `{$category.id}` (object to array access) |

6. **Update product/category variable access.** PS 9 uses `ProductLazyArray` and `CategoryLazyArray`:

    ```smarty
    {* BEFORE (object access, may break) *}
    {$product->name}
    {$product->getPrice()}

    {* AFTER (array access via LazyArray) *}
    {$product.name}
    {$product.price}
    ```

7. **Replace `{url}` deprecated parameters:**

    ```smarty
    {* BEFORE *}
    {url entity='category' id=$category.id}

    {* AFTER (same syntax, but verify params match PS 9 UrlGenerator) *}
    {url entity='category' id=$category.id}
    ```

8. **Validate - enable debug mode and check logs:**

    ```bash
    # In config/defines.inc.php
    define('_PS_MODE_DEV_', true);
    ```

    Browse all theme pages. Check `var/logs/dev.log` for remaining deprecation notices.

## Do

- Always include `d='Shop.Theme.*'` domain in `{l}` tags.
- Use the `$urls` array for all link generation in templates.
- Access product/category data as arrays (`$product.name`) not objects (`$product->name`).
- Test with `_PS_MODE_DEV_` enabled to catch silent deprecations.

## Don't

- Don't suppress deprecation warnings without fixing the underlying cause.
- Don't use `{php}` tags; they are disabled by default and removed in PS 9.
- Don't assume variable names are unchanged between major versions; always check the assigned variables in the controller.
- Don't use `{include file='...' absolute path}`; use module or theme-relative paths.

## Canonical examples

- [devdocs - Core updates for themes](https://devdocs.prestashop-project.org/9/modules/core-updates/)
- [devdocs - Template inheritance](https://devdocs.prestashop-project.org/9/themes/reference/template-inheritance/parent-child-feature/)

## Related skills

- `migrate-classic-to-hummingbird` - full theme migration (includes Smarty fixes).
- `theme-add-translation` - modern translation workflow.
- `migrate-legacy-translations` - module translation migration.
