---
name: theme-add-page-section
description: Customise a specific PrestaShop 9 front-office page (home, category, product, cart, checkout, login, my account, contact) by injecting markup at a documented hook position rather than overriding the template. Use whenever the user wants to add or replace a section on one page family.
---

## Requirements

Ask the user:
* Which page family is being customised. The mapping ties the controller's `php_self` to a template path and a hook map:
  * `index` -> `templates/index.tpl`
  * `category` / `manufacturer` / `supplier` -> `templates/catalog/listing/...tpl`
  * `product` -> `templates/catalog/product.tpl`
  * `cart` -> `templates/checkout/cart.tpl`
  * `checkout` / `order` -> `templates/checkout/checkout.tpl`
  * `authentication` / `registration` / `password` -> `templates/customer/...tpl`
  * `my-account` / `history` / `addresses` / `identity` -> `templates/customer/...tpl`
  * `contact` -> `templates/contact.tpl`
* What should appear in the new section (static markup, a module's output, a CMS block, a custom widget) and where on the page (above the title, below the gallery, between two existing blocks, in the footer of the page).
* Whether the section is page-specific or shared (header/footer hooks fire on every page and belong in the layout, not in a page template).

If the goal is to replace a whole page tree or every variant of a layout, redirect to `theme-override-template` or `theme-add-layout`.

## Steps

1. Find the documented hook for the desired position. PrestaShop publishes a visual map of every Hummingbird front-office hook (header and footer hooks are listed once on the home page; page-specific hooks are listed under each page):

    | Page | Hook map |
    |---|---|
    | Home | [hooks/homepage](https://devdocs.prestashop-project.org/9/themes/hummingbird/hooks/homepage/) |
    | Category | [hooks/categorypage](https://devdocs.prestashop-project.org/9/themes/hummingbird/hooks/categorypage/) |
    | Product | [hooks/productpage](https://devdocs.prestashop-project.org/9/themes/hummingbird/hooks/productpage/) |
    | Cart | [hooks/cartpage](https://devdocs.prestashop-project.org/9/themes/hummingbird/hooks/cartpage/) |
    | Checkout | [hooks/checkoutflow](https://devdocs.prestashop-project.org/9/themes/hummingbird/hooks/checkoutflow/) |
    | Login & registration | [hooks/connexionpage](https://devdocs.prestashop-project.org/9/themes/hummingbird/hooks/connexionpage/) |
    | My account | [hooks/myaccount](https://devdocs.prestashop-project.org/9/themes/hummingbird/hooks/myaccount/) |
    | Contact | [hooks/contactpage](https://devdocs.prestashop-project.org/9/themes/hummingbird/hooks/contactpage/) |

    Pick the hook closest to the wanted position (e.g. `displayWrapperTop` for "below the breadcrumb, above the page content", `displayFooterBefore` for "above the footer", `displayProductActions` inside the product card).

2. If a hook already exists at the right position, render it and stop. In Smarty templates use the `{hook}` function:

    ```smarty
    {* templates/index.tpl *}
    {hook h='displayHome'}
    {hook h='displayWrapperTop'}
    ```

    Modules subscribe to that hook from their `install()` (`registerHook('displayHome')`) and inject their output. The theme keeps its template untouched and stays update-friendly.

3. If the desired position is page-specific and no hook exists yet, add one in the page template (NOT the layout) and declare it as a theme-defined hook in `config/theme.yml` so the BO surfaces it under Design > Positions. See `theme-add-theme-hook` for the full declaration; the call site looks like:

    ```smarty
    {* templates/catalog/product.tpl *}
    {block name='product_buy_section'}
      {hook h='displayProductBuyButtons' product=$product}
      {* the new hook a module can attach to *}
      {hook h='displayThemeProductTrustBadge' product=$product}
    {/block}
    ```

    `{hook h='...' product=$foo bar=$baz}` passes parameters to every attached module via `$params` in the hook handler.

4. Only override the template (`theme-override-template`) when none of the above works (the position is mid-tag, inside a block that is not extension-friendly, or the change is structural). When you do, copy the parent template into your theme's matching path, extend it with the `parent:` Smarty resource prefix and override only the relevant `{block}`:

    ```smarty
    {* themes/<your_theme>/templates/checkout/cart-detailed.tpl *}
    {extends file='parent:checkout/cart-detailed.tpl'}

    {block name='hook_shopping_cart'}
      {$smarty.block.parent}
      {hook h='displayThemeCartReassurance'}
    {/block}
    ```

5. Verify. With Developer Mode on (`_PS_MODE_DEV_ = true`, BO Performance: Force compilation Yes / Cache No), open the page and view-source. Each rendered template is wrapped with `<!-- begin /var/www/html/themes/.../template.tpl -->` markers; rendered hooks appear as `<!-- begin module:<modulename>/... -->` blocks. Confirm the new section is in the right spot before reactivating production cache.

## Do

- Prefer hooks over template overrides whenever a hook exists at the right position. Hooks survive theme upgrades; templates have to be diff-merged.
- Use the visual hook map for the target page (linked above) before inventing a new hook. The Hummingbird theme already exposes ~15-25 hooks per page family.
- Pass page context through hook parameters (`{hook h='displayThemeProductExtra' product=$product}`) so module handlers do not have to re-fetch the entity.
- If you must override a template, override the smallest possible `{block}` and call `{$smarty.block.parent}` to keep the parent's markup intact.

## Don't

- Don't copy 90% of a page's template just to inject 5 lines. Add (or use) a hook instead.
- Don't put a page-specific hook in `templates/_partials/header.tpl` or in a layout. Header and footer hooks fire on every page and pollute the BO Positions map.
- Don't introduce business logic (`{if Configuration::get(...)}`, raw SQL via `{php}`) in templates. Encapsulate it in a module that handles the hook.
- Don't bypass the hook system with hard-coded module includes (`{include file='module:ps_imageslider/views/templates/hook/slider.tpl'}`). It re-couples your theme to a specific module and breaks when merchants disable that module.

## Canonical examples

- [devdocs - Theme hooks (concepts)](https://devdocs.prestashop-project.org/9/themes/concepts/hooks/) - the `{hook}` Smarty function, parameter passing and `modules_to_unhook` / `modules_to_hook`.
- [devdocs - Hummingbird hook maps](https://devdocs.prestashop-project.org/9/themes/hummingbird/hooks/) - Figma maps and reference tables of every front-office hook, page by page.
- [devdocs - Templates and layouts](https://devdocs.prestashop-project.org/9/themes/concepts/templates/templates-and-layouts/) - resolution chain so you can target locale or entity-ID variants of a page.
- [devdocs - Smarty helpers](https://devdocs.prestashop-project.org/9/themes/reference/smarty-helpers/) - `{hook}`, `{widget}`, `{render}`, `{l}` reference.
- [`PrestaShop/hummingbird` - `templates/index.tpl`](https://github.com/PrestaShop/hummingbird/blob/develop/templates/index.tpl) and [`templates/catalog/product.tpl`](https://github.com/PrestaShop/hummingbird/blob/develop/templates/catalog/product.tpl) - reference call sites for every documented hook.

## Related skills

- `theme-add-theme-hook` - register a brand-new theme-defined hook when none of the documented ones fit.
- `theme-override-template` - last-resort path when the change is structural and no hook can carry it.
- `theme-add-layout` - add a new layout instead of mutating a page template.
- `module-add-widget` and `module-register-hooks` - the module-side counterpart that plugs content into the hook the theme exposes.
