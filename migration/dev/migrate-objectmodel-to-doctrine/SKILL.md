---
name: migrate-objectmodel-to-doctrine
description: Replace an ObjectModel class (with public static $definition) with a Doctrine ORM entity, ServiceEntityRepository, and proper DI wiring. Use when a module defines persistence via ObjectModel and must migrate to Doctrine for PS 9 compatibility.
---

## Requirements

- Identify the legacy `ObjectModel` class and its `public static $definition = [...]` array.
- Map each field in `$definition['fields']` to a Doctrine column type.
- Confirm whether the entity has multilang fields (`'lang' => true` entries) requiring a `<Entity>Lang` companion.
- Confirm the existing table name (the Doctrine entity reuses it).

## Steps

1. **Identify the legacy ObjectModel:**

    ```php
    // LEGACY - to be replaced
    class MyModVoucher extends \ObjectModel
    {
        public $id_voucher;
        public $code;
        public $discount_percent;
        public $expires_at;

        public static $definition = [
            'table' => 'mymod_voucher',
            'primary' => 'id_voucher',
            'fields' => [
                'code' => ['type' => self::TYPE_STRING, 'size' => 32, 'required' => true],
                'discount_percent' => ['type' => self::TYPE_INT, 'required' => true],
                'expires_at' => ['type' => self::TYPE_DATE],
            ],
        ];
    }
    ```

2. **Create the Doctrine entity** under `src/Entity/<Name>.php` using PHP 8 attributes. Map each `$definition['fields']` entry:

    | ObjectModel TYPE | Doctrine type |
    |---|---|
    | `TYPE_STRING` | `string` (with `length:`) |
    | `TYPE_INT` | `integer` |
    | `TYPE_BOOL` | `boolean` |
    | `TYPE_FLOAT` | `decimal` (with `precision:`, `scale:`) |
    | `TYPE_DATE` | `datetime_immutable` |
    | `TYPE_HTML` / `TYPE_SQL` | `text` |

    ```php
    <?php
    namespace MyVendor\Mymodule\Entity;

    use Doctrine\ORM\Mapping as ORM;
    use MyVendor\Mymodule\Repository\VoucherRepository;

    #[ORM\Entity(repositoryClass: VoucherRepository::class)]
    #[ORM\Table(name: 'mymod_voucher')]
    class Voucher
    {
        #[ORM\Id]
        #[ORM\Column(name: 'id_voucher', type: 'integer')]
        #[ORM\GeneratedValue(strategy: 'AUTO')]
        private ?int $id = null;

        #[ORM\Column(name: 'code', type: 'string', length: 32)]
        private string $code;

        #[ORM\Column(name: 'discount_percent', type: 'integer')]
        private int $discountPercent;

        #[ORM\Column(name: 'expires_at', type: 'datetime_immutable')]
        private \DateTimeImmutable $expiresAt;

        // Constructor, getters, domain methods...
    }
    ```

3. **Create the repository** extending `ServiceEntityRepository`:

    ```php
    <?php
    namespace MyVendor\Mymodule\Repository;

    use Doctrine\Bundle\DoctrineBundle\Repository\ServiceEntityRepository;
    use Doctrine\Persistence\ManagerRegistry;
    use MyVendor\Mymodule\Entity\Voucher;

    final class VoucherRepository extends ServiceEntityRepository
    {
        public function __construct(ManagerRegistry $registry)
        {
            parent::__construct($registry, Voucher::class);
        }
    }
    ```

4. **Exclude entities from the service container** in `config/services.yml`:

    ```yaml
    MyVendor\Mymodule\:
      resource: '../src/*'
      exclude: '../src/{Entity}'
    ```

5. **Handle multilang fields.** If the legacy `$definition` has `'lang' => true` entries, create a `<Entity>Lang` companion entity with composite key `(id_<entity>, id_lang)` and a `@ORM\OneToMany` on the parent. See `module-add-doctrine-entity` for the full pattern.

6. **Remove the ObjectModel class.** Update all call sites to use the repository or CQRS handlers (see `migrate-objectmodel-to-cqrs`). Replace:
   - `new MyModVoucher($id)` with `$repository->find($id)`
   - `$obj->add()` / `$obj->update()` with `$em->persist($entity); $em->flush();`
   - `$obj->delete()` with `$em->remove($entity); $em->flush();`

7. **Keep the existing SQL install script.** The table schema is already in production; do not change column names. The Doctrine entity maps to the same table transparently.

## Do

- Reuse the existing table name and column names so no migration SQL is needed.
- Use PHP 8 ORM attributes (not XML or YAML mapping).
- Use `ServiceEntityRepository` so the repository is autowirable by FQCN.
- PrestaShop auto-discovers entities in `src/Entity/` of installed modules.

## Don't

- Don't run `bin/console doctrine:schema:update --force` against a live PrestaShop database.
- Don't rename columns to camelCase in the DB; use the `name:` attribute argument to map snake_case columns to camelCase properties.
- Don't extend `ObjectModel` and `ORM\Entity` simultaneously; pick one persistence strategy.
- Don't use `Db::getInstance()` for tables managed by Doctrine.

## Canonical examples

- [devdocs - Doctrine](https://devdocs.prestashop-project.org/9/modules/concepts/doctrine/)
- [devdocs - Migration guide](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/)

## Related skills

- `module-add-doctrine-entity` - full recipe for creating a Doctrine entity from scratch.
- `migrate-objectmodel-to-cqrs` - replacing ObjectModel CRUD calls with CQRS commands.
