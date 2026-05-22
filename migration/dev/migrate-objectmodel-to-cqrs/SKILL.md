---
name: migrate-objectmodel-to-cqrs
description: Replace ObjectModel CRUD calls (new MyObject(), ->add(), ->update(), ->delete()) with CQRS Commands and Queries dispatched on the Symfony Messenger bus. Use when a module mutates state via ObjectModel directly and must adopt the PS 9 domain-driven architecture.
---

## Requirements

- Identify all ObjectModel instantiation and mutation sites (controllers, hooks, services).
- List the aggregate name and the operations performed (create, update, delete, toggle).
- Confirm persistence backend: Doctrine entity (preferred) or legacy ObjectModel retained only inside the handler.

## Steps

1. **Identify the legacy pattern** being replaced:

    ```php
    // LEGACY - scattered across controllers/hooks
    $voucher = new MyModVoucher();
    $voucher->code = $request->get('code');
    $voucher->discount_percent = (int) $request->get('discount');
    $voucher->add();  // ObjectModel::add()

    $voucher = new MyModVoucher((int) $id);
    $voucher->code = 'NEW_CODE';
    $voucher->update();

    $voucher = new MyModVoucher((int) $id);
    $voucher->delete();
    ```

2. **Create the Command** as an immutable DTO with `readonly` properties:

    ```php
    <?php
    namespace MyVendor\Mymodule\Domain\Voucher\Command;

    final class CreateVoucherCommand
    {
        public function __construct(
            public readonly string $code,
            public readonly int $discountPercent,
        ) {}
    }
    ```

3. **Create the Handler** marked with `#[AsCommandHandler]`. The handler owns the persistence logic (Doctrine or legacy ObjectModel wrapped privately):

    ```php
    <?php
    namespace MyVendor\Mymodule\Domain\Voucher\CommandHandler;

    use Doctrine\ORM\EntityManagerInterface;
    use MyVendor\Mymodule\Domain\Voucher\Command\CreateVoucherCommand;
    use MyVendor\Mymodule\Domain\Voucher\ValueObject\VoucherId;
    use MyVendor\Mymodule\Entity\Voucher;
    use PrestaShop\PrestaShop\Core\CommandBus\Attributes\AsCommandHandler;

    #[AsCommandHandler]
    final class CreateVoucherHandler
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

4. **Repeat for Update and Delete:**

    ```php
    // UpdateVoucherCommand carries the ID + changed fields
    final class UpdateVoucherCommand
    {
        public function __construct(
            public readonly int $voucherId,
            public readonly string $code,
        ) {}
    }

    // DeleteVoucherCommand carries only the ID
    final class DeleteVoucherCommand
    {
        public function __construct(public readonly int $voucherId) {}
    }
    ```

5. **Replace call sites.** Inject `CommandBusInterface` and dispatch:

    ```php
    use PrestaShop\PrestaShop\Core\CommandBus\CommandBusInterface;

    // Before: $voucher = new MyModVoucher(); $voucher->code = ...; $voucher->add();
    // After:
    $id = $commandBus->handle(new CreateVoucherCommand($code, $discount));

    // Before: $voucher->update();
    // After:
    $commandBus->handle(new UpdateVoucherCommand($id, $newCode));

    // Before: $voucher->delete();
    // After:
    $commandBus->handle(new DeleteVoucherCommand($id));
    ```

6. **Registration is automatic** with `#[AsCommandHandler]` and a standard `services.yml` that has `autowire: true` + `autoconfigure: true`. No manual tags required.

7. **Create domain exceptions** extending a module-local base (`VoucherException extends \PrestaShopException`) for constraint violations, not-found, and persistence failures.

## Do

- Keep commands immutable; validate trivial constraints in the constructor.
- Return a Value Object (`VoucherId`) from create handlers, `void` from update/delete.
- Throw dedicated domain exceptions from handlers; let the controller translate them into flash messages.

## Don't

- Don't call `ObjectModel::add()`, `->update()`, or `->delete()` from controllers or hook listeners; always dispatch a command.
- Don't inject `ObjectModel` instances into commands; pass scalar IDs and let the handler load.
- Don't call one handler from another; use a domain service for cross-aggregate orchestration.
- Don't return Doctrine entities from a handler; return a Value Object or void.

## Canonical examples

- [devdocs - CQRS](https://devdocs.prestashop-project.org/9/development/architecture/domain/cqrs/)
- [devdocs - Domain exceptions](https://devdocs.prestashop-project.org/9/development/architecture/domain/domain-exceptions/)

## Related skills

- `module-add-cqrs-command` - full recipe for scaffolding commands and handlers.
- `migrate-objectmodel-to-doctrine` - migrating the persistence layer the handler uses.
