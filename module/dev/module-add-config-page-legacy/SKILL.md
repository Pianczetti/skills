---
name: module-add-config-page-legacy
description: Build a legacy getContent() + HelperForm configuration page for an existing PrestaShop module. Use ONLY when the module must support PrestaShop < 8 or you are maintaining a legacy module that already uses this pattern.
---

> **(legacy)** Use this skill ONLY if the module must support PrestaShop versions older than 8.0 or you are maintaining an existing legacy module. For new modules on PS 9, use [`module-add-config-page-modern`](../module-add-config-page-modern/SKILL.md) instead. Every section below is the legacy path and is intentionally NOT aligned with PS 9 modern conventions.

## Requirements (legacy)

Ask the user:
* The list of fields to expose, plus types compatible with `HelperForm` (`text`, `select`, `switch`, `textarea`, `password`, `radio`).
* The configuration keys, in `UPPER_SNAKE_CASE` and prefixed with the module name (e.g. `MYMODULE_API_KEY`).
* Whether the module needs translations. In legacy mode strings are wrapped with `$this->l(...)` (ALSO legacy - do not introduce this pattern in modern code).
* Whether the form must save per-shop. In legacy code, multistore is handled by `Shop::getContextShopID()` / `Shop::getContextShopGroupID()` passed to `Configuration::get/updateValue`.

## Steps (legacy)

1. **(legacy)** In the main module class, implement `public function getContent(): string`. PrestaShop calls it when the merchant clicks "Configure" on the module:

    ```php
    public function getContent(): string
    {
        $output = '';
        if (Tools::isSubmit('submit'.$this->name)) {
            $apiKey = (string) Tools::getValue('MYMODULE_API_KEY');
            $limit = (int) Tools::getValue('MYMODULE_DEFAULT_LIMIT');

            if ($apiKey === '' || $limit < 1) {
                $output .= $this->displayError($this->l('Invalid configuration values.'));
            } else {
                Configuration::updateValue('MYMODULE_API_KEY', $apiKey);
                Configuration::updateValue('MYMODULE_DEFAULT_LIMIT', $limit);
                $output .= $this->displayConfirmation($this->l('Settings updated.'));
            }
        }

        return $output . $this->renderForm();
    }
    ```

2. **(legacy)** Implement `protected function renderForm(): string` building a `HelperForm`:

    ```php
    protected function renderForm(): string
    {
        $fields_form = [[
            'form' => [
                'legend' => ['title' => $this->l('Settings'), 'icon' => 'icon-cogs'],
                'input' => [
                    ['type' => 'text', 'label' => $this->l('API key'), 'name' => 'MYMODULE_API_KEY', 'required' => true],
                    ['type' => 'text', 'label' => $this->l('Default limit'), 'name' => 'MYMODULE_DEFAULT_LIMIT'],
                ],
                'submit' => ['title' => $this->l('Save')],
            ],
        ]];

        $helper = new HelperForm();
        $helper->module = $this;
        $helper->name_controller = $this->name;
        $helper->table = $this->table;
        $helper->identifier = $this->identifier;
        $helper->submit_action = 'submit'.$this->name;
        $helper->default_form_language = (int) Configuration::get('PS_LANG_DEFAULT');
        $helper->fields_value = [
            'MYMODULE_API_KEY' => Tools::getValue('MYMODULE_API_KEY', Configuration::get('MYMODULE_API_KEY')),
            'MYMODULE_DEFAULT_LIMIT' => Tools::getValue('MYMODULE_DEFAULT_LIMIT', Configuration::get('MYMODULE_DEFAULT_LIMIT')),
        ];
        $helper->currentIndex = AdminController::$currentIndex.'&configure='.$this->name;
        $helper->token = Tools::getAdminTokenLite('AdminModules');

        return $helper->generateForm($fields_form);
    }
    ```

3. **(legacy)** Optional: a custom `views/templates/admin/configure.tpl` only when you need a layout HelperForm cannot produce. If you do, render it from `getContent()` with `$this->display(__FILE__, 'configure.tpl')` and `assign` Smarty variables before returning the HelperForm output.

4. **(legacy)** In `install()`, seed default values: `Configuration::updateValue('MYMODULE_API_KEY', '')`. In `uninstall()`, delete them: `Configuration::deleteByName('MYMODULE_API_KEY')`.

5. **(legacy)** Multistore handling: pass `id_shop_group` / `id_shop` to `Configuration::get/updateValue`. Resolve the current context with `Shop::getContext()`, `Shop::getContextShopGroupID()`, `Shop::getContextShopID()`.

## Do (legacy)

- **(legacy)** Use `Tools::isSubmit('submit<ModuleName>')` to detect form submission and `Tools::getValue('KEY', $default)` to read inputs (it handles slashes and missing keys).
- **(legacy)** Validate every value before calling `Configuration::updateValue`. There is no automatic Symfony validator on this path.
- **(legacy)** Return the HelperForm output from `getContent()`. Do not echo it; PrestaShop wraps the return value in the BO layout.

## Don't (legacy)

- **(legacy)** Don't introduce this pattern in new PS 9 modules. The modern path is `module-add-config-page-modern` (Symfony Form + FormHandler + FormDataProvider).
- **(legacy)** Don't mix `HelperForm` with a modern admin route. The legacy form expects to be rendered by the legacy `AdminModulesController` flow via `getContent()`.
- **(legacy)** Don't call `parent::__construct()` AFTER setting `$this->displayName`. The constructor must follow the canonical order: properties → `parent::__construct()` → translated strings.
- **(legacy)** Don't forget to delete the configuration keys in `uninstall()`. Stale rows in `ps_configuration` accumulate across reinstalls.

## Canonical examples (legacy)

- [devdocs - Adding a configuration page (legacy)](https://devdocs.prestashop-project.org/9/modules/creation/adding-configuration-page/).
- For migration guidance toward the modern path: [devdocs - Migration guide: settings forms](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/forms/settings-forms/).
