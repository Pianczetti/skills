---
name: module-add-doctrine-entity
description: Add a Doctrine ORM entity (with optional translatable fields) and a service repository to a PrestaShop 9 module, plus a SQL install script. Use when the user wants persistent storage for the module without going through legacy ObjectModel.
---

## Requirements

Ask the user:
* Entity name in PascalCase singular (e.g. `Voucher`, `Subscription`). The class lives in `src/Entity/<Entity>.php`.
* Table name in `snake_case`, prefixed with the module short code (e.g. `mymod_voucher`). PrestaShop automatically prepends the configured DB prefix (default `ps_`), so the actual table becomes `ps_mymod_voucher`.
* Field list with Doctrine types (`integer`, `string` with length, `text`, `boolean`, `datetime_immutable`, `decimal` with precision/scale).
* Whether some fields vary per language. If yes, the module also needs a `<Entity>Lang` companion entity joined on `id_<entity>` + `id_lang`.
* Indexes and unique constraints (e.g. unique on `code`, index on `id_shop`).

## Steps

1. Create the main entity under `src/Entity/<Entity>.php` using PHP 8 attributes. PrestaShop scans `src/Entity/` of every installed module and auto-registers mappings; you do NOT need a custom `prestashop.entity_manager.<modulename>` setup, the standard `doctrine.orm.entity_manager` handles it:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Entity;

    use Doctrine\Common\Collections\ArrayCollection;
    use Doctrine\Common\Collections\Collection;
    use Doctrine\ORM\Mapping as ORM;
    use MyVendor\Mymodule\Repository\VoucherRepository;

    #[ORM\Entity(repositoryClass: VoucherRepository::class)]
    #[ORM\Table(name: 'mymod_voucher')]
    #[ORM\UniqueConstraint(name: 'uniq_mymod_voucher_code', columns: ['code'])]
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

        /** @var Collection<int, VoucherLang> */
        #[ORM\OneToMany(mappedBy: 'voucher', targetEntity: VoucherLang::class, cascade: ['persist', 'remove'], orphanRemoval: true)]
        private Collection $voucherLangs;

        public function __construct(string $code, int $discountPercent, \DateTimeImmutable $expiresAt)
        {
            $this->code = $code;
            $this->discountPercent = $discountPercent;
            $this->expiresAt = $expiresAt;
            $this->voucherLangs = new ArrayCollection();
        }

        public function getId(): ?int { return $this->id; }
        public function getCode(): string { return $this->code; }
        public function getDiscountPercent(): int { return $this->discountPercent; }
        public function getExpiresAt(): \DateTimeImmutable { return $this->expiresAt; }
        public function getVoucherLangs(): Collection { return $this->voucherLangs; }

        public function addVoucherLang(VoucherLang $lang): void
        {
            if (!$this->voucherLangs->contains($lang)) {
                $this->voucherLangs->add($lang);
                $lang->setVoucher($this);
            }
        }
    }
    ```

2. If the entity has translatable fields, create the `<Entity>Lang` companion under `src/Entity/<Entity>Lang.php`. The composite key is `id_<entity>` + `id_lang` and the parent owns the relation:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Entity;

    use Doctrine\ORM\Mapping as ORM;

    #[ORM\Entity]
    #[ORM\Table(name: 'mymod_voucher_lang')]
    class VoucherLang
    {
        #[ORM\Id]
        #[ORM\ManyToOne(targetEntity: Voucher::class, inversedBy: 'voucherLangs')]
        #[ORM\JoinColumn(name: 'id_voucher', referencedColumnName: 'id_voucher', nullable: false, onDelete: 'CASCADE')]
        private Voucher $voucher;

        #[ORM\Id]
        #[ORM\Column(name: 'id_lang', type: 'integer')]
        private int $langId;

        #[ORM\Column(name: 'description', type: 'text')]
        private string $description;

        public function __construct(Voucher $voucher, int $langId, string $description)
        {
            $this->voucher = $voucher;
            $this->langId = $langId;
            $this->description = $description;
        }

        public function getVoucher(): Voucher { return $this->voucher; }
        public function setVoucher(Voucher $voucher): void { $this->voucher = $voucher; }
        public function getLangId(): int { return $this->langId; }
        public function getDescription(): string { return $this->description; }
    }
    ```

