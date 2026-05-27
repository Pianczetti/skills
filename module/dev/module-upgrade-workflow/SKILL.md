---
name: module-upgrade-workflow
description: Ship a complete module version upgrade to merchants who already have the module installed - version bump, upgrade scripts, hook/config changes, changelog, and Addons re-submission. Use when the user wants to release a new version of an existing module.
---

## Requirements

Ask the user:
* Module technical name and current `$this->version`.
* New target version following semver (MAJOR.MINOR.PATCH).
* What changed: schema evolution, new hooks, new/removed Configuration keys, new dependencies, or cosmetic-only changes.
* Whether the module is distributed via the Addons Marketplace (triggers a review cycle) or self-hosted.

## Steps

1. Determine version bump type using semver:
   - **MAJOR** (2.0.0): breaking changes - removed hooks, renamed Configuration keys, dropped PHP/PS version support, changed public API.
   - **MINOR** (1.1.0): new features - new hooks, new admin pages, new DB tables/columns, new Configuration keys.
   - **PATCH** (1.0.1): bug fixes and cosmetic changes - template fixes, translation updates, CSS adjustments.

2. Bump `$this->version` in `<modulename>.php`. This is the trigger that makes PrestaShop detect an available upgrade:

    ```php
    $this->version = '1.2.0'; // must match the highest upgrade script version
    ```

3. Decide whether an upgrade script is needed. Create `upgrade/upgrade-<version>.php` if ANY of these changed:
   - Database schema (new/altered tables or columns)
   - Hook registrations (new hooks the module needs)
   - Configuration keys (new defaults, renamed keys)
   - Tabs (new admin menu entries)

   Skip the upgrade script only for cosmetic changes (templates, CSS, JS, translations, bug fixes in existing PHP logic).

4. Write the upgrade script. The function name uses underscores for dots in the version:

    ```php
    <?php
    // upgrade/upgrade-1.2.0.php
    declare(strict_types=1);

    function upgrade_module_1_2_0(Mymodule $module): bool
    {
        // Register new hooks
        $module->registerHook('actionMymoduleNewEvent');

        // Add new Configuration with default
        Configuration::updateValue('MYMODULE_NEW_SETTING', '1');

        // Schema changes (idempotent)
        $db = Db::getInstance();
        $columns = $db->executeS('SHOW COLUMNS FROM `' . _DB_PREFIX_ . 'mymod_table` LIKE "new_col"');
        if (empty($columns)) {
            $db->execute('ALTER TABLE `' . _DB_PREFIX_ . 'mymod_table` ADD `new_col` VARCHAR(255) DEFAULT NULL');
        }

        return true;
    }
    ```

5. Understand the upgrade discovery mechanism. When a merchant clicks **Upgrade** in the BO Module Manager or runs `bin/console prestashop:module upgrade mymodule`, PrestaShop:
   - Reads the installed version from `ps_module.version` (e.g. `1.0.0`).
   - Scans `upgrade/` for all `upgrade-<version>.php` files where version > installed version.
   - Sorts them in ascending version order.
   - Executes each function sequentially. If any returns `false`, the chain stops.
   - Writes the new `$this->version` to `ps_module.version` on success.

6. For chained upgrades (merchant jumping from 1.0.0 to 1.3.0), ensure each intermediate script is self-contained. A merchant on 1.0.0 will execute `upgrade-1.1.0.php`, then `upgrade-1.2.0.php`, then `upgrade-1.3.0.php` in order. Never assume which prior scripts ran; use idempotent guards.

7. Update `install()` to include all cumulative state. A fresh install on the latest version must produce the same result as upgrading through every version. If `upgrade-1.2.0.php` adds a column, `install()` must also create that column.

8. Test the upgrade path end-to-end:

    ```bash
    # Install old version
    git checkout v1.0.0
    bin/console prestashop:module install mymodule

    # Upgrade to new version
    git checkout v1.2.0
    bin/console prestashop:module upgrade mymodule

    # Verify: version in DB matches, new hooks registered, schema correct
    mysql -e "SELECT version FROM ps_module WHERE name='mymodule';"
    ```

9. Update `CHANGELOG.md` with the new version entry. Include: version number, date, list of changes grouped by Added/Changed/Fixed/Removed, and note any breaking changes for MAJOR bumps.

10. For Addons Marketplace distribution: rebuild the zip with `module-package-zip`, re-run the Validator, then submit the new version through the seller back-office. The Addons team reviews it; once approved, merchants see the update notification in their BO Module Manager automatically.

## Do

- Always bump `$this->version` and ship the corresponding `upgrade-<version>.php` in the same commit.
- Keep upgrade scripts idempotent: `SHOW COLUMNS` guard before `ALTER TABLE`, `INSERT IGNORE` or `ON DUPLICATE KEY UPDATE` for data.
- Mirror every upgrade-script change in `install()` so fresh installs are equivalent to upgraded installs.
- Test the full upgrade chain: install oldest supported version, then upgrade directly to the latest.
- Document breaking changes prominently in CHANGELOG.md and in the Addons product description.

## Don't

- Don't bump `$this->version` without an upgrade script if the version introduces schema, hook, or Configuration changes. The upgrade silently skips and merchants get a broken module.
- Don't assume the previous version's state. Always guard with `SHOW COLUMNS`, `SHOW INDEX`, `Configuration::hasKey()`.
- Don't put logic in upgrade scripts that depends on the module being enabled (hooks fire only after the upgrade completes).
- Don't remove a Configuration key without first migrating its value to the replacement key in the upgrade script.
- Don't skip intermediate versions. If you released 1.1.0 and 1.2.0, merchants may be on either. Both upgrade scripts must exist.

## Canonical examples

- [devdocs - Modules / Enabling auto-update](https://devdocs.prestashop-project.org/9/modules/creation/enabling-auto-update/) - the `upgrade/upgrade-<version>.php` contract.
- [devdocs - prestashop:module CLI](https://devdocs.prestashop-project.org/9/development/components/console/prestashop-module/) - `install`, `upgrade`, `reset` subcommands.
- [`PrestaShop/example-modules`](https://github.com/PrestaShop/example-modules) - real modules with versioned upgrade scripts.

## Related skills

- `module-add-migration` - create a single upgrade script (this skill covers the full lifecycle).
- `module-database-migration-patterns` - patterns for schema changes inside upgrade scripts.
- `module-database-rollback-strategy` - handling failures during upgrades.
- `module-bump-compatibility` - when the upgrade also targets a new PS version.
- `module-package-zip` - build the zip for Addons re-submission.
- `module-validate` - full validation before releasing the upgrade.
