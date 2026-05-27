---
name: module-database-migration-patterns
description: Apply idempotent database schema evolution patterns specific to PrestaShop 9 modules - adding/removing/renaming columns, multi-lang and multi-shop tables, large table migrations, and index management. Use when the user needs to evolve a module's database schema safely across upgrades.
---

## Requirements

Ask the user:
* Module technical name and the table(s) being modified (`_DB_PREFIX_` convention).
* The specific schema change needed (add/remove/rename column, add/remove table, add index, change type, multi-lang, multi-shop).
* Whether the table has more than 100k rows (requires chunked migration).
* Current and new module version.

## Steps

### Adding a column

1. Guard with `SHOW COLUMNS` for idempotency. Provide a `DEFAULT` so existing rows are valid. Mirror in `install()`:

    ```php
    $columns = $db->executeS('SHOW COLUMNS FROM `' . _DB_PREFIX_ . 'mymod_order` LIKE "priority"');
    if (empty($columns)) {
        $db->execute('ALTER TABLE `' . _DB_PREFIX_ . 'mymod_order` ADD `priority` TINYINT(1) NOT NULL DEFAULT 0');
    }
    // Back-fill if default is insufficient
    $db->execute('UPDATE `' . _DB_PREFIX_ . 'mymod_order` SET `priority` = 1 WHERE `status` = "urgent"');
    ```

### Removing a column

2. Never remove a column in the same version that stops reading it. Two-version approach:
   - **Version N**: stop writing/reading the column. Rename to `_deprecated_<name>` as soft-delete.
   - **Version N+1** (next MAJOR): drop the column.

    ```php
    // Version N: soft-delete
    $db->execute('ALTER TABLE `' . _DB_PREFIX_ . 'mymod_order` CHANGE `old_col` `_deprecated_old_col` VARCHAR(255)');
    // Version N+1: hard drop
    $db->execute('ALTER TABLE `' . _DB_PREFIX_ . 'mymod_order` DROP COLUMN `_deprecated_old_col`');
    ```

### Renaming a column

3. Treat as add-new + copy + drop-old across two versions:

    ```php
    // Version 1.2.0: add new column and copy data
    $db->execute('ALTER TABLE `' . _DB_PREFIX_ . 'mymod_order` ADD `shipping_method` VARCHAR(64) DEFAULT NULL');
    $db->execute('UPDATE `' . _DB_PREFIX_ . 'mymod_order` SET `shipping_method` = `carrier_name`');
    // Version 1.3.0: drop old column after code uses only the new one
    $db->execute('ALTER TABLE `' . _DB_PREFIX_ . 'mymod_order` DROP COLUMN `carrier_name`');
    ```

### Adding a table

4. Always use `CREATE TABLE IF NOT EXISTS` with `_DB_PREFIX_`, proper engine, charset, and indexes from day one:

    ```php
    $db->execute('CREATE TABLE IF NOT EXISTS `' . _DB_PREFIX_ . 'mymod_log` (
        `id_mymod_log` INT(11) UNSIGNED NOT NULL AUTO_INCREMENT,
        `id_order` INT(11) UNSIGNED NOT NULL,
        `action` VARCHAR(128) NOT NULL,
        `date_add` DATETIME NOT NULL,
        PRIMARY KEY (`id_mymod_log`),
        INDEX `idx_mymod_log_order` (`id_order`)
    ) ENGINE=' . _MYSQL_ENGINE_ . ' DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci');
    ```

### Removing a table

5. Only drop tables in `uninstall()` or in a MAJOR version upgrade. Back up first:

    ```php
    $db->execute('CREATE TABLE `' . _DB_PREFIX_ . 'mymod_log_backup_200` AS SELECT * FROM `' . _DB_PREFIX_ . 'mymod_log`');
    $db->execute('DROP TABLE IF EXISTS `' . _DB_PREFIX_ . 'mymod_log`');
    ```

### Adding an index

6. Guard with `SHOW INDEX` and use the naming convention `idx_<table>_<columns>`:

    ```php
    $indexes = $db->executeS('SHOW INDEX FROM `' . _DB_PREFIX_ . 'mymod_order` WHERE Key_name = "idx_mymod_order_status_date"');
    if (empty($indexes)) {
        $db->execute('CREATE INDEX `idx_mymod_order_status_date` ON `' . _DB_PREFIX_ . 'mymod_order` (`status`, `date_add`)');
    }
    ```