3. Create the repository under `src/Repository/<Entity>Repository.php` extending `ServiceEntityRepository` (auto-registered as a service via the `resource:` glob in `services.yml`):

    ```php
    <?php
    declare(strict_types=1);

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

        public function findActiveByCode(string $code, \DateTimeImmutable $now): ?Voucher
        {
            return $this->createQueryBuilder('v')
                ->andWhere('v.code = :code')
                ->andWhere('v.expiresAt > :now')
                ->setParameter('code', $code)
                ->setParameter('now', $now)
                ->getQuery()
                ->getOneOrNullResult();
        }
    }
    ```

4. Exclude `Entity/` from the auto-registered services (entities are not services). The default `services.yml` produced by `module-create` already does this:

    ```yaml
    services:
      _defaults:
        autowire: true
        autoconfigure: true
        public: false

      MyVendor\Mymodule\:
        resource: '../src/*'
        exclude: '../src/{Entity,DTO}'
    ```

5. Create the schema with a SQL install script. The official Doctrine page warns explicitly NOT to run `doctrine:schema:update --force` against a PrestaShop install (it adds foreign keys that break core). Instead, write `sql/install.sql` and `sql/uninstall.sql` and execute them from the module:

    ```sql
    -- modules/mymodule/sql/install.sql
    CREATE TABLE IF NOT EXISTS `PREFIX_mymod_voucher` (
        `id_voucher`        INT(11) UNSIGNED NOT NULL AUTO_INCREMENT,
        `code`              VARCHAR(32) NOT NULL,
        `discount_percent`  INT(11) NOT NULL,
        `expires_at`        DATETIME NOT NULL,
        PRIMARY KEY (`id_voucher`),
        UNIQUE KEY `uniq_mymod_voucher_code` (`code`)
    ) ENGINE=ENGINE_TYPE DEFAULT CHARSET=utf8mb4;

    CREATE TABLE IF NOT EXISTS `PREFIX_mymod_voucher_lang` (
        `id_voucher` INT(11) UNSIGNED NOT NULL,
        `id_lang`    INT(11) UNSIGNED NOT NULL,
        `description` TEXT NOT NULL,
        PRIMARY KEY (`id_voucher`, `id_lang`)
    ) ENGINE=ENGINE_TYPE DEFAULT CHARSET=utf8mb4;
    ```

    ```php
    // mymodule.php
    public function install(): bool
    {
        if (!parent::install()) {
            return false;
        }

        $sql = (string) file_get_contents(__DIR__ . '/sql/install.sql');
        $sql = str_replace(['PREFIX_', 'ENGINE_TYPE'], [_DB_PREFIX_, _MYSQL_ENGINE_], $sql);

        foreach (array_filter(array_map('trim', explode(';', $sql))) as $statement) {
            if (!\Db::getInstance()->execute($statement)) {
                return false;
            }
        }

        return true;
    }

    public function uninstall(): bool
    {
        $sql = (string) file_get_contents(__DIR__ . '/sql/uninstall.sql');
        $sql = str_replace('PREFIX_', _DB_PREFIX_, $sql);
        foreach (array_filter(array_map('trim', explode(';', $sql))) as $statement) {
            \Db::getInstance()->execute($statement);
        }

        return parent::uninstall();
    }
    ```

   To bootstrap the SQL, generate it once with `bin/console doctrine:schema:update --dump-sql` (DUMP only, never `--force`) against a sandbox install, then strip foreign-key statements and commit the cleaned-up file.

6. For schema evolution, ship an `upgrade/upgrade-<version>.php` per release. PrestaShop auto-runs every file whose version is greater than the installed one:

    ```php
    <?php
    // modules/mymodule/upgrade/upgrade-1.1.0.php
    declare(strict_types=1);

    function upgrade_module_1_1_0($module): bool
    {
        return (bool) \Db::getInstance()->execute(
            'ALTER TABLE `' . _DB_PREFIX_ . 'mymod_voucher` ADD `usage_count` INT(11) NOT NULL DEFAULT 0'
        );
    }
    ```

7. Inject the repository into a Command/Query handler or a service. The repository is autowired by FQCN:

    ```php
    public function __construct(private readonly VoucherRepository $vouchers) {}
    ```

## Do

