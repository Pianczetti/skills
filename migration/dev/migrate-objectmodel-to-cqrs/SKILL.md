---
name: migrate-objectmodel-to-cqrs
description: Replace direct ObjectModel usage (new MyObject / add / update / delete) with CQRS Commands and Queries dispatched on the Symfony Messenger bus. Use when a module mutates or reads aggregates through ObjectModel and needs to move to the PS 9 domain layer.
---

## Requirements

- Module name and vendor namespace.
- ObjectModel class being replaced (e.g. `MymoduleVoucher extends ObjectModel`).
- Mutation operations to migrate (create, update, delete, toggle, bulk).
- Read operations to migrate (get by id, list, search).
- Domain rules currently embedded in the ObjectModel or controller.

## Steps

1. Identify every call site: grep the module for `new MymoduleVoucher(`, `->add()`, `->update()`, `->delete()`, `->save()`, `ObjectModel::` static helpers. Each one becomes either a Command dispatch or a Query dispatch.

2. Create the Command for each mutation under `src/Domain/<Aggregate>/Command/`:

    ```php
    namespace MyVendor\Mymodule\Domain\Voucher\Command;

    final class CreateVoucherCommand
    {
        public function __construct(
            public readonly string $code,
            public readonly int $discountPercent,
        ) {}
    }
    ```

3. Create the CommandHandler under `src/Domain/<Aggregate>/CommandHandler/` tagged with `#[AsCommandHandler]`:

    ```php
    namespace MyVendor\Mymodule\Domain\Voucher\CommandHandler;

    use PrestaShop\PrestaShop\Core\CommandBus\Attributes\AsCommandHandler;

    #[AsCommandHandler]
    final class CreateVoucherHandler implements CreateVoucherCommandHandlerInterface
    {
        public function __construct(private readonly EntityManagerInterface $em) {}

        public function handle(CreateVoucherCommand $command): VoucherId
        {
            $voucher = new Voucher($command->code, $command->discountPercent);
            $this->em->persist($voucher);
            $this->em->flush();
            return new VoucherId($voucher->getId());
        }
    }
    ```

4. Create Queries and QueryHandlers for read operations under `src/Domain/<Aggregate>/Query/` and `src/Domain/<Aggregate>/QueryHandler/`, tagged with `#[AsQueryHandler]`:

    ```php
    namespace MyVendor\Mymodule\Domain\Voucher\QueryHandler;

    use PrestaShop\PrestaShop\Core\CommandBus\Attributes\AsQueryHandler;

    #[AsQueryHandler]
    final class GetVoucherHandler implements GetVoucherHandlerInterface
    {
        public function handle(GetVoucherQuery $query): VoucherDTO { /* ... */ }
    }
    ```

5. Create Value Objects (`VoucherId`) and domain exceptions (`VoucherException`, `VoucherNotFoundException`) under the same namespace tree.

6. Replace every ObjectModel call site in controllers, hook listeners and services with a bus dispatch:

    ```php
    // Before
    $voucher = new MymoduleVoucher();
    $voucher->code = $code;
    $voucher->add();

    // After
    $id = $this->commandBus->handle(new CreateVoucherCommand($code, $discount));
    ```

7. With `autowire: true` and `autoconfigure: true` in `config/services.yml`, the `#[AsCommandHandler]` and `#[AsQueryHandler]` attributes auto-register the handlers on the Messenger bus.

8. Once all consumers are migrated, delete the ObjectModel class file or mark it deprecated if external code still references it.

## Do

- Keep Commands immutable (readonly properties, validation in constructor).
- Return Value Objects from create handlers, `void` from update/delete handlers.
- Wrap domain invariants in the handler, not the controller.

## Don't

- Don't call `ObjectModel::add()` / `update()` / `delete()` from controllers after migration.
- Don't return Doctrine entities from handlers; return Value Objects or DTOs.
- Don't chain handlers; use a domain service for cross-aggregate orchestration.

## Related skills

- `module-add-cqrs-command` - greenfield CQRS Command scaffolding.
- `module-add-cqrs-query` - greenfield CQRS Query scaffolding.
- `migrate-objectmodel-to-doctrine` - replacing the persistence layer with Doctrine entities.
