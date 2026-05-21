---
name: module-add-mail-template
description: Add custom transactional or marketing mail templates to a PrestaShop 9 module and render them through MailTemplateRendererInterface, with translatable subject and body. Use when the user wants the module to send emails or extend the core mail themes.
---

## Requirements

Ask the user:
* Template technical name in `snake_case` (e.g. `order_recap`, `low_stock_alert`). It becomes the file stem under `mails/<lang>/`.
* Languages to seed (ISO codes - at minimum `en`).
* Layout type: transactional (uses the active mail theme - classic / modern / dark) or marketing (custom HTML, no theme wrapper).
* Whether the template is module-specific or contributes a new layout to every theme.

## Steps

1. For a simple module-owned template, drop the source files under `mails/<lang>/<template>.{html,txt}` (legacy path, still fully supported in PS 9):

    ```
    mymodule/
      mails/
        en/
          order_recap.html
          order_recap.txt
        fr/
          order_recap.html
          order_recap.txt
    ```

   The HTML file may contain `{name}`, `{order_id}`, ... placeholders consumed by `Mail::send()`.

2. Send the email from a service via the legacy bridge `Mail::send()`. Inject the module path and language so the right template variant is picked:

    ```php
    Mail::send(
        (int) $idLang,
        'order_recap',
        $this->translator->trans('Your order recap', [], 'Emails.Subject'),
        ['{name}' => $customer->firstname, '{order_id}' => (int) $order->id],
        $customer->email,
        $customer->firstname . ' ' . $customer->lastname,
        null, null, null, null,
        _PS_MODULE_DIR_ . 'mymodule/mails/'
    );
    ```

3. To render templates through the modern pipeline (transformations, theme variables, hooks), inject `PrestaShop\PrestaShop\Core\MailTemplate\MailTemplateRendererInterface` (service ID `prestashop.core.mail_template.mail_template_renderer`):

    ```php
    use PrestaShop\PrestaShop\Core\MailTemplate\MailTemplateRendererInterface;
    use PrestaShop\PrestaShop\Core\MailTemplate\Layout\LayoutInterface;

    final class OrderRecapMailer
    {
        public function __construct(private readonly MailTemplateRendererInterface $renderer) {}

        public function previewHtml(LayoutInterface $layout, string $locale): string
        {
            return $this->renderer->renderHtml($layout, $locale);
        }

        public function previewTxt(LayoutInterface $layout, string $locale): string
        {
            return $this->renderer->renderTxt($layout, $locale);
        }
    }
    ```

4. To contribute a new layout to every mail theme (so merchants can preview and regenerate it from the BO), listen to the `actionListMailThemes` hook and append a `Layout` to the relevant `ThemeInterface`:

    ```php
    use PrestaShop\PrestaShop\Core\MailTemplate\Layout\Layout;
    use PrestaShop\PrestaShop\Core\MailTemplate\ThemeCollectionInterface;

    public function hookActionListMailThemes(array $hookParams): void
    {
        /** @var ThemeCollectionInterface $themes */
        $themes = $hookParams['mailThemes'];
        foreach ($themes as $theme) {
            if (!in_array($theme->getName(), ['classic', 'modern'], true)) {
                continue;
            }
            $theme->getLayouts()->add(new Layout(
                'order_recap',
                '@Modules/mymodule/mails/layouts/order_recap_' . $theme->getName() . '.html.twig',
                '@Modules/mymodule/mails/layouts/order_recap_' . $theme->getName() . '.txt.twig',
                $this->name
            ));
        }
    }
    ```

   Register the `actionListMailThemes` hook in `install()` (see `module-register-hooks`).

5. Regenerate the on-disk HTML/TXT files for a theme + locale from the PrestaShop project root:

    ```bash
    bin/console prestashop:mail:generate <theme> <locale>
    # e.g. bin/console prestashop:mail:generate modern en
    ```

   The command takes a theme name and a locale (e.g. `en`, `fr`, `pt_BR`) and writes both `.html` and `.txt` outputs.

6. Translations live in dedicated XLIFF domains: subjects in `Emails.Subject`, bodies in `Emails.Body`. In PHP, in templates, and in form labels:

    ```php
    $subject = $this->translator->trans('Your order recap', [], 'Emails.Subject');
    ```

    ```twig
    {{ 'Thanks for your order, {name}'|trans({'{name}': customer_name}, 'Emails.Body') }}
    ```

   Export the module's strings with `bin/console prestashop:translation:export-module mymodule` (see `module-add-translations-new`).

## Do

- Use `Emails.Subject` for subjects and `Emails.Body` for body content. Other strings (BO labels for the email preview, etc.) keep their usual `Modules.<Name>.<Area>` domain.
- Inject `MailTemplateRendererInterface` (or `MailTemplateGenerator` for batch regeneration) instead of duplicating Twig render logic.
- Provide BOTH `.html` and `.txt` variants of every template. Mail clients fall back to `.txt` and accessibility / spam filters require it.
- Use the `actionListMailThemes`, `actionBuildMailLayoutVariables`, and `actionGetMailLayoutTransformations` hooks to extend or replace core layouts cleanly.

## Don't

- Don't hardcode template strings inline in PHP. Translate everything through the `Emails.Subject` / `Emails.Body` domains so the BO translation tool can pick them up.
- Don't write to `mails/<lang>/<file>` from runtime code. Source files are commit-only; runtime emails are produced by `Mail::send()` or the renderer.
- Don't bypass the renderer for HTML emails - it applies the CSS-inline and HTML-to-text transformations that ensure clients render the message correctly.
- Don't call `Hook::exec()` to dispatch mail-related hooks from new code; use `HookDispatcherInterface` (see `module-register-hooks`).

## Canonical examples

- [devdocs - Mail templates (concept)](https://devdocs.prestashop-project.org/9/modules/concepts/mail-templates/).
- [devdocs - Mail templates (component reference)](https://devdocs.prestashop-project.org/9/development/components/mail-templates/).
- [`PrestaShop/PrestaShop` - MailTemplateRendererInterface](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/MailTemplate/MailTemplateRendererInterface.php).
- [`PrestaShop/PrestaShop` - core mail themes](https://github.com/PrestaShop/PrestaShop/tree/develop/mails/themes/modern).
- Working sample: [`example-modules/example_module_mailtheme`](https://github.com/PrestaShop/example-modules/tree/master/example_module_mailtheme) (adds a layout, contributes a whole theme, hooks into transformations).
