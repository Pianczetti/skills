---
name: module-make-multistore-aware
description: Make a PrestaShop 9 module behave correctly across multiple shops and shop groups. Use this checklist whenever the module reads or writes Configuration, queries the database, exposes a BO admin page, or ships CQRS commands and queries.
---

## Requirements

Ask the user:
* Which `Configuration` keys must hold per-shop values vs. per-group vs. global. List every key.
* Which queries / repository methods must be filtered by `id_shop`. Inventory the joins they need (`product_shop`, `category_shop`, `<entity>_shop`).
* Whether the module ships any BO admin page that must respect the multistore selector at the top of the BO (the "All Shops / Group / Shop X" picker).
* Whether the module exposes CQRS commands or queries that mutate per-shop state. Each one must accept a `ShopConstraint` argument.

## Steps

1. Read `.ai/MULTISTORE.md` from the core repo before writing code. It is the authoritative cross-cutting guide and lists every gotcha (per-shop overrides, default-shop reads, customer sharing, grid filters): [`PrestaShop/PrestaShop/.ai/MULTISTORE.md`](https://github.com/PrestaShop/PrestaShop/blob/develop/.ai/MULTISTORE.md). Also read the devdocs page for the module-author angle: [Multistore for modules](https://devdocs.prestashop-project.org/9/modules/concepts/multistore/).

2. Configuration reads and writes must always pass shop scope. Inside a module, the explicit-args form of the legacy API is acceptable; in modern services prefer the `Configuration` adapter:

    ```php
    // OK inside a module class - explicit args, multistore-aware
    Configuration::updateValue('MYMOD_API_KEY', $value, false, $idShopGroup ?? null, $idShop ?? null);
    $value = Configuration::get('MYMOD_API_KEY', null, $idShopGroup ?? null, $idShop ?? null);

    // Better in modern services - inject the adapter and pass a ShopConstraint
    use PrestaShop\PrestaShop\Adapter\Configuration;
    use PrestaShop\PrestaShop\Core\Domain\Shop\ValueObject\ShopConstraint;

    public function __construct(private readonly Configuration $configuration) {}

    public function read(int $idShop): ?string
    {
        return $this->configuration->get('MYMOD_API_KEY', null, ShopConstraint::shop($idShop));
    }
    ```

   Per the core multistore guide: a `ShopConstraint::allShops()` write does NOT clobber existing per-shop values - it sets the all-shops default and shop overrides keep precedence at read time.

3. In repositories and Doctrine queries, join on the `*_shop` link table and filter by `Shop::getContextListShopID()` (or the shop ids carried by a `ShopConstraint`):

    ```php
    public function findByCart(int $cartId): array
    {
        $qb = $this->createQueryBuilder('v')
            ->innerJoin('mymod_voucher_shop', 'vs', 'WITH', 'vs.id_voucher = v.id')
            ->where('vs.id_shop IN (:shopIds)')
            ->setParameter('shopIds', Shop::getContextListShopID());

        return $qb->getQuery()->getResult();
    }
    ```

   Add a `<entity>_shop` link table when an entity can have per-shop values; otherwise a single shared row is fine but the query must still filter by the configured shops.

4. In modern admin controllers, inject the shop context adapter and read the current scope from it. Never call `Shop::getContext()` from a service:

    ```php
    use PrestaShop\PrestaShop\Adapter\Shop\Context as ShopContextAdapter;
    use Symfony\Component\DependencyInjection\Attribute\Autowire;

    public function indexAction(
        Request $request,
        #[Autowire(service: 'prestashop.adapter.shop.context')]
        ShopContextAdapter $shopContext,
    ): Response {
        $shopIds = $shopContext->getContextListShopID();      // int[]
        $constraint = $shopContext->getShopConstraint();       // ShopConstraint
        // ...
    }
    ```

   The service id is `prestashop.adapter.shop.context` and the FQCN is `PrestaShop\PrestaShop\Adapter\Shop\Context`. It implements `MultistoreContextCheckerInterface`, `ShopContextInterface` and `ShopConstraintContextInterface`, which is what most modern code typehints.

5. For CQRS commands and queries, accept a `ShopConstraint` (or `ShopId` / `ShopGroupId`) value object on the command itself. Apply the constraint inside the handler:

    ```php
    use PrestaShop\PrestaShop\Core\Domain\Shop\ValueObject\ShopConstraint;

    final class CreateVoucherCommand
    {
        public function __construct(
            public readonly string $code,
            public readonly int $discountPercent,
            public readonly ShopConstraint $shopConstraint,
        ) {}
    }
    ```

    ```php
    public function handle(CreateVoucherCommand $command): VoucherId
    {
        $voucher = new Voucher(...);
        $this->em->persist($voucher);

        foreach ($this->resolveShopIds($command->shopConstraint) as $shopId) {
            $this->em->persist(new VoucherShop($voucher, $shopId));
        }
        $this->em->flush();
        return new VoucherId($voucher->getId());
    }
    ```

   `ShopConstraint::shop($id)`, `ShopConstraint::shopGroup($id)` and `ShopConstraint::allShops()` are the three factory methods you will use.

6. For settings forms that need a "use shop default" checkbox per field, extend `PrestaShop\PrestaShop\Core\Configuration\AbstractMultistoreConfiguration`. It reads/writes through `ShopConstraint` derived from the admin's current selector and renders the checkbox automatically.

7. Add a Behat (or unit) scenario per command that exercises the three contexts: `allShops`, `shopGroup($id)`, `shop($id)`. The simplest possible regression check is that a write to one shop never changes another shop's value.

## Do

- Always pass shop scope explicitly. The default args of `Configuration::get()` / `updateValue()` use the current static context, which is invisible coupling.
- Add a `<entity>_shop` link table for any entity that can be configured per shop; query it on every read.
- Inject `prestashop.adapter.shop.context` into modern services and controllers; let the adapter answer `getContextShopID()`, `getContextListShopID()`, `getShopConstraint()`.
- Make commands and queries accept `ShopConstraint` (or `ShopId|ShopGroupId`) and apply it inside the handler.
- Write a multistore Behat scenario per command. The `.ai/MULTISTORE.md` guide insists on this.

## Don't

- Don't assume `Shop::CONTEXT_SHOP`. The merchant can be browsing the BO in `CONTEXT_GROUP` or `CONTEXT_ALL`; the module must work in all three.
- Don't read shop context from a static helper inside a CQRS handler. Pass the constraint in via the command.
- Don't call `Configuration::updateValue('KEY', $value)` with default args from a multishop-aware module - it ignores per-shop scope.
- Don't filter Grid Doctrine queries on `id_shop = ...`; use `IN (:shopIds)` populated from `Shop::getContextListShopID()` so group context still works.

## Canonical examples

- [`PrestaShop/PrestaShop` - .ai/MULTISTORE.md](https://github.com/PrestaShop/PrestaShop/blob/develop/.ai/MULTISTORE.md) - the cross-cutting cheat sheet (read this first).
- [devdocs - Modules / multistore](https://devdocs.prestashop-project.org/9/modules/concepts/multistore/) and [devdocs - Multistore (development)](https://devdocs.prestashop-project.org/9/development/multistore/).
- [`PrestaShop/PrestaShop` - ShopConstraint](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Domain/Shop/ValueObject/ShopConstraint.php), [`ShopId`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Domain/Shop/ValueObject/ShopId.php), [`ShopGroupId`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Domain/Shop/ValueObject/ShopGroupId.php).
- [`PrestaShop/PrestaShop` - Adapter\Shop\Context](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Adapter/Shop/Context.php) - the `prestashop.adapter.shop.context` service.
- [`PrestaShop/example-modules` - demomultistoreform](https://github.com/PrestaShop/example-modules/tree/master/demomultistoreform) - working FormDataProvider that branches on the multistore context.
