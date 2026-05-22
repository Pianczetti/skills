---
name: migrate-legacy-translations
description: Replace $this->l('...') and $module->l(...) calls with Symfony Translator using Modules.<Modulename>.<Domain> keys and XLIFF catalogues. Use when a module uses the legacy translation helper and must migrate to the PS 9 XLIFF-based translation system.
---

## Requirements

- Grep the codebase for `$this->l(`, `$module->l(`, and `Translate::getModuleTranslation(` calls.
- Decide the domain suffix per audience: `Admin` for Back Office, `Shop` for front office.
- Confirm the PascalCase module name for the domain (e.g. module `mymodule` becomes `Modules.Mymodule.Admin`).

## Steps

1. **Identify legacy patterns** to replace:

    ```php
    // LEGACY PHP (module class, legacy controller, helper)
    $this->l('Settings saved')
    $this->module->l('Invalid code')

    // LEGACY Smarty
    {l s='Welcome' mod='mymodule'}

    // LEGACY Twig (rare but exists)
    {{ modules.mymodule.l('Hello') }}
    ```

2. **Replace PHP calls** - in services, inject `TranslatorInterface`:

    ```php
    use Symfony\Contracts\Translation\TranslatorInterface;

    class MyService
    {
        public function __construct(private readonly TranslatorInterface $translator) {}

        public function message(): string
        {
            return $this->translator->trans('Settings saved', [], 'Modules.Mymodule.Admin');
        }
    }
    ```

3. **Replace PHP calls in controllers** - use the inherited `$this->trans()`:

    ```php
    // In PrestaShopAdminController actions:
    $this->addFlash('success', $this->trans('Settings saved', 'Modules.Mymodule.Admin'));
    ```

4. **Replace PHP calls in the main module class** (legacy bridge context):

    ```php
    // The main module class supports the three-argument form:
    $this->trans('Settings saved', [], 'Modules.Mymodule.Admin');
    ```

5. **Replace Smarty templates** - use `{l}` with the `d=` parameter:

    ```smarty
    {* BEFORE: *}  {l s='Welcome' mod='mymodule'}
    {* AFTER:  *}  {l s='Welcome' d='Modules.Mymodule.Shop'}
    ```

6. **Replace Twig templates** - use the `trans` filter with the domain:

    ```twig
    {# BEFORE: #}  {{ 'Hello'|trans }}
    {# AFTER:  #}  {{ 'Hello'|trans({}, 'Modules.Mymodule.Admin') }}
    ```

7. **Export XLIFF catalogues** after all replacements:

    ```bash
    php bin/console prestashop:translation:export-module mymodule
    ```

    This generates `translations/en-US/Modules.Mymodule.Admin.xlf` (and `.Shop.xlf` if used). Commit these files.

8. **Provide translations** for other languages by adding `translations/<locale>/Modules.Mymodule.<Area>.xlf` files with `<target>` elements. Re-run the export command whenever new source strings are added.

## Do

- Use exactly `Modules.<PascalName>.<Area>` as the domain. The PascalCase name must match the module's technical name with first letter capitalized.
- Always pass the domain as the third argument to `trans()` or second to the Twig `trans` filter.
- Commit the `translations/en-US/*.xlf` source catalogues as the canonical key reference.

## Don't

- Don't use `$this->l('...')` in any new code; it writes to the legacy translation tables and is invisible to the BO Translation interface.
- Don't omit the domain argument; `trans('Hello')` falls into the shared `messages` domain.
- Don't hand-edit `<source>` tags in XLIFF; they must match the string passed to `trans()`. Edit the code, then re-export.
- Don't invent domains like `MyModule.Stuff`; only `Modules.<Name>.<Area>` is recognized.

## Canonical examples

- [devdocs - Module translation (new system)](https://devdocs.prestashop-project.org/9/modules/creation/module-translation/new-system/)
- [devdocs - Translation domains](https://devdocs.prestashop-project.org/9/development/internationalization/translation/translation-domains/)

## Related skills

- `module-add-translations-new` - full recipe for setting up the new translation system.
- `migrate-helperform-to-symfony-form` - often the form migration triggers translation migration.
