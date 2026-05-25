---
name: migrate-legacy-translations
description: Replace $this->l() / $module->l() legacy translation calls with Symfony Translator using Modules.<Modulename>.<Area> domains. Use when a module still uses the legacy on-the-fly translation system and needs extractable, Crowdin-compatible keys.
---

## Requirements

- Module name (determines the translation domain, e.g. `Modules.Mymodule.Admin`).
- Areas in use: `Admin` (back office), `Shop` (front office), `Global` (shared).
- Whether XLIFF catalogues already exist under `translations/`.

## Steps

1. Grep for all legacy translation calls:

    ```bash
    grep -rn '\$this->l(' modules/mymodule/
    grep -rn '\$module->l(' modules/mymodule/
    grep -rn "->l(" modules/mymodule/
    ```

2. In PHP service classes and controllers, replace `$this->l('text')` with `$this->trans('text', [], 'Modules.Mymodule.Admin')`. The `trans()` method is available on any class extending `PrestaShopAdminController` or using `TranslatorAwareTrait`:

    ```php
    // Before
    $this->l('Settings saved');

    // After
    $this->trans('Settings saved', [], 'Modules.Mymodule.Admin');
    ```

3. In the module main class, replace `$this->l('...')` in `install()`, `getContent()`, or hook methods with `$this->trans('...', [], 'Modules.Mymodule.Admin')`. The base `Module` class exposes `trans()` since PS 1.7.6.

4. In Twig templates, use the `trans` filter with the domain:

    ```twig
    {# Before (if migrating from Smarty) #}
    {l s='Save' mod='mymodule'}

    {# After #}
    {{ 'Save'|trans({}, 'Modules.Mymodule.Admin') }}
    ```

5. In Smarty front-office templates (`.tpl`), use the `{l}` function with `d=` parameter:

    ```smarty
    {l s='Add to cart' d='Modules.Mymodule.Shop'}
    ```

6. Export the translation catalogue to generate XLIFF files:

    ```bash
    bin/console prestashop:translation:export-module mymodule
    ```

    This creates `translations/<locale>/Modules.Mymodule.<Area>.xlf` files.

7. Verify all keys are extracted by running:

    ```bash
    bin/console prestashop:translation:find-missing mymodule
    ```

8. Remove any leftover `$this->l()` calls. Confirm no regressions by loading each page and verifying strings render correctly.

## Do

- Use exactly `Modules.<Modulename>.<Area>` as the domain. The module name segment is PascalCase with first letter uppercase and rest lowercase (e.g. `Mymodule` not `MyModule`).
- Keep the original English string as the translation key for readability.
- Group strings: `Admin` for BO pages, `Shop` for FO templates, `Global` for shared (rare).

## Don't

- Don't invent custom domains (e.g. `Modules.Mymodule.Emails`); stick to `Admin`, `Shop`, `Global`.
- Don't use `$this->l()` in new code; it is not extractable by the catalogue export.
- Don't hardcode translated strings; always pass them through the translator.

## Related skills

- `module-add-translations-new` - greenfield translation wiring.
- `migrate-helperform-to-symfony-form` - form labels need translated domains.
- `migrate-admin-controller` - controller flashes need translated domains.
