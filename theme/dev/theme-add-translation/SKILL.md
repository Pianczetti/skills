---
name: theme-add-translation
description: Translate a PrestaShop 9 theme - wrap user-facing strings in templates with the `{l}` Smarty function under the right domain (`Shop.<Themename>` for theme-owned wordings, `Shop.Theme.*` to reuse core translations), then export the catalog from the Back Office. Use when the theme renders any English string and needs to support more than one language.
---

## Requirements

Ask the user:
* The theme's technical name (drives the translation domain). The display rule is strict: capital first letter, the rest lowercase, no other uppercase. `Shop.Mytheme` is correct; `Shop.MyTheme`, `Shop.MYTHEME`, `Shop.my-theme` are not.
* Which strings need to be translated and which are already covered by PrestaShop's core domains (Add to cart, Sign in, Your cart, Continue shopping...). Reuse core domains whenever the wording matches - they ship pre-translated in every official language pack.
* Which languages must be supported. Each must be installed in International > Localization > Languages first; the language pack pulls the core `Shop.Theme.*` translations automatically.
* Whether translations will be distributed with the theme (`translations/<lang>/Shop.<Themename>.<area>.xlf` files inside the theme zip) or handled by each merchant after install.

## Steps

1. Wrap every user-facing string in a Smarty template with `{l}` and a `d=` domain. All source strings MUST be written in English; PrestaShop indexes them by source string, so any change to the English wording invalidates existing translations:

    ```smarty
    <h1>{l s='Sign in' d='Shop.Theme.CustomerAccount'}</h1>
    <button>{l s='Add to cart' d='Shop.Theme.Actions'}</button>
    <p>{l s='Welcome back to %shop_name%' d='Shop.Mytheme' sprintf=['%shop_name%' => $shop.name]}</p>
    ```

    Pick the domain by ownership, not by topic:
    | Domain | When to use |
    |---|---|
    | `Shop.<Themename>` | Strings owned by your theme (slogans, custom block titles, theme-specific banners). Replace `<Themename>` with the theme name, capital first letter only. |
    | `Shop.Theme.Global` | Generic UI labels (Yes, No, Search, Close...). |
    | `Shop.Theme.Actions` | Button labels (`Add to cart`, `Save`, `Continue shopping`...). |
    | `Shop.Theme.Catalog` | Catalog and listing wordings (`No products available yet`, `Sort by`...). |
    | `Shop.Theme.Checkout` | Cart and checkout wordings. |
    | `Shop.Theme.CustomerAccount` | Account, login, register, addresses, orders. |
    | `Shop.Theme.Email` | Strings rendered inside transactional emails. |

    Reusing core domains is preferred - those wordings are translated in every official language pack at no cost.

2. For dynamic values, use the `sprintf` parameter rather than concatenating strings (concatenation breaks every non-English language):

    ```smarty
    {l s='%count% items in your cart' d='Shop.Mytheme' sprintf=['%count%' => $cart.products_count]}
    {l s='Welcome %firstname% %lastname%!' d='Shop.Mytheme' sprintf=['%firstname%' => $customer.firstname, '%lastname%' => $customer.lastname]}
    ```

    Tag-style placeholders also work for inline HTML (`[1]link[/1]`) - the placeholder values inject opening and closing tags so translators see clean text.

3. For strings consumed by JavaScript, render them from a Smarty template into a JS object and read them client-side. The `js=1` flag escapes quotes for safe embedding:

    ```smarty
    <script>
      window.themeTranslations = {
        addedToCart: '{l s='Product added to cart' d='Shop.Mytheme' js=1}',
        confirmRemove: '{l s='Are you sure?' d='Shop.Mytheme' js=1}'
      };
    </script>
    ```

    Never hard-code English strings inside `assets/js/` - they are not picked up by the BO translation form.

