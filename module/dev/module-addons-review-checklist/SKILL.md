---
name: module-addons-review-checklist
description: Run through the specific technical requirements that PrestaShop Addons Marketplace reviewers check before approving a module submission - security, structure, performance, translations, and clean uninstall. Use as a pre-submission gate after module-validate passes.
---

## Requirements

Ask the user:
* Module technical name and absolute path to the module root.
* Target PrestaShop version range declared in `ps_versions_compliancy`.
* Whether the module makes any external HTTP calls (API, telemetry, license check).

## Steps

### File structure

1. Verify `logo.png` exists at module root and is exactly 32x32 pixels PNG.

2. Verify every directory contains `index.php` (prevents directory listing):

    ```bash
    find . -type d | while read dir; do
      [ ! -f "$dir/index.php" ] && echo "MISSING: $dir/index.php"
    done
    ```

3. Verify no hidden files (`.DS_Store`, `.env`, `.git`, `.idea`, `.vscode`) are present in the distributable.

4. Verify `composer.json` has `"name"`, `"type": "prestashop-module"`, and PSR-4 `"autoload"` section.

### PHP security - forbidden functions

5. Search for forbidden functions. The Validator rejects any occurrence:

    ```bash
    grep -rn --include='*.php' -E '\b(eval|exec|passthru|system|shell_exec|proc_open|popen)\s*\(' .
    grep -rn --include='*.php' -E '\bcurl_exec\s*\(' .   # allowed only with validated URLs
    grep -rn --include='*.php' -E '\bfile_get_contents\s*\(' .  # forbidden on remote URLs
    ```

   Replace `file_get_contents($url)` with a proper HTTP client (`GuzzleHttp\Client` or `PrestaShop\PrestaShop\Core\Http\Client`).

### SQL security

6. Verify every query uses `pSQL()` or `bqSQL()` for string interpolation, or parameterised queries. No raw concatenation of user input:

    ```php
    // WRONG
    $db->execute("SELECT * FROM table WHERE name = '" . $name . "'");
    // CORRECT
    $db->execute("SELECT * FROM table WHERE name = '" . pSQL($name) . "'");
    ```

7. Search for potential injection vectors:

    ```bash
    grep -rn --include='*.php' 'execute\|executeS\|getValue' . | grep -v 'pSQL\|bqSQL\|(int)'
    ```

### CSRF protection

8. Every form submission must include a CSRF token. For Symfony forms, this is automatic. For legacy `getContent()` forms, verify token validation:

    ```php
    // In form processing
    if (!$this->isTokenValid()) {
        $this->_errors[] = $this->trans('Invalid token', [], 'Modules.Mymodule.Admin');
        return;
    }
    ```

9. For AJAX endpoints in admin controllers, verify `$this->getAdminLink()` generates URLs with tokens, or use `Tools::getAdminTokenLite()` for manual validation.

### XSS protection

10. Every output must be escaped:
    - PHP: `htmlspecialchars($var, ENT_QUOTES, 'UTF-8')`
    - Smarty: `{$var|escape:'html':'UTF-8'}`
    - Twig: `{{ var|e }}` (auto-escaping enabled by default, but verify raw filters are not used)

11. Search for unescaped output:

    ```bash
    grep -rn --include='*.tpl' '{$' . | grep -v '|escape\|nofilter'
    grep -rn --include='*.twig' '|raw' .   # each |raw usage must be justified
    ```

### External calls and GDPR

12. If the module makes HTTP requests to external servers, it MUST document the call in README, require explicit merchant opt-in, and never transmit customer PII without consent. Search:

    ```bash
    grep -rn --include='*.php' -E '(curl_init|GuzzleHttp|HttpClient|file_get_contents\s*\(\s*["\x27]https?)' .
    ```

### Performance

14. Verify: no `SELECT *` (select only needed columns), no queries inside loops, proper indexing on `WHERE` clause columns, and no heavy queries in global hooks (`displayHeader`, `actionFrontControllerSetMedia`) without caching.

    ```bash
    grep -rn --include='*.php' -B5 'execute\|executeS\|getValue' . | grep -E 'foreach|while|for\s*\('
    ```

### Translation completeness

15. Every user-facing string must use a translation function (`$this->trans(...)`, `{l s='...' mod='mymodule'}`, `{{ '...'|trans }}`). Search for hardcoded strings:

    ```bash
    grep -rn --include='*.tpl' --include='*.twig' -E '>[A-Z][a-z]+\s+[a-z]+' . | grep -v '{l\|trans\||escape'
    ```

### Clean uninstall

16. Verify `uninstall()` removes ALL custom tables, Configuration keys, hook registrations, tabs, and overrides. Run the round-trip:

    ```bash
    bin/console prestashop:module install mymodule
    bin/console prestashop:module uninstall mymodule
    mysql -e "SELECT * FROM ps_configuration WHERE name LIKE 'MYMODULE_%';"  # must be empty
    ```

### Compatibility and license

17. Verify `ps_versions_compliancy` matches actually tested versions. Verify a license header in every PHP file (AFL-3.0, MIT, Apache-2.0, or proprietary with copyright). Verify no obfuscated code (ionCube, Zend Guard) and no minified PHP.

## Do

- Run this checklist after `module-validate` passes. The Validator catches most items automatically, but manual review catches edge cases.
- Fix every item before submission. Addons reviewers reject on first failure and the re-review queue adds days.
- Keep a copy of this checklist in your CI pipeline as automated grep checks.

## Don't

- Don't assume `|escape` in Smarty means safe. Verify the escape mode matches the context (html, javascript, url).
- Don't submit with any `grep` hit from the forbidden-functions check, even if the function is "only used for local files."
- Don't claim GDPR compliance by documenting external calls in a file nobody reads. The opt-in must be a BO UI toggle.

## Canonical examples

- [PrestaShop Module Validator](https://validator.prestashop.com/) - the automated checks this skill mirrors.
- [devdocs - Modules / Distribution](https://devdocs.prestashop-project.org/9/modules/sell/) - Addons submission process.
- [devdocs - Modules / Testing / Basic checks](https://devdocs.prestashop-project.org/9/modules/testing/basic-checks/) - install/uninstall round-trip.

## Related skills

- `module-validate` - the full automated validation pipeline (run before this checklist).
- `module-prepare-addons-submission` - the end-to-end preparation workflow.
- `module-package-zip` - build the zip that gets submitted.
