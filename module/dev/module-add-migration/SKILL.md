---
name: module-add-migration
description: Ship idempotent schema and data migrations with a PrestaShop 9 module via versioned upgrade/upgrade-<version>.php scripts. Use when the user bumps the module version and needs to evolve the database (new column, new table, default-value change) without re-installing the module.
---

## Requirements

Ask the user:
* Target version the upgrade lifts the install to (e.g. `1.2.0`). Bump `$this->version` in the main module class to match.
* What changes: schema (`ALTER TABLE`, `CREATE TABLE`), data (back-fill rows, rename `Configuration` keys), or both.
* Whether the migration is destructive (drops a column, removes a row). Destructive migrations need a backup note in the module changelog.
* Whether the previous version's schema is reliably known. If the module has been in the wild for years, assume the schema is what core tools see RIGHT NOW; check column existence before altering.

## Steps

1. Place each upgrade script under `upgrade/upgrade-<version>.php`. The file MUST expose a function named exactly `upgrade_module_<version_with_underscores>($module)`. PS scans this folder when the merchant clicks **Upgrade** in the BO module manager OR runs `bin/console prestashop:module upgrade <modulename>`, and it executes EVERY script whose version is greater than the currently-installed version, in ascending order:

    ```
    mymodule/
    ├── mymodule.php          # bump $this->version = '1.2.0'
    └── upgrade/
        ├── upgrade-1.1.0.php # function upgrade_module_1_1_0($module)
        ├── upgrade-1.2.0.php # function upgrade_module_1_2_0($module)
        └── upgrade-1.2.1.php # function upgrade_module_1_2_1($module)
    ```

2. Bump `$this->version` to the highest version present under `upgrade/`. If the module class still claims `1.0.0` but you ship `upgrade-1.2.0.php`, the upgrade loop runs but the module manager keeps showing the old version. Always bump in the same commit:

    ```php
    // mymodule/mymodule.php
    public function __construct()
    {
        $this->name = 'mymodule';
        $this->version = '1.2.0'; // matches the highest upgrade-<version>.php
        // ...
    }
    ```

3. Each upgrade function returns `true` on success, `false` to abort the upgrade chain. Keep statements idempotent: `CREATE TABLE IF NOT EXISTS`, `ALTER TABLE ... ADD COLUMN IF NOT EXISTS` (MySQL 8.0+) or a `SHOW COLUMNS` guard. The merchant may run the upgrade twice (e.g. after a partial failure) - the second run must be a no-op:

    ```php
    <?php
    // mymodule/upgrade/upgrade-1.2.0.php
    declare(strict_types=1);

    function upgrade_module_1_2_0(Mymodule $module): bool
    {
        $db = Db::getInstance();
        $table = _DB_PREFIX_ . 'mymod_voucher';

        // Idempotent ADD COLUMN guarded by SHOW COLUMNS (works on MySQL 5.7 too).
        $columns = $db->executeS('SHOW COLUMNS FROM `' . $table . '` LIKE "usage_count"');
        if (empty($columns)) {
            if (!$db->execute('ALTER TABLE `' . $table . '` ADD `usage_count` INT(11) NOT NULL DEFAULT 0')) {
                return false;
            }
        }

        // Idempotent index creation.
        $indexes = $db->executeS('SHOW INDEX FROM `' . $table . '` WHERE Key_name = "idx_mymod_voucher_active"');
        if (empty($indexes)) {
            $db->execute('CREATE INDEX `idx_mymod_voucher_active` ON `' . $table . '` (`active`)');
        }

        // Data back-fill.
        $db->execute('UPDATE `' . $table . '` SET `usage_count` = 0 WHERE `usage_count` IS NULL');

        // Register a new hook introduced in 1.2.0.
        $module->registerHook('actionMymoduleAfterImport');

        return true;
    }
    ```