4. Generate the catalog from the Back Office. There is no CLI scan for theme catalogs; the BO walks `templates/` and surfaces every `{l s='...' d='...'}` it finds:

    1. International > Translations > Add / Update a language: install every target language. This downloads the core language pack, including `Shop.Theme.*`.
    2. International > Translations > Modify translations:
        * Type: `Theme translations` (front office).
        * Theme: your theme name.
        * Language: target language.
    3. Click Modify. The form lists every wording grouped by domain. Navigate to `Shop` > `<Themename>` for your custom strings; `Shop` > `Theme` > `<area>` for any core strings you reused but want to override.
    4. Fill in translations and Save. PrestaShop writes them into the database (the `translation` table); the active language picks them up immediately.

5. To distribute the theme with translations baked in, export them as XLIFF and ship them in `translations/`:

    1. International > Translations > Export a language: select your theme and the language, download the ZIP.
    2. Extract it; the result is one `.xlf` per domain, e.g. `translations/fr-FR/Shop.Mytheme.xlf`, `translations/fr-FR/Shop.Theme.Catalog.xlf`.
    3. Commit the `translations/` directory to your theme repository. The theme zip exporter (`prestashop:theme:export`) and `theme-export-zip` skill bundle it automatically.

6. Verify. Switch the front office to the target language and walk every page that contains translated strings. The `?debug-translations=1` query string highlights every translated wording on screen and is the fastest way to spot any English fallthrough.

## Do

- Use `Shop.Theme.*` whenever the wording exists there. It comes pre-translated in every language pack and saves you hundreds of translator hours.
- Keep the `<Themename>` casing strict: capital first letter, rest lowercase, no other uppercase. PrestaShop's translation indexer rejects mixed case.
- Use placeholders (`sprintf=['%name%' => $value]`) and `[1]...[/1]` tag pairs instead of concatenation; concatenation is untranslatable.
- Run a final pass with `?debug-translations=1` on every page family before shipping.

## Don't

- Don't invent custom top-level domains (`Theme.Custom.Buttons`, `MyTheme.Forms`, ...). The translation system only resolves `Shop.<Themename>` for theme-owned strings and `Shop.Theme.*` for shared ones.
- Don't translate from a non-English source. PrestaShop hashes the English source string; using French (or any other language) as the source means future English speakers see French and existing translations break the day someone fixes the source.
- Don't write strings directly in `assets/js/`. They are invisible to the BO translation form. Inject them via a Smarty template and `js=1`.
- Don't edit `.xlf` files by hand for live shops. Export, edit in the BO (which keeps the database and XLIFF in sync), then re-export.

## Canonical examples

- [devdocs - Translation (theme)](https://devdocs.prestashop-project.org/9/themes/concepts/translation/) - full `{l}` syntax, theme-name casing rules, `sprintf`, `js=1`, BO export workflow.
- [devdocs - Translation domains](https://devdocs.prestashop-project.org/9/development/internationalization/translation/translation-domains/) - the `Shop.*`, `Modules.*`, `Admin.*` namespaces and how PrestaShop resolves them.
- [devdocs - Internationalization](https://devdocs.prestashop-project.org/9/development/internationalization/translation/) - architecture of the translator, XLIFF format, language packs.
- [devdocs - Smarty helpers](https://devdocs.prestashop-project.org/9/themes/reference/smarty-helpers/) - `{l}` parameter list (`s`, `d`, `sprintf`, `js`, `tags`).
- [`PrestaShop/hummingbird` templates](https://github.com/PrestaShop/hummingbird/blob/develop/templates/index.tpl) - reference call sites mixing `Shop.Theme.*` reuse with theme-owned `Shop.Hummingbird` strings.

## Related skills

- `theme-create-from-scratch` - bootstraps the theme that owns the `Shop.<Themename>` domain.
- `theme-export` - bundles the `translations/` directory in the distributable archive.
- `module-add-translations-new` - module-side counterpart using `Modules.<modulename>.<domain>` keys.
