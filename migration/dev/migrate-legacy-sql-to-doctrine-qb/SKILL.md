---
name: migrate-legacy-sql-to-doctrine-qb
description: Replace raw `Db::getInstance()->executeS()` / `->execute()` SQL with Doctrine DBAL Connection or ORM QueryBuilder. Use when a module scatters raw SQL throughout controllers and helpers and must adopt the PS 9 persistence layer.
---

## Requirements

- Inventory all `Db::getInstance()` calls (`executeS`, `execute`, `insert`, `update`, `delete`, `getValue`).
- Confirm whether the tables have a Doctrine entity (use QueryBuilder ORM) or remain legacy (use DBAL Connection).
- Identify the table prefix usage (`_DB_PREFIX_`).

## Steps

1. **Identify the legacy pattern:**

    ```php
    // SELECT
    $sql = 'SELECT * FROM `' . _DB_PREFIX_ . 'mymod_voucher` WHERE active = 1';
    $rows = Db::getInstance()->executeS($sql);

    // INSERT
    Db::getInstance()->insert('mymod_voucher', ['code' => pSQL($code), 'active' => 1]);

    // UPDATE
    Db::getInstance()->update('mymod_voucher', ['active' => 0], 'id_voucher = ' . (int) $id);

    // DELETE
    Db::getInstance()->execute('DELETE FROM `' . _DB_PREFIX_ . 'mymod_voucher` WHERE id_voucher = ' . (int) $id);
    ```

2. **Inject Doctrine DBAL Connection** in your service (for tables without an ORM entity):

    ```yaml
    # config/services.yml
    services:
      MyVendor\Mymodule\Repository\VoucherRepository:
        arguments:
          $connection: '@doctrine.dbal.default_connection'
          $dbPrefix: '%database_prefix%'
    ```

3. **Replace SELECT with DBAL QueryBuilder:**

    ```php
    use Doctrine\DBAL\Connection;

    final class VoucherRepository
    {
        public function __construct(
            private readonly Connection $connection,
            private readonly string $dbPrefix,
        ) {}

        public function findActive(): array
        {
            $qb = $this->connection->createQueryBuilder();
            $qb->select('v.*')
               ->from($this->dbPrefix . 'mymod_voucher', 'v')
               ->where('v.active = :active')
               ->setParameter('active', 1);

            return $qb->executeQuery()->fetchAllAssociative();
        }
    }
    ```

4. **Replace INSERT:**

    ```php
    public function create(string $code): void
    {
        $this->connection->insert($this->dbPrefix . 'mymod_voucher', [
            'code' => $code,
            'active' => 1,
        ]);
    }
    ```

5. **Replace UPDATE:**

    ```php
    public function deactivate(int $id): void
    {
        $this->connection->update(
            $this->dbPrefix . 'mymod_voucher',
            ['active' => 0],
            ['id_voucher' => $id]
        );
    }
    ```

6. **Replace DELETE:**

    ```php
    public function remove(int $id): void
    {
        $this->connection->delete(
            $this->dbPrefix . 'mymod_voucher',
            ['id_voucher' => $id]
        );
    }
    ```

7. **If a Doctrine ORM entity exists**, use the EntityManager QueryBuilder instead:

    ```php
    use Doctrine\ORM\EntityManagerInterface;
    use MyVendor\Mymodule\Entity\Voucher;

    final class VoucherRepository
    {
        public function __construct(private readonly EntityManagerInterface $em) {}

        public function findActive(): array
        {
            return $this->em->createQueryBuilder()
                ->select('v')
                ->from(Voucher::class, 'v')
                ->where('v.active = :active')
                ->setParameter('active', true)
                ->getQuery()
                ->getResult();
        }
    }
    ```

8. **Use `%database_prefix%`** instead of `_DB_PREFIX_` constant. The DI parameter is available in service definitions and avoids coupling to the global constant.

## Do

- Use parameterized queries (`setParameter`) for all user-supplied values; never concatenate.
- Inject `%database_prefix%` from the container; never use `_DB_PREFIX_` in a Symfony service.
- Centralize queries in a repository service; keep controllers free of SQL.

## Don't

- Don't mix `Db::getInstance()` and Doctrine in the same service; pick one persistence layer per repository.
- Don't use `pSQL()` with Doctrine; parameter binding handles escaping.
- Don't forget to register the repository in `config/services.yml` with autowire.

## Related skills

- `migrate-objectmodel-to-doctrine` - migrating full ObjectModel entities to Doctrine ORM.
- `module-add-doctrine-entity` - creating a new Doctrine entity from scratch.
