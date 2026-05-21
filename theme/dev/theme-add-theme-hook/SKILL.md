---
name: theme-add-theme-hook
description: Register a new theme-defined display hook in `config/theme.yml` (under `global_settings.hooks.custom_hooks`), call it from the right Smarty template, and document the payload modules will receive. Use when no built-in hook fits and you need a new extension point modules can attach to.
---

## Requirements

Ask the user:
* Hook name. Must follow PrestaShop conventions: PascalCase, prefixed `display`, ideally namespaced with the theme (e.g. `displayThemeFooterReassurance`, `displayHummingbirdProductExtra`). Action hooks are not the theme's job - stay on `display*`.
* Where the hook fires: which template file and which `{block}` region. One hook = one position; do not reuse a name in multiple templates.
* Payload variables the hook will receive. Modules read these via `$params` in their handler, so the contract has to be stable from day one (e.g. `product=$product category=$category`).
* A short title and description for the BO Design > Positions screen, where merchants attach modules to the hook.

## Steps

1. Pick the call site. New theme hooks belong inside an existing page template, NOT inside a layout (layouts are shared - a "page section" hook in `layouts/layout-full-width.tpl` would fire on every page and break the contract). Wrap the hook call in a named Smarty block so child themes can override the surrounding markup:

    ```smarty
    {* templates/catalog/product.tpl *}
    {block name='product_trust_badges'}
      <div class="product-trust-badges">
        {hook h='displayThemeProductTrustBadge' product=$product category=$category}
      </div>
    {/block}
    ```

    Always pass the relevant entity through hook parameters (`product=$product`, `cart=$cart`, `customer=$customer`). It saves every subscribing module from re-fetching the object.

2. Declare the hook in `config/theme.yml` under `global_settings.hooks.custom_hooks`. The same key holds `modules_to_hook` and `modules_to_unhook`; keep all hook-related declarations together. The `name` field MUST exactly match the string passed to `{hook h='...'}`:

    ```yaml
    global_settings:
      hooks:
        custom_hooks:
          - name: displayThemeProductTrustBadge
            title: displayThemeProductTrustBadge
            description: Renders trust badges below the product price on the product page.
          - name: displayThemeFooterReassurance
            title: displayThemeFooterReassurance
            description: Adds reassurance items (returns, support, payment) above the footer.
        modules_to_hook:
          displayThemeFooterReassurance:
            - ps_emailsubscription
    ```

    `modules_to_hook` and `modules_to_unhook` are applied ONLY when the theme is activated. Editing them on an already-active theme has no runtime effect; reactivate the theme (BO or `bin/console prestashop:theme:enable <name>`) to re-run the hook wiring.

3. Reactivate the theme so PrestaShop registers the new hook in the database. The Back Office surfaces it under Design > Positions; merchants can drag any compatible module onto it. CLI shortcut:

    ```bash
    bin/console prestashop:theme:enable <theme-name>
    ```

4. Document the contract in your theme README so module authors know the payload. List the hook name, where it fires (template + block), the variables passed, and any expected return contract (display hooks should return safe HTML; PrestaShop concatenates the output of every attached module).

5. Modules subscribe to the new hook from their own `install()`:

    ```php
    public function install(): bool
    {
        return parent::install()
            && $this->registerHook('displayThemeProductTrustBadge');
    }

    public function hookDisplayThemeProductTrustBadge(array $params): string
    {
        $product = $params['product'] ?? null;
        return $this->fetch('module:'.$this->name.'/views/templates/hook/trust-badge.tpl');
    }
    ```

    See `module-register-hooks` for the full module-side flow (subscribing, dispatching, multistore-aware activation).

6. Verify. With Developer Mode on, view-source on the target page and look for the rendered hook block:

    ```html
    <!-- begin module:<your_module>/views/templates/hook/trust-badge.tpl -->
    <div class="trust-badge">...</div>
    <!-- end module:<your_module>/views/templates/hook/trust-badge.tpl -->
    ```

    If nothing appears, check Design > Positions to confirm the hook exists and a module is attached, then clear the Smarty cache (Advanced Parameters > Performance > Clear cache).

## Do

- Use the `displayTheme<Name>` or `display<ThemeName><Position>` convention. Generic names like `displayExtraBlock` collide with other themes and confuse module authors.
- Define a stable payload from day one. Renaming a `$params` key after modules have shipped against it is a breaking change.
- Wrap the `{hook}` call in a named `{block}` so child themes and overrides can move it without touching the hook itself.
- Document hooks in the theme README with: name, file:line, payload, expected output.

## Don't

- Don't add `display*` hooks to a layout. Use page templates (`templates/<page>.tpl`) - layout hooks fire site-wide.
- Don't reuse a built-in hook name (`displayHome`, `displayFooter`, ...) for theme-specific positions. Pick a new namespaced name.
- Don't define `action*` hooks from the theme. Action hooks are meant for PHP logic; themes are templates and should expose `display*` hooks only.
- Don't mutate `modules_to_hook` / `modules_to_unhook` and expect a hot reload. They run only at theme activation.

## Canonical examples

- [devdocs - Theme hooks](https://devdocs.prestashop-project.org/9/themes/concepts/hooks/) - `{hook}` syntax, `custom_hooks` declaration, `modules_to_hook` / `modules_to_unhook` activation behaviour.
- [devdocs - Theme structure](https://devdocs.prestashop-project.org/9/themes/concepts/theme-structure/) - full `config/theme.yml` schema including `global_settings.hooks`.
- [devdocs - Hummingbird hook maps](https://devdocs.prestashop-project.org/9/themes/hummingbird/hooks/) - reference for how PrestaShop's flagship theme names and positions display hooks.
- [`PrestaShop/hummingbird` - `config/theme.yml`](https://github.com/PrestaShop/hummingbird/blob/develop/config/theme.yml) - real-world `global_settings.hooks.custom_hooks` and `modules_to_hook` declarations.
- [devdocs - `prestashop:theme:enable`](https://devdocs.prestashop-project.org/9/development/components/console/prestashop-theme-enable/) - CLI command that re-runs hook wiring after editing `theme.yml`.

## Related skills

- `theme-add-page-section` - the consumer side: how to choose the right hook (built-in or new) for a page customisation.
- `theme-override-template` - if you cannot expose a hook (third-party theme, frozen contract), template override is the fallback.
- `module-register-hooks` - the module-side `registerHook()` / `hookDisplay...()` flow that subscribes to your new hook.
- `module-add-widget` - alternative content-injection mechanism when the merchant should pick the position from any `{widget}` call site.