4. For schema changes that involve Doctrine entities (you also added or modified an attribute under `src/Entity/`), pair the SQL with a Doctrine Migration. Place the migration under `src/Migrations/Version<timestamp>.php` and trigger it from the upgrade script. PrestaShop does NOT run Doctrine migrations automatically for modules - your upgrade script must invoke them, OR the SQL block above must mirror the Doctrine change exactly:

    ```php
    // src/Migrations/Version20250114120000.php
    namespace MyVendor\Mymodule\Migrations;

    use Doctrine\DBAL\Schema\Schema;
    use Doctrine\Migrations\AbstractMigration;

    final class Version20250114120000 extends AbstractMigration
    {
        public function up(Schema $schema): void
        {
            $this->addSql('ALTER TABLE `' . _DB_PREFIX_ . 'mymod_voucher` ADD `usage_count` INT(11) NOT NULL DEFAULT 0');
        }
        public function down(Schema $schema): void
        {
            $this->addSql('ALTER TABLE `' . _DB_PREFIX_ . 'mymod_voucher` DROP COLUMN `usage_count`');
        }
    }
    ```

   Never run `bin/console doctrine:schema:update --force` against a PrestaShop database. The schema-update command generates foreign keys and column changes that conflict with core tables; the Doctrine module documentation warns against it explicitly. Ship explicit migrations or hand-written SQL.

5. Run the upgrade. The merchant has two equivalent paths:

    ```bash
    # CLI - upgrades every enabled module that has a higher version on disk than in DB.
    bin/console prestashop:module upgrade mymodule

    # OR Back Office: Modules > Module Manager > Upgrade button next to the module.
    ```

   Both paths execute every applicable `upgrade-*.php` in version order, then write the new version into `ps_module.version`.

6. Test the upgrade end-to-end every release. The standard recipe:

    ```bash
    git checkout v1.1.0       # previous version
    bin/console prestashop:module install mymodule
    git checkout v1.2.0       # current version
    bin/console prestashop:module upgrade mymodule
    bin/console prestashop:module reset mymodule    # confirm install + uninstall + reinstall still works
    ```

   A common failure is shipping `upgrade-1.2.0.php` but forgetting to bump `$this->version`; the upgrade loop runs but the module manager refuses to advance. Catch this in CI with a one-liner that greps both files.

## Do

- Make every statement idempotent. Guard `ALTER TABLE` with `SHOW COLUMNS`, `CREATE INDEX` with `SHOW INDEX`, `INSERT` with `INSERT ... ON DUPLICATE KEY UPDATE` or a pre-`SELECT`.
- Bump `$this->version` in the same commit as the new `upgrade-<version>.php`.
- Treat each upgrade as a forward-only step. PrestaShop has no built-in "downgrade" path; if you need to revert, ship a NEW upgrade script that rolls forward to the previous behaviour.
- Register any newly-needed hooks (`$module->registerHook(...)`) inside the upgrade script - reinstalling is not an option for an existing merchant.
- Document destructive migrations in the module CHANGELOG with the exact data loss surface.

## Don't

- Don't assume the previous version's schema. The merchant may be on an old patched build; verify with `SHOW COLUMNS` / `SHOW INDEX` before altering.
- Don't run `bin/console doctrine:schema:update --force` against a PrestaShop database (the Doctrine docs forbid it) - it adds foreign keys core does not expect.
- Don't drop `Configuration` keys without a back-fill plan. Use `Configuration::deleteByName()` only after copying the value to its replacement key.
- Don't put `ALTER TABLE ps_product` (or any other core table) in a module migration. Module migrations belong inside the module's namespace tables (`mymod_*`) only.
- Don't forget to bump `$this->version`. The upgrade script is invisible to the module manager otherwise.

## Canonical examples

- [devdocs - Module upgrade](https://devdocs.prestashop-project.org/9/modules/creation/enabling-auto-update/) - the `upgrade/upgrade-<version>.php` contract and the `upgrade_module_<v_w_underscores>` naming rule.
- [devdocs - Doctrine in modules](https://devdocs.prestashop-project.org/9/modules/concepts/doctrine/) - the explicit warning against `doctrine:schema:update --force` and the recommended SQL-install workflow.
- [devdocs - prestashop:module CLI](https://devdocs.prestashop-project.org/9/development/components/console/) - lists `prestashop:module upgrade`, `prestashop:module install`, `prestashop:module reset`.
- [`PrestaShop/example-modules` - demodoctrine](https://github.com/PrestaShop/example-modules/tree/master/demodoctrine) - canonical SQL install + Doctrine entity layout that an upgrade script must keep in sync.
- [`PrestaShop/ps_apiresources`](https://github.com/PrestaShop/ps_apiresources) - real module shipped by core with a tracked version history; read its `composer.json` and module class to see the version-bump discipline.
