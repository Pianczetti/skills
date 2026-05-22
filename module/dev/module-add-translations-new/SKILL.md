---
name: module-add-translations-new
description: Wire the new (XLIFF + Symfony Translator) translation system into a PrestaShop 9 module. Use when the user wants translatable strings using `Modules.<Modulename>.<Area>` domains.
---

## Requirements

Ask the user:
* The translation domain area suffix per audience: `Admin` for Back Office strings, `Shop` for front-office strings, optionally `Modules.<Modulename>.<feature>` for a specific feature subset. Pick exactly one suffix per audience to keep the catalogue scannable.
* The languages to seed (e.g. `fr-FR`, `es-ES`). The default source catalogue is always English.
* Whether the module already has legacy `$this->l(...)` calls. If yes, plan to migrate them - they are NOT supported on the modern path.

## Steps

1. **PHP code** - inject `Symfony\Contracts\Translation\TranslatorInterface` (autowired) and use the right domain:

    ```php
    use Symfony\Contracts\Translation\TranslatorInterface;

    class GreetingService
    {
        public function __construct(private readonly TranslatorInterface $translator) {}

        public function greet(): string
        {
            return $this->translator->trans('Hello', [], 'Modules.Mymodule.Admin');
        }
    }
    ```

   In a modern admin controller (`PrestaShopAdminController`), use `$this->trans('Hello', 'Modules.Mymodule.Admin')`. In the main module class (legacy bridge), use `$this->trans('Hello', [], 'Modules.Mymodule.Admin')`.

2. **Twig** - call the `trans` filter with the domain as the second argument:

    ```twig
    {{ 'Hello'|trans({}, 'Modules.Mymodule.Admin') }}
    {{ 'Welcome %name%'|trans({'%name%': customer.firstname}, 'Modules.Mymodule.Shop') }}
    ```

3. **Smarty** (front-office templates) - use the `{l}` tag with the `d=` domain attribute:

    ```smarty
    {l s='Hello' d='Modules.Mymodule.Shop'}
    {l s='Welcome %name%' sprintf=['%name%' => $customer.firstname] d='Modules.Mymodule.Shop'}
    ```

4. **Generate XLIFF source files** by running the `prestashop:translation:export-module` console command from the PrestaShop root. It scans all known sources (PHP, Twig, Smarty) and produces a fresh XLIFF tree under the module:

    ```bash
    php bin/console prestashop:translation:export-module mymodule
    ```

   This creates `translations/<lang>/Modules.Mymodule.<area>.xlf` (for example `translations/en-US/Modules.Mymodule.Admin.xlf`). Re-run after adding new strings.

5. **Commit policy**:

    * Commit the source catalogue (`translations/en-US/*.xlf`) - it is the canonical list of keys.
    * Commit any human translations you ship (`translations/fr-FR/*.xlf`, etc.).
    * Do NOT commit auto-generated empty files for languages you do not actually translate.

6. **Domain naming** must follow `Modules.<Modulename>.<Area>` exactly. The module name segment is **PascalCase of the technical name** (e.g. module `mymodule` → `Modules.Mymodule.Admin`). Mismatched casing breaks the catalogue lookup silently.

## Do

- Pick one domain area per audience: `Modules.Mymodule.Admin` (BO), `Modules.Mymodule.Shop` (FO), `Modules.Mymodule.<Feature>` for a feature with its own glossary.
- Inject `TranslatorInterface` rather than reaching for a global. It honours the request locale and BO language switcher.
- Re-run `bin/console prestashop:translation:export-module <name>` after every batch of new strings; commit the diff.
- For pluralisation use ICU MessageFormat: `'{count, plural, one {# item} other {# items}}'` with the domain.

## Don't

- **Do NOT use `$this->l('...')` in modern code.** That helper is the legacy translator; it does not write to the new domains, has no Twig/Smarty equivalent, and is excluded from the PS 9 modern path. Only the `*-legacy` skills are allowed to mention it.
- Don't invent ad-hoc domains like `MyModule.Stuff` or omit the `Modules.` prefix; the BO Translation interface will not group them.
- Don't hand-edit XLIFF `<source>` tags. They must match the exact string passed to `trans()` / `{l s='...'}`. Edit the calling code, then re-export.
- Don't skip the domain argument: `trans('Hello')` falls back to the `messages` domain, which is shared by the entire BO and will not appear under your module in the Translation interface.

## Canonical examples

- [devdocs - Module translation (new system)](https://devdocs.prestashop-project.org/9/modules/creation/module-translation/new-system/).
- [devdocs - Translation domains](https://devdocs.prestashop-project.org/9/development/internationalization/translation/translation-domains/).
- [devdocs - Using the Translator](https://devdocs.prestashop-project.org/9/development/internationalization/translation/using-the-translator/).
- [devdocs - `prestashop:translation:export-module` command](https://devdocs.prestashop-project.org/9/development/components/console/prestashop-translation-export-module/).
