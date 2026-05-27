---
name: module-prepare-addons-submission
description: Take a development-stage PrestaShop 9 module to a production-ready state compliant with PrestaShop Addons Marketplace requirements - metadata, security, file structure, translations, uninstall cleanliness, and Validator pass. Use when the user wants to submit a module to the Addons Marketplace for the first time.
---

## Requirements

Ask the user:
* Module technical name and absolute path to the module root.
* Author name and company (used in `$this->author` and copyright headers).
* License type (AFL-3.0 for open-source, proprietary header for paid modules).
* The exact PrestaShop version range the module was tested on (e.g. `9.0.0` to `9.1.99`).
* Whether the module phones home (API calls, telemetry). GDPR requires explicit merchant opt-in.

## Steps

1. Verify and fix module class metadata in `<modulename>.php`. Every field below is mandatory for Addons acceptance:

    ```php
    public function __construct()
    {
        $this->name = 'mymodule';
        $this->tab = 'front_office_features';       // valid tab from Addons taxonomy
        $this->version = '1.0.0';
        $this->author = 'My Company';
        $this->need_instance = 0;
        $this->ps_versions_compliancy = ['min' => '9.0.0', 'max' => '9.1.99'];

        parent::__construct();

        $this->displayName = $this->trans('My Module', [], 'Modules.Mymodule.Admin');  // max 56 chars
        $this->description = $this->trans('Short description of...', [], 'Modules.Mymodule.Admin');  // max 200 chars
        $this->confirmUninstall = $this->trans('Are you sure?', [], 'Modules.Mymodule.Admin');
    }
    ```

2. Ensure every PHP file has a license header block at the top (after `<?php` and `declare(strict_types=1)`). Use the project's chosen license (AFL-3.0 or proprietary). The Validator rejects files without headers.

3. Validate file structure. Every directory (including nested `vendor/` subdirs) must contain `index.php`. The module root must have `logo.png` at exactly 32x32 pixels:

    ```bash
    # Generate missing index.php placeholders
    find . -type d -exec sh -c '
      if [ ! -f "$1/index.php" ]; then
        printf "<?php\nheader(\"Location: ../\");\nexit;\n" > "$1/index.php"
      fi
    ' _ {} \;

    # Verify logo dimensions
    identify logo.png   # must be 32x32 PNG
    ```

4. Enforce PSR-4 autoloading under `src/`. In `composer.json`:

    ```json
    {
      "name": "myvendor/mymodule",
      "type": "prestashop-module",
      "autoload": {
        "psr-4": {
          "MyVendor\\MyModule\\": "src/"
        }
      }
    }
    ```

5. Audit security. Search the entire codebase for forbidden patterns and fix each one:

    ```bash
    # Forbidden functions (Validator rejects these)
    grep -rn 'eval\s*(' --include='*.php' .
    grep -rn '\b\(exec\|passthru\|system\|shell_exec\|proc_open\|popen\)\s*(' --include='*.php' .
    grep -rn 'file_get_contents\s*(' --include='*.php' .  # allowed only on local paths

    # Raw superglobals (must use Tools::getValue() / Tools::getIsset())
    grep -rn '\$_GET\|\$_POST\|\$_REQUEST' --include='*.php' .

    # SQL injection: every variable in SQL must pass through pSQL() or bqSQL()
    grep -rn 'execute\|executeS' --include='*.php' . | grep -v 'pSQL\|bqSQL'
    ```

   Replace `$_GET['key']` with `Tools::getValue('key')`. Wrap all user input in SQL with `pSQL()`. Escape all output with `htmlspecialchars()` in PHP, `|escape:'html':'UTF-8'` in Smarty, `|e` in Twig.

6. Ensure CSRF protection on every form. Modern Symfony forms get tokens automatically. For legacy `getContent()` pages, validate with:

    ```php
    if (!$this->isTokenValid()) {
        // reject the submission
    }
    ```

7. Verify translation readiness. Every user-facing string must use the Symfony Translator with domain `Modules.Mymodule.Admin` or `Modules.Mymodule.Shop`:

    ```php
    // In controllers / module class
    $this->trans('Save settings', [], 'Modules.Mymodule.Admin');
    // In Twig
    {{ 'Save settings'|trans({}, 'Modules.Mymodule.Admin') }}
    // In Smarty
    {l s='Save settings' mod='mymodule'}
    ```

8. Verify `uninstall()` removes ALL module state. The round-trip must leave zero residual rows:

    ```php
    public function uninstall(): bool
    {
        return parent::uninstall()
            && Db::getInstance()->execute('DROP TABLE IF EXISTS `' . _DB_PREFIX_ . 'mymod_table`')
            && Configuration::deleteByName('MYMODULE_SETTING_1')
            && Configuration::deleteByName('MYMODULE_SETTING_2')
            && $this->uninstallTab();
    }
    ```

9. Check performance. No queries on every `hookDisplayHeader` or `hookActionFrontControllerSetMedia` without caching. Lazy-load CSS/JS only on pages where the module is active. Avoid `SELECT *`; select only needed columns.

10. Build the zip with `module-package-zip`, then upload to [validator.prestashop.com](https://validator.prestashop.com/). Fix every error and warning. Re-run until the Validator reports zero issues.

## Do

- Set `$this->need_instance = 0` unless the module genuinely needs the database at class-load time.
- Keep `displayName` under 56 characters and `description` under 200 characters (Addons truncates beyond that).
- Ship a README.md with install instructions, configuration steps and known limitations.
- Test `install()` and `uninstall()` on a completely fresh shop before submission.

## Don't

- Don't use `file_get_contents()` on remote URLs. Use `PrestaShop\PrestaShop\Core\Http\Client` or Guzzle with proper timeout and error handling.
- Don't ship obfuscated or minified PHP (ionCube, Zend Guard). The Validator rejects it.
- Don't leave `var_dump()`, `print_r()` or `error_log()` debug statements in production code.
- Don't hardcode `ps_` as the table prefix. Always use `_DB_PREFIX_`.
- Don't phone home without explicit merchant consent (GDPR compliance is a hard requirement).

## Canonical examples

- [devdocs - Modules / Distribution](https://devdocs.prestashop-project.org/9/modules/sell/) - Addons Marketplace publishing guide.
- [devdocs - Modules / Concepts / Configuration](https://devdocs.prestashop-project.org/9/modules/concepts/configuration/) - proper Configuration usage.
- [PrestaShop Module Validator](https://validator.prestashop.com/) - the official checks the Addons team runs.
- [`PrestaShop/example-modules`](https://github.com/PrestaShop/example-modules) - reference implementations passing all Validator rules.

## Related skills

- `module-validate` - the full validation pipeline (linters, round-trip, Validator).
- `module-package-zip` - build the distributable zip for submission.
- `module-add-translations-new` - wire translations before submission.
- `module-addons-review-checklist` - the specific checks Addons reviewers apply.
