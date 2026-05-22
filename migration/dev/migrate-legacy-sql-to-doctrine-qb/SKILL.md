---
name: migrate-legacy-sql-to-doctrine-qb
description: Replace Db::getInstance()->executeS/execute/getValue raw SQL calls with Doctrine DBAL Connection QueryBuilder or ORM QueryBuilder with parameterised queries. Use when a module runs raw SQL through the legacy database wrapper and needs injection-safe, testable queries.
---

## Requirements

- Module name and vendor namespace.
- List of raw SQL call sites (`Db::getInstance()->executeS()`, `->execute()`, `->getValue()`).
- Whether the queries are read-only (select) or mutating (insert/update/delete).
- Target table names and their Doctrine entity mappings (if entities exist).

## Steps

1. Grep for all legacy DB calls:

    ```bash
    grep -rn 'Db::getInstance()' modules/mymodule/
    grep -rn '\$this->db->' modules/mymodule/
    ```

2. For each read query, inject `Doctrine\DBAL\Connection` and use `createQueryBuilder()`:

    ```php
    // Before
    $sql = 'SELECT * FROM ' . _DB_PREFIX_ . 'mymod_item WHERE active = 1';
    $rows = Db::getInstance()->executeS($sql);

    // After
    $qb = $this->connection->createQueryBuilder()
        ->select('*')
        ->from($this->dbPrefix . 'mymod_item')
        ->where('active = :active')
        ->setParameter('active', 1, \PDO::PARAM_INT);
    $rows = $qb->executeQuery()->fetchAllAssociative();
    ```

3. For single-value queries (`getValue`), use `fetchOne()`:

    ```php
    // Before
    $count = (int) Db::getInstance()->getValue(
        'SELECT COUNT(*) FROM ' . _DB_PREFIX_ . 'mymod_item'
    );

    // After
    $count = (int) $this->connection->createQueryBuilder()
        ->select('COUNT(*)')
        ->from($this->dbPrefix . 'mymod_item')
        ->executeQuery()
        ->fetchOne();
    ```

4. For mutating queries (INSERT/UPDATE/DELETE), use the Connection directly or the QueryBuilder:

    ```php
    // Before
    Db::getInstance()->execute(
        "UPDATE " . _DB_PREFIX_ . "mymod_item SET active = 0 WHERE id_item = " . (int) $id
    );

    // After
    $this->connection->createQueryBuilder()
        ->update($this->dbPrefix . 'mymod_item')
        ->set('active', ':active')
        ->where('id_item = :id')
        ->setParameter('active', 0, \PDO::PARAM_INT)
        ->setParameter('id', $id, \PDO::PARAM_INT)
        ->executeStatement();
    ```

5. Inject the Connection and dbPrefix via DI. Use the `prestashop.core.admin.connection` service or autowire `Doctrine\DBAL\Connection`:

    ```yaml
    services:
      MyVendor\Mymodule\Repository\ItemRepository:
        arguments:
          $connection: '@doctrine.dbal.default_connection'
          $dbPrefix: '%database_prefix%'
    ```

6. If the module already has Doctrine ORM entities, prefer the EntityManager / Repository for single-entity CRUD and reserve DBAL QueryBuilder for complex joins, aggregates, or Grid query builders.

7. Remove `use Db;` and all `Db::getInstance()` calls once migrated. Confirm no regressions by running the module's test suite and exercising each migrated query path.

## Do

- Always use `setParameter()` with typed bindings (`PARAM_INT`, `PARAM_STR`, `PARAM_BOOL`).
- Use `$this->dbPrefix` (injected) instead of `_DB_PREFIX_` constant for testability.
- Use `fetchAllAssociative()` for result sets and `fetchOne()` for scalars.

## Don't

- Don't concatenate variables into SQL strings, even with `(int)` cast; use parameters.
- Don't mix `Db::getInstance()` and Doctrine in the same class; migrate fully.
- Don't use `->execute()` (deprecated in DBAL 3+); use `->executeQuery()` for SELECT and `->executeStatement()` for DML.

## Related skills

- `migrate-objectmodel-to-doctrine` - full ORM entity migration.
- `module-add-doctrine-entity` - creating entities that these queries may target.
- `migrate-helperlist-to-grid` - Grid query builders use the same DBAL pattern.
