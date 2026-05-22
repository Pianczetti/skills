---
name: module-package-zip
description: Build a clean, Marketplace-ready zip for a PrestaShop 9 module - no dev dependencies, no test artefacts, no dotfiles, with the correct `index.php` placeholders. Use right before publishing on the Marketplace, attaching to a GitHub release, or deploying to a managed shop.
---

## Requirements

Ask the user:
* Module technical name and module root (the folder that will become the zip's top-level directory). The zip's root entry MUST equal the technical name; the BO installer rejects mismatches.
* `$this->version` value from `<modulename>.php`. The zip is conventionally named `<modulename>-<version>.zip` or simply `<modulename>.zip` for a release artifact.
* Target environment: public Marketplace (mandatory full strip plus the official Validator pass) vs. internal deployment (same strip; Validator optional).
* Whether the build runs from a clean checkout (recommended) or from the working tree (risk of leaking local modifications, IDE state, untracked files).
* Whether any additional folders should be excluded beyond the standard list (e.g. `docs/`, `_demo/`, custom internal scripts).

## Steps

1. Start from a clean checkout into a scratch directory. Building from the working tree leaks local IDE state, untracked files, and previously-installed `node_modules`:

    ```bash
    git clone --depth 1 https://github.com/myvendor/mymodule.git /tmp/build/mymodule
    cd /tmp/build/mymodule
    ```

2. Install production-only dependencies and lock the autoloader. `--classmap-authoritative` skips filesystem checks at runtime, which is a measurable performance gain on shared hosting:

    ```bash
    composer install \
      --no-dev \
      --optimize-autoloader \
      --classmap-authoritative \
      --no-progress \
      --no-interaction
    ```

3. Strip everything that has no business in a published module. Match the canonical exclusion list - the Validator flags most of these explicitly:

    ```bash
    rm -rf _dev tests node_modules \
           .git .github .gitignore .gitattributes \
           .idea .vscode .editorconfig \
           .php-cs-fixer.php .php-cs-fixer.dist.php .php-cs-fixer.cache \
           phpunit.xml.dist phpunit.xml .phpunit.cache \
           phpstan.neon phpstan.neon.dist psalm.xml psalm.xml.dist \
           Makefile docker-compose.yml
    ```

   Keep `composer.json` and `composer.lock` (the Validator parses them), `vendor/` (with `--no-dev` already applied), `config/`, `src/`, `views/`, `controllers/`, `mails/`, `translations/`, `upgrade/`, `sql/`, and the main `<modulename>.php`.

4. Generate the PrestaShop `index.php` placeholders. Every directory under the module (and inside `vendor/`) MUST contain an `index.php` so that no directory listing is exposed if the host's `mod_autoindex` is enabled:

    ```bash
    find . -type d -exec sh -c 'test -f "$1/index.php" || cp /dev/null "$1/index.php"' _ {} \;
    ```

   Or copy a stock placeholder (`<?php header("Location: ../"); exit;`) into each missing slot if the project ships one in `_dev/index.php.dist`.

5. Zip from one level above the module root so the zip's top-level entry matches the module technical name. The BO installer reads that top-level entry as the module identifier:

    ```bash
    cd /tmp/build
    zip -r mymodule-${VERSION:-1.0.0}.zip mymodule \
        -x 'mymodule/.git/*' 'mymodule/tests/*' 'mymodule/_dev/*'   # belt-and-braces
    ```

6. Verify the archive. The first listed entry must be `mymodule/`, no test files should appear, and `vendor/` must be present without `phpunit/` or `symfony/var-dumper/`:

    ```bash
    unzip -l mymodule-1.0.0.zip | head -50
    unzip -l mymodule-1.0.0.zip | grep -E '/(tests|_dev|node_modules|\.git)/'   # must be empty
    unzip -l mymodule-1.0.0.zip | grep -E 'vendor/(phpunit|squizlabs|friendsofphp)/' # must be empty
    ```

7. Smoke-test the zip in a fresh shop. Upload via **Module Manager > Upload a module**, click **Install**, exercise the BO config page and any front-office hook, then uninstall. Confirm the round-trip leaves zero rows in `ps_module` / `ps_hook_module` (see `module-validate`).

8. For Marketplace submission, upload the same zip to [validator.prestashop.com](https://validator.prestashop.com/) and address every error before opening the seller-back-office submission. The Validator runs the structural, security and conformity checks the local linters do not.

## Do

- Optimise the autoloader at packaging time: `composer install --no-dev --optimize-autoloader --classmap-authoritative`. The classmap eliminates filesystem stats on every class load.
- Match the standard exclusion list: `_dev/`, `tests/`, `phpunit.xml.dist`, `.git*`, `.idea`, `.vscode`, `.editorconfig`, `.php-cs-fixer.*`, `node_modules/`, plus any internal scripts. The Validator rejects most of these explicitly.
- Generate `index.php` placeholders in every directory after stripping, including inside `vendor/`. PrestaShop expects them to block directory listing.
- Make the zip's top-level folder name equal to the module technical name. The BO installer uses that name as the module identifier; mismatches cause "module not found" errors after install.
- Tag the release in git and name the zip `<modulename>-<version>.zip` so support and rollback can identify the artifact at a glance.

## Don't

- Don't ship `vendor/bin/`, `tests/`, `phpunit.xml.dist`, `.git*` or any IDE folder. Even if PHP ignores them at runtime, the Validator and human reviewers do not.
- Don't ship dev dependencies. Run `composer install --no-dev` before zipping; PHPUnit, php-cs-fixer and PHPStan have no business inside a deployed module.
- Don't zip the working tree without a clean checkout. Untracked files, half-applied changes and previously-installed `node_modules` all leak into the artifact.
- Don't rename the top-level directory inside the zip to anything other than the module technical name. The installer rejects mismatched names with a confusing error.
- Don't omit the `index.php` placeholders in subdirectories. Hosts with `mod_autoindex` enabled will expose your folder structure; the Validator flags this as a security warning.
- Don't ship `composer.json` without also shipping `vendor/`. Production hosts may not have Composer available, and PrestaShop will not run `composer install` for you.

## Canonical examples

- [devdocs - Modules / Composer](https://devdocs.prestashop-project.org/9/modules/concepts/composer/) - the autoloader contract every module follows.
- [devdocs - Modules / Testing / Basic checks](https://devdocs.prestashop-project.org/9/modules/testing/basic-checks/) - the install/uninstall smoke test you run after building the zip.
- [devdocs - Modules / Testing / CI-CD](https://devdocs.prestashop-project.org/9/modules/testing/ci-cd/) - end-to-end GitHub Actions workflow that builds, validates and publishes a zip.
- [PrestaShop Module Validator](https://validator.prestashop.com/) - the public service that runs Marketplace conformity checks on the zip.
- [devdocs - Modules / Sample modules](https://devdocs.prestashop-project.org/9/modules/sample-modules/) and [`PrestaShop/example-modules`](https://github.com/PrestaShop/example-modules) - reference release scripts and `Makefile` patterns.
- [Composer - autoloader optimisation](https://getcomposer.org/doc/articles/autoloader-optimization.md) - explains `--optimize-autoloader` and `--classmap-authoritative` in detail.

## Related skills

- `module-validate` - the lint, install/uninstall and Validator pipeline you run on the zip before publishing.
- `module-add-unit-tests` - keeps `tests/` clean so the strip step has no nasty surprises.
- `module-add-migration` - upgrade scripts must stay inside `upgrade/`, which the zip ships.
- `module-bump-compatibility` - re-run the package step after bumping `ps_versions_compliancy` and `$this->version`.