- Use the `<Entity>Lang` pattern for any field that varies by language. Join on `id_<entity>` + `id_lang` and expose the lang collection from the parent.
- Use repositories that extend `ServiceEntityRepository` so they are auto-registered as services and inject as `MyVendor\...\Repository\VoucherRepository`.
- Generate the install SQL once with `doctrine:schema:update --dump-sql`, then commit the cleaned-up file under `sql/install.sql`.
- Use the existing `doctrine.orm.entity_manager` service. PrestaShop scans `src/Entity/` of every installed module automatically.

## Don't

- Don't extend `ObjectModel` for new persistence. Use Doctrine entities. `ObjectModel` is the legacy code path.
- Don't run `bin/console doctrine:schema:update --force` against a PrestaShop database. The Doctrine doc warns explicitly against it - it generates foreign keys incompatible with core.
- Don't use raw `Db::getInstance()->insert()` / `->update()` for entities that are managed by Doctrine. Use the repository and `EntityManager::persist()` + `flush()`.
- Don't ship Doctrine migrations that drop or alter core tables (`ps_product`, `ps_customer`, ...). Migrations belong inside your module's namespace (`mymod_*`) only.

## PS9 Module Compatibility Notes

### Annotations vs PHP 8 Attributes

PrestaShop 9's `ModulesDoctrineCompilerPass` registers module entities using **AnnotationDriver**, NOT the PHP 8 Attribute driver. While the core codebase may use attributes, modules discovered via the compiler pass must use **docblock annotations**:

```php
// WRONG - causes MappingException in PS9 module context
#[ORM\Entity(repositoryClass: VoucherRepository::class)]
#[ORM\Table(name: 'mymod_voucher')]
class Voucher { }

// CORRECT - PS9 module entity mapping uses annotations
/**
 * @ORM\Entity(repositoryClass="MyVendor\Mymodule\Repository\VoucherRepository")
 * @ORM\Table(name="mymod_voucher")
 */
class Voucher { }
```

This applies to ALL ORM mapping: `@ORM\Column`, `@ORM\Id`, `@ORM\GeneratedValue`, `@ORM\OneToMany`, `@ORM\ManyToOne`, `@ORM\JoinColumn`.

### Table Prefix

PrestaShop uses a configurable DB prefix (e.g. `ps_`, `ps0g_`). The raw table name in `@ORM\Table(name="mymod_voucher")` will NOT be auto-prefixed by Doctrine.

**Solution:** Create a Doctrine event subscriber that prepends `%database_prefix%` to table names for your module's entity namespace:

```php
namespace MyVendor\Mymodule\Doctrine;

use Doctrine\ORM\Event\LoadClassMetadataEventArgs;

class TablePrefixSubscriber implements \Doctrine\Common\EventSubscriber
{
    public function __construct(private readonly string $prefix) {}

    public function getSubscribedEvents(): array { return ['loadClassMetadata']; }

    public function loadClassMetadata(LoadClassMetadataEventArgs $args): void
    {
        $metadata = $args->getClassMetadata();
        if (!str_starts_with($metadata->getName(), 'MyVendor\\Mymodule\\Entity\\')) { return; }
        $table = $metadata->getTableName();
        if (!str_starts_with($table, $this->prefix)) {
            $metadata->setPrimaryTable(['name' => $this->prefix . $table]);
        }
    }
}
```

Register in `config/services.yml`:
```yaml
MyVendor\Mymodule\Doctrine\TablePrefixSubscriber:
  arguments:
    $prefix: '%database_prefix%'
  tags:
    - { name: doctrine.event_subscriber }
```

## Canonical examples

- [devdocs - Doctrine](https://devdocs.prestashop-project.org/9/modules/concepts/doctrine/) (covers entity discovery, the `--dump-sql` workflow, and the foreign-key warning).
- [devdocs - Modules - Composer](https://devdocs.prestashop-project.org/9/modules/concepts/composer/) for the namespace setup that lets PrestaShop find your `src/Entity/` classes.
- Working sample with entity + repository + SQL install: [`example-modules/demodoctrine`](https://github.com/PrestaShop/example-modules/tree/master/demodoctrine).
- Multi-lang reference (core): [`Doctrine/Entity/Lang`-style classes](https://github.com/PrestaShop/PrestaShop/tree/develop/src/Core/Domain/Product) under `src/PrestaShopBundle/Entity/` and `src/Core/Domain/`.
