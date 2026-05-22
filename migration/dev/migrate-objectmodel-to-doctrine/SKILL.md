---
name: migrate-objectmodel-to-doctrine
description: Replace an ObjectModel class (its $definition array and public static properties) with a Doctrine ORM entity, a ServiceEntityRepository, and DI wiring. Use when a module persists data through ObjectModel and needs to adopt the PS 9 Doctrine-first persistence layer.
---

## Requirements

- Module name and vendor namespace.
- ObjectModel class name and its `$definition` array (table name, primary key, fields with types and validation).
- Whether translatable fields exist (`MultilangType` in the definition).
- Whether other module code still references the ObjectModel (handlers, services, hooks).

## Steps

1. Create the Doctrine entity under `src/Entity/<Name>.php` with `#[ORM\Entity]` and `#[ORM\Table]`. Map each field from `$definition['fields']` to a Doctrine column:

    ```php
    namespace MyVendor\Mymodule\Entity;

    use Doctrine\ORM\Mapping as ORM;

    #[ORM\Entity(repositoryClass: VoucherRepository::class)]
    #[ORM\Table(name: 'mymod_voucher')]
    class Voucher
    {
        #[ORM\Id]
        #[ORM\GeneratedValue]
        #[ORM\Column(type: 'integer')]
        private int $id;

        #[ORM\Column(type: 'string', length: 64)]
        private string $code;

        #[ORM\Column(type: 'integer')]
        private int $discountPercent;

        #[ORM\Column(type: 'boolean')]
        private bool $active = true;
    }
    ```

2. If translatable fields exist, create a separate entity `<Name>Lang` with a composite primary key (`id` + `id_lang`) and a `ManyToOne` relation back to the parent.

3. Create the repository under `src/Repository/<Name>Repository.php` extending `Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository`:

    ```php
    namespace MyVendor\Mymodule\Repository;

    use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
    use Doctrine\Persistence\ManagerRegistry;
    use MyVendor\Mymodule\Entity\Voucher;

    class VoucherRepository extends ServiceEntityRepository
    {
        public function __construct(ManagerRegistry $registry)
        {
            parent::__construct($registry, Voucher::class);
        }
    }
    ```

4. Register the Doctrine mapping in `config/services.yml` (or a dedicated `config/doctrine.yml` imported by the module):

    ```yaml
    services:
      _defaults:
        autowire: true
        autoconfigure: true

      MyVendor\Mymodule\Repository\VoucherRepository: ~
    ```

   Ensure the module's `composer.json` autoload maps `src/` and the entity directory is picked up by `DoctrineExtension` via the bundle config or the module's `getEntityMappingDir()` method.

5. Keep the install SQL in `sql/install.sql` for the initial schema creation (PrestaShop modules do not run Doctrine migrations on install). The Doctrine mapping must match this schema exactly.

6. Replace all `new MymoduleVoucher()` / `ObjectModel::` calls with repository or EntityManager calls in CQRS handlers:

    ```php
    // Before
    $v = new MymoduleVoucher($id);
    $v->code = 'NEW';
    $v->update();

    // After (in handler)
    $voucher = $this->repository->find($id);
    $voucher->setCode('NEW');
    $this->em->flush();
    ```

7. Once all consumers are migrated, mark the ObjectModel file for deletion. If third-party code may still reference it, deprecate with a `@deprecated` docblock and a trigger_error notice.

8. Validate with `bin/console doctrine:mapping:info` to confirm the entity is registered, and run the module's install/uninstall round-trip.

## Do

- Use `getEntityMappingDir()` in the module class to point Doctrine at `src/Entity/`.
- Match the `#[ORM\Table(name:)]` to the real table name without the `_DB_PREFIX_`; PrestaShop's Doctrine config prepends the prefix automatically via `TablePrefix` subscriber.
- Keep install SQL as the source of truth for schema creation; Doctrine mapping is for runtime only.

## Don't

- Don't run `doctrine:schema:update --force` on a PrestaShop install; ship schema changes via `upgrade/upgrade-<version>.php`.
- Don't expose Doctrine entities outside the domain layer; return DTOs or Value Objects from handlers.
- Don't delete the ObjectModel file until every consumer is confirmed migrated.

## Related skills

- `module-add-doctrine-entity` - greenfield Doctrine entity creation.
- `migrate-objectmodel-to-cqrs` - the CQRS layer that calls this persistence.
- `migrate-legacy-sql-to-doctrine-qb` - replacing raw SQL with Doctrine QueryBuilder.
