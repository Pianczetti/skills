---
name: module-database-rollback-strategy
description: Handle rollbacks and failed migrations in PrestaShop 9 modules - backup tables, transaction wrapping, two-phase migrations, feature flags, and manual recovery. Use when the user needs to make destructive schema changes reversible or recover from a failed upgrade.
---

## Requirements

Ask the user:
* Module technical name and current version.
* Which upgrade script failed or which destructive operation needs a rollback strategy.
* Whether the affected tables are under 100k rows (single-pass safe) or larger (needs chunking).
* Whether the module is distributed via Addons (merchants cannot SSH) or self-hosted (manual recovery possible).

## Steps

### Understanding the limitation

1. PrestaShop has NO automatic rollback for module upgrades. If `upgrade-<version>.php` returns `false`, the chain stops but already-executed SQL statements within that script are NOT reversed. The module version in `ps_module` stays at the last successfully completed upgrade.

### Pattern: try/catch with manual reversal

2. Wrap destructive operations in a try/catch. If a later statement fails, reverse the earlier ones before returning `false`:

    ```php
    function upgrade_module_1_3_0(Mymodule $module): bool
    {
        $db = Db::getInstance();
        $prefix = _DB_PREFIX_;

        // Step 1: add column
        $step1 = $db->execute("ALTER TABLE `{$prefix}mymod_order` ADD `risk_score` INT(11) DEFAULT NULL");
        if (!$step1) {
            return false;
        }

        // Step 2: add index
        $step2 = $db->execute("CREATE INDEX `idx_mymod_order_risk` ON `{$prefix}mymod_order` (`risk_score`)");
        if (!$step2) {
            // Reverse step 1
            $db->execute("ALTER TABLE `{$prefix}mymod_order` DROP COLUMN `risk_score`");
            return false;
        }

        return true;
    }
    ```

### Pattern: backup table before destructive operation

3. Before dropping or heavily altering a table, create a backup copy:

    ```php
    $db->execute("CREATE TABLE `{$prefix}mymod_legacy_backup_{$version}` AS SELECT * FROM `{$prefix}mymod_legacy`");
    $db->execute("DROP TABLE `{$prefix}mymod_legacy`");
    ```

   Document in CHANGELOG.md that the backup table exists. Clean it up in the next MAJOR version.

### Pattern: two-phase migration

4. Split destructive changes across two versions:
   - **Phase 1 (version N)**: add new structures (non-breaking), update code to write to both old and new.
   - **Phase 2 (version N+1)**: remove old structures after Phase 1 is confirmed stable.

    ```php
    // upgrade-1.3.0.php (Phase 1: additive, non-breaking)
    function upgrade_module_1_3_0(Mymodule $module): bool
    {
        $db = Db::getInstance();
        $db->execute("ALTER TABLE `" . _DB_PREFIX_ . "mymod_order` ADD `new_status` VARCHAR(32) DEFAULT NULL");
        $db->execute("UPDATE `" . _DB_PREFIX_ . "mymod_order` SET `new_status` = `old_status`");
        return true;
    }

    // upgrade-1.4.0.php (Phase 2: remove old after 1.3.0 is stable)
    function upgrade_module_1_4_0(Mymodule $module): bool
    {
        Db::getInstance()->execute("ALTER TABLE `" . _DB_PREFIX_ . "mymod_order` DROP COLUMN `old_status`");
        return true;
    }
    ```

### Pattern: transactions for DML (data changes)

5. Use transactions for data manipulation (INSERT, UPDATE, DELETE). DDL statements (ALTER TABLE, CREATE TABLE, DROP TABLE) auto-commit in MySQL/MariaDB and cannot be rolled back:

    ```php
    $db->execute('START TRANSACTION');
    try {
        $db->execute("UPDATE `{$prefix}mymod_config` SET `value` = 'new' WHERE `key` = 'format'");
        $db->execute("INSERT INTO `{$prefix}mymod_log` (`action`) VALUES ('migrated_format')");
        $db->execute('COMMIT');
    } catch (\Exception $e) {
        $db->execute('ROLLBACK');
        return false;
    }
    ```

### Pattern: feature-flag the new code path

6. Gate new behaviour behind a Configuration flag so merchants can revert to old behaviour without a full module downgrade:

    ```php
    // In upgrade script
    Configuration::updateValue('MYMODULE_USE_NEW_ENGINE', '0'); // off by default

    // In runtime code
    if (Configuration::get('MYMODULE_USE_NEW_ENGINE')) {
        // new code path
    } else {
        // legacy code path (safe fallback)
    }
    ```

   Merchants enable the flag after verifying the upgrade works. Remove the flag and legacy path in the next MAJOR version.

### Recovery: when a migration fails mid-way

7. If a merchant reports a broken state after a failed upgrade: check `ps_module.version` to see where the chain stopped, fix the schema to match what the failed script expected (or restore from backup table), reset the version to retry, then re-run:

    ```bash
    mysql -e "UPDATE ps_module SET version = '1.2.0' WHERE name = 'mymodule';"
    bin/console prestashop:module upgrade mymodule
    ```

8. Document recovery steps in the module README under a "Troubleshooting" section. For Addons-distributed modules where merchants cannot SSH, provide a BO-accessible diagnostic page.

## Do

- Back up tables before DROP or destructive ALTER operations.
- Use two-phase migrations for any change that removes data or structures.
- Wrap pure DML (INSERT/UPDATE/DELETE) in transactions for atomicity.
- Feature-flag risky new code paths so merchants have a rollback lever.

## Don't

- Don't assume DDL (ALTER, CREATE, DROP) can be rolled back. They auto-commit in MySQL/MariaDB.
- Don't drop a table or column without a backup copy in the same upgrade script.
- Don't leave backup tables forever. Remove them in a future MAJOR version.
- Don't rely solely on `return false`. Already-executed statements in the same script are permanent.
- Don't ship Phase 2 of a two-phase migration in the same release as Phase 1.

## Canonical examples

- [devdocs - Modules / Enabling auto-update](https://devdocs.prestashop-project.org/9/modules/creation/enabling-auto-update/) - upgrade script contract and return-false behaviour.
- [devdocs - Modules / Doctrine](https://devdocs.prestashop-project.org/9/modules/concepts/doctrine/) - Doctrine migration patterns.
- [MySQL docs - DDL implicit commit](https://dev.mysql.com/doc/refman/8.0/en/implicit-commit.html) - explains why DDL cannot be rolled back.

## Related skills

- `module-add-migration` - create a single upgrade script.
- `module-database-migration-patterns` - idempotent patterns for each schema change type.
- `module-upgrade-workflow` - the full version release lifecycle.
- `module-add-feature-flag` - register a feature flag for gating new code paths.