### Changing column type

7. Widening is safe (INT to BIGINT, VARCHAR(64) to VARCHAR(255)). Narrowing requires data validation:

    ```php
    // Safe widening
    $db->execute('ALTER TABLE `' . _DB_PREFIX_ . 'mymod_order` MODIFY `amount` BIGINT(20) NOT NULL DEFAULT 0');
    // Narrowing: validate first, then modify only if data fits
    $overflow = $db->getValue('SELECT COUNT(*) FROM `' . _DB_PREFIX_ . 'mymod_order` WHERE LENGTH(`code`) > 32');
    if ((int) $overflow === 0) {
        $db->execute('ALTER TABLE `' . _DB_PREFIX_ . 'mymod_order` MODIFY `code` VARCHAR(32) NOT NULL');
    }
    ```

### Multi-lang and multi-shop tables

8. For translatable columns, create a `_lang` sibling with `id_lang` in the composite PK. For shop-specific data, create a `_shop` sibling with `id_shop`. Mirror additions in both base and sibling tables:

    ```php
    // _lang table for translatable data
    $db->execute('CREATE TABLE IF NOT EXISTS `' . _DB_PREFIX_ . 'mymod_item_lang` (
        `id_mymod_item` INT(11) UNSIGNED NOT NULL,
        `id_lang` INT(11) UNSIGNED NOT NULL,
        `title` VARCHAR(255) NOT NULL,
        PRIMARY KEY (`id_mymod_item`, `id_lang`)
    ) ENGINE=' . _MYSQL_ENGINE_ . ' DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci');

    // _shop table for per-shop overrides
    $db->execute('CREATE TABLE IF NOT EXISTS `' . _DB_PREFIX_ . 'mymod_item_shop` (
        `id_mymod_item` INT(11) UNSIGNED NOT NULL,
        `id_shop` INT(11) UNSIGNED NOT NULL,
        `active` TINYINT(1) NOT NULL DEFAULT 1,
        PRIMARY KEY (`id_mymod_item`, `id_shop`)
    ) ENGINE=' . _MYSQL_ENGINE_ . ' DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci');
    ```

### Large table migrations (>100k rows)

9. Chunk `UPDATE` operations to avoid lock timeouts:

    ```php
    $batchSize = 5000;
    do {
        $affected = $db->execute(
            'UPDATE `' . _DB_PREFIX_ . 'mymod_order` SET `priority` = 0 WHERE `priority` IS NULL LIMIT ' . (int) $batchSize
        );
    } while ($affected > 0);
    ```

## Do

- Make every migration idempotent. The merchant may run the upgrade twice after a partial failure.
- Add indexes at table creation time. Use `_MYSQL_ENGINE_` and `_DB_PREFIX_` constants.
- Mirror every upgrade-script change in `install()` so fresh installs match upgraded installs.

## Don't

- Don't remove a column and stop reading it in the same version. Deprecate first, remove later.
- Don't run `ALTER TABLE` on core tables (`ps_product`, `ps_order`). Module migrations touch module-namespaced tables only.
- Don't use `ADD COLUMN IF NOT EXISTS` (MariaDB 10.0.2+ only). Use `SHOW COLUMNS` guard for compatibility.
- Don't modify large tables without chunking. A single `UPDATE` on 1M rows causes lock timeouts.

## Canonical examples

- [devdocs - Modules / Enabling auto-update](https://devdocs.prestashop-project.org/9/modules/creation/enabling-auto-update/) - upgrade script contract.
- [devdocs - Modules / Doctrine](https://devdocs.prestashop-project.org/9/modules/concepts/doctrine/) - Doctrine entity + SQL install parallel.
- [`PrestaShop/example-modules` - demodoctrine](https://github.com/PrestaShop/example-modules/tree/master/demodoctrine) - reference schema with `_lang` and `_shop` tables.

## Related skills

- `module-add-migration` - create a single upgrade script file.
- `module-database-rollback-strategy` - handle failures and reversibility.
- `module-upgrade-workflow` - the full release lifecycle for module upgrades.
- `module-add-doctrine-entity` - Doctrine entity that parallels the raw SQL schema.
