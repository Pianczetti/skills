---
name: module-validate
description: Run the full pre-publication validation pipeline against a PrestaShop 9 module - composer manifest, install/uninstall round-trip, naming and legacy-link linters, plus the official Marketplace validator. Use before opening a release PR or submitting to the Addons Marketplace.
---

## Requirements

Ask the user:
* Module technical name (matches the folder under `modules/`) and the absolute path to the PrestaShop root.
* Target PrestaShop version (e.g. `9.0.x`, `9.1.x`). The validator and linters run against the installed core, so the host must already be on that version.
* Whether the module ships its own `composer.json`. Most modern modules do; a strict composer manifest is mandatory for Marketplace submission.
* Whether a fresh shop is available for the install/uninstall round-trip. Validation must run against a database where the module is NOT already installed; otherwise the install assertion is meaningless.
* Whether the module is destined for the public Marketplace (in which case the official Validator must pass) or a private/internal deployment (in which case the linters and round-trip suffice).

## Steps

1. From the module folder, validate the Composer manifest. `--strict` upgrades any warning (missing `description`, malformed `version`, etc.) to an error:

    ```bash
    cd modules/mymodule
    composer validate --strict
    composer install --no-dev --optimize-autoloader   # what the published zip will ship
    ```

2. From the PrestaShop root, run the install/uninstall round-trip via the modern console command. Both invocations must return a zero exit code, and the database must end with **zero rows** for the module in `ps_module` and `ps_hook_module`:

    ```bash
    cd /path/to/prestashop
    bin/console prestashop:module install   mymodule
    bin/console prestashop:module uninstall mymodule

    # Round-trip assertion (replace the prefix if needed)
    mysql -uroot -proot prestashop -e \
      "SELECT COUNT(*) AS module_rows FROM ps_module WHERE name='mymodule';
       SELECT COUNT(*) AS hook_rows   FROM ps_hook_module hm
         JOIN ps_module m ON m.id_module = hm.id_module WHERE m.name='mymodule';"
    ```

   Any non-zero count means `uninstall()` is leaking state. Fix it before shipping; the Marketplace validator will reject it otherwise.

3. Run the in-tree linters from the PrestaShop root, scoping each to the module path. They scan controllers, routes, templates and translation domains:

    ```bash
    cd /path/to/prestashop
    bin/console prestashop:linter:naming-convention modules/mymodule
    bin/console prestashop:linter:legacy-link       modules/mymodule
    ```

   * `naming-convention` flags class/file names that drift from PSR-4 and the project naming guide.
   * `legacy-link` finds `index.php?controller=...&token=...` style URLs that must be migrated to Symfony routes (`url('admin_...')`). Each hit is a TODO for the next bump-compat pass; see `module-bump-compatibility`.

4. If the module ships PHPStan, Psalm or `php-cs-fixer` configurations, run them too. These are project-level standards and have no global default; pick the same versions the host PrestaShop CI uses to avoid version drift:

    ```bash
    vendor/bin/phpstan analyse --configuration=phpstan.neon --memory-limit=2G
    vendor/bin/php-cs-fixer fix --dry-run --diff
    ```

5. Run the unit tests. They're documented in `module-add-unit-tests`:

    ```bash
    cd modules/mymodule
    vendor/bin/phpunit --testsuite=unit
    ```

6. For Marketplace submission, run the official PrestaShop Validator. It performs structural, security and Marketplace-conformity checks that the local linters do not (forbidden functions, missing `index.php` in subdirectories, copyright header, `ps_versions_compliancy`, ...):

    * Build the distributable zip first with `module-package-zip`.
    * Upload at [validator.prestashop.com](https://validator.prestashop.com/).
    * Address every error and warning. Errors are blockers; warnings are best-effort but Addons reviewers usually require them fixed too.

7. Final sanity: install the freshly-built zip into a clean shop via the BO Module Manager, click through the configuration page, exercise the front-office hook output, then uninstall. The round-trip must again leave zero rows in `ps_module` and `ps_hook_module`.

## Do

- Run the install/uninstall round-trip on a database where the module is NOT yet installed. The first install is the only one that exercises the migration path.
- Treat any leftover row in `ps_module`, `ps_hook_module` or the module's own tables after `uninstall` as a release blocker. Fix `uninstall()`, drop the leftover SQL in `sql/uninstall.sql`, then re-run the round-trip.
- Pin the PHPStan / php-cs-fixer / PHPUnit versions to the same major as the host PrestaShop CI workflow uses. Version drift creates noisy diffs on legitimate code.
- Re-run `prestashop:linter:legacy-link` on every PS minor bump. Each new minor migrates more legacy URLs to Symfony routes; old `index.php?controller=...` calls eventually break.
- Validate against the real target PS version. A 9.0 install will not catch issues that surface only on 9.1.

## Don't

- Don't skip `composer validate --strict`. Marketplace submission rejects malformed manifests.
- Don't run validation on a database where the module was already installed by a previous run. The install path is the one that breaks; only a clean DB exercises it.
- Don't suppress linter findings without addressing them. `# phpcs:disable` and friends are review smells; fix the underlying issue or document why the rule does not apply.
- Don't trust the local linters as a substitute for the official Validator. The Validator catches Marketplace-specific rules (forbidden functions, `index.php` placeholders, copyright header) that the linters do not.
- Don't ship the module's `tests/`, `_dev/`, `phpunit.xml.dist` or `.git*` files. Strip them at packaging time (see `module-package-zip`); the Validator complains otherwise.

## Canonical examples

- [devdocs - Modules / Testing](https://devdocs.prestashop-project.org/9/modules/testing/) - the umbrella page covering every check below.
- [devdocs - Modules / Basic checks](https://devdocs.prestashop-project.org/9/modules/testing/basic-checks/) - canonical install/uninstall round-trip pattern, the round-trip is also where the `ps_module` / `ps_hook_module` zero-row rule comes from.
- [devdocs - Modules / Advanced checks](https://devdocs.prestashop-project.org/9/modules/testing/advanced-checks/) - linters and static-analysis recommendations.
- [devdocs - Modules / CI-CD](https://devdocs.prestashop-project.org/9/modules/testing/ci-cd/) - GitHub Actions templates wiring all of the above together.
- [PrestaShop Module Validator](https://validator.prestashop.com/) - the official service used by Addons reviewers; same checks the Marketplace runs.
- [devdocs - prestashop:module CLI](https://devdocs.prestashop-project.org/9/development/components/console/prestashop-module/) - reference for the install / uninstall / upgrade subcommands.
- [`PrestaShop/PrestaShop` - ModuleCommand](https://github.com/PrestaShop/PrestaShop/blob/develop/src/PrestaShopBundle/Command/ModuleCommand.php) - source for the console command, useful when debugging an exit code.

## Related skills

- `module-add-unit-tests` - PHPUnit suite for handlers and value objects, complementary to install/uninstall validation.
- `module-package-zip` - produces the distributable zip the Validator consumes.
- `module-bump-compatibility` - re-run this skill after bumping `ps_versions_compliancy`.
- `module-add-migration` - upgrade scripts whose `uninstall()` symmetry the round-trip exercises.
