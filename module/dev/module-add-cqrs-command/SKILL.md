---
name: module-add-cqrs-command
description: Scaffold a CQRS Command and CommandHandler in a PrestaShop 9 module to mutate state via the Symfony Messenger command bus instead of ObjectModel. Use when the user wants to create, update, delete or otherwise change an aggregate from a controller, console command, or hook listener.
---

## Requirements

Ask the user:
* Aggregate name in PascalCase singular (e.g. `Voucher`, `Subscription`). This becomes the namespace `Domain\<Aggregate>\...`.
* Action verb in PascalCase (`Create`, `Update`, `Delete`, `ToggleStatus`, `Bulk<Action>`). Combined with the aggregate it forms `<Verb><Aggregate>Command`.
* Input fields with PHP types (scalar primitives or Value Objects, never raw `ObjectModel` instances).
* Expected return: usually a Value Object identifier such as `<Aggregate>Id` for `Create` commands; `void` for `Update`/`Delete`.
* Domain rules that must be enforced in the handler (uniqueness, transitions, multistore, etc.) so failures map to dedicated exceptions.

## Steps

1. Create the directory layout under your module root. Each concern lives in its own sub-namespace so the boundary is obvious:

    ```
    src/Domain/<Aggregate>/
    ├── Command/<Verb><Aggregate>Command.php
    ├── CommandHandler/<Verb><Aggregate>CommandHandlerInterface.php
    ├── CommandHandler/<Verb><Aggregate>Handler.php
    ├── Exception/<Aggregate>Exception.php
    ├── Exception/<SpecificFailure>Exception.php
    ├── ValueObject/<Aggregate>Id.php
    └── Result/<Verb><Aggregate>Result.php   # only when the handler returns more than an ID
    ```

2. Write the Command as an immutable DTO. Use PHP 8 readonly properties; provide getters only, never setters. Validate trivially-checkable inputs in the constructor and throw a domain exception on failure:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Domain\Voucher\Command;

    use MyVendor\Mymodule\Domain\Voucher\Exception\VoucherConstraintException;

    final class CreateVoucherCommand
    {
        public function __construct(
            public readonly string $code,
            public readonly int $discountPercent,
            public readonly \DateTimeImmutable $expiresAt,
        ) {
            if ($discountPercent < 1 || $discountPercent > 100) {
                throw new VoucherConstraintException('Discount must be between 1 and 100.');
            }
        }
    }
    ```

3. Write the CommandHandler interface alongside the command handler. Mark the handler class with `#[AsCommandHandler]` from `PrestaShop\PrestaShop\Core\CommandBus\Attributes\` so PrestaShop's `CommandAndQueryRegisterPass` auto-tags it as a `messenger.message_handler` with `handles:` derived from the typed `handle()` parameter:

    ```php
    <?php
    namespace MyVendor\Mymodule\Domain\Voucher\CommandHandler;

    use MyVendor\Mymodule\Domain\Voucher\Command\CreateVoucherCommand;
    use MyVendor\Mymodule\Domain\Voucher\ValueObject\VoucherId;

    interface CreateVoucherCommandHandlerInterface
    {
        public function handle(CreateVoucherCommand $command): VoucherId;
    }
    ```

    ```php
    <?php
    namespace MyVendor\Mymodule\Domain\Voucher\CommandHandler;

    use Doctrine\ORM\EntityManagerInterface;
    use MyVendor\Mymodule\Domain\Voucher\Command\CreateVoucherCommand;
    use MyVendor\Mymodule\Domain\Voucher\Exception\VoucherException;
    use MyVendor\Mymodule\Domain\Voucher\ValueObject\VoucherId;
    use MyVendor\Mymodule\Entity\Voucher;
    use PrestaShop\PrestaShop\Core\CommandBus\Attributes\AsCommandHandler;

    #[AsCommandHandler]
    final class CreateVoucherHandler implements CreateVoucherCommandHandlerInterface
    {
        public function __construct(private readonly EntityManagerInterface $em) {}

        public function handle(CreateVoucherCommand $command): VoucherId
        {
            try {
                $voucher = new Voucher($command->code, $command->discountPercent, $command->expiresAt);
                $this->em->persist($voucher);
                $this->em->flush();
            } catch (\Throwable $e) {
                throw new VoucherException('Unable to persist the voucher.', 0, $e);
            }

            return new VoucherId($voucher->getId());
        }
    }
    ```

4. Domain exceptions extend a module-local base that extends `PrestaShopException`. This lets controllers catch one type and still report meaningful messages:

    ```php
    <?php
    namespace MyVendor\Mymodule\Domain\Voucher\Exception;

    class VoucherException extends \PrestaShopException {}

    final class VoucherConstraintException extends VoucherException {}
    final class VoucherNotFoundException extends VoucherException {}
    ```

5. Registration is automatic. With the default `services.yml` from `module-create` (`autowire: true`, `autoconfigure: true`, `resource:` glob over `src/`), the `#[AsCommandHandler]` attribute is picked up by `PrestaShopBundle\DependencyInjection\Compiler\CommandAndQueryRegisterPass`, which adds `messenger.message_handler` with `handles: <CommandFQCN>` (read from the typed `handle()` parameter). Do NOT add a `tactician.handler` tag - PS 9 ships Symfony Messenger and that tag is unused. If you cannot use attributes (PHP < 8 code path), tag the service manually:

    ```yaml
    services:
      MyVendor\Mymodule\Domain\Voucher\CommandHandler\CreateVoucherHandler:
        autowire: true
        tags:
          - { name: 'messenger.message_handler', handles: 'MyVendor\Mymodule\Domain\Voucher\Command\CreateVoucherCommand' }
    ```

6. Dispatch from a modern admin controller. Inject `PrestaShop\PrestaShop\Core\CommandBus\CommandBusInterface` (the same bus instance handles commands and queries) and call `handle()`:

    ```php
    <?php
    namespace MyVendor\Mymodule\Controller\Admin;

    use MyVendor\Mymodule\Domain\Voucher\Command\CreateVoucherCommand;
    use MyVendor\Mymodule\Domain\Voucher\Exception\VoucherException;
    use MyVendor\Mymodule\Domain\Voucher\ValueObject\VoucherId;
    use PrestaShop\PrestaShop\Core\CommandBus\CommandBusInterface;
    use PrestaShopBundle\Controller\Admin\PrestaShopAdminController;
    use Symfony\Component\DependencyInjection\Attribute\Autowire;
    use Symfony\Component\HttpFoundation\RedirectResponse;
    use Symfony\Component\HttpFoundation\Request;

    class VoucherController extends PrestaShopAdminController
    {
        public function createAction(
            Request $request,
            #[Autowire(service: 'prestashop.core.command_bus')]
            CommandBusInterface $commandBus,
        ): RedirectResponse {
            try {
                /** @var VoucherId $id */
                $id = $commandBus->handle(new CreateVoucherCommand(
                    (string) $request->request->get('code'),
                    (int) $request->request->get('discount'),
                    new \DateTimeImmutable((string) $request->request->get('expires_at')),
                ));
                $this->addFlash('success', $this->trans('Voucher %id% created.', ['%id%' => $id->getValue()], 'Modules.Mymodule.Admin'));
            } catch (VoucherException $e) {
                $this->addFlash('error', $e->getMessage());
            }

            return $this->redirectToRoute('mymodule_voucher_index');
        }
    }
    ```

## Do

- Keep the controller thin: validate the request, build the Command, dispatch, redirect or render. Business rules belong in the handler.
- Make every Command immutable (`readonly` properties, no setters). The bus may serialize, replay or queue it.
- Wrap inputs that have invariants in Value Objects (`VoucherId`, `EmailAddress`, `Money`) so they cannot exist in an invalid state.
- Map domain failures to dedicated exceptions extending `<Aggregate>Exception`. The HTTP layer translates them into flashes or 4xx responses.

## Don't

- Don't call `ObjectModel::add()`, `ObjectModel::update()` or `ObjectModel::delete()` from a controller. Those calls bypass the bus, the domain rules and the multistore context.
- Don't mutate a Command after construction. If a value must change, build a new Command.
- Don't call another handler from inside a handler. Cross-aggregate orchestration belongs to a domain service or a process manager, not to handler chaining.
- Don't return Doctrine entities or `ObjectModel` instances from a handler. Return a Value Object (`<Aggregate>Id`) or a small, framework-agnostic Result DTO.

## Canonical examples

- [devdocs - CQRS](https://devdocs.prestashop-project.org/9/development/architecture/domain/cqrs/).
- [devdocs - Domain exceptions](https://devdocs.prestashop-project.org/9/development/architecture/domain/domain-exceptions/).
- [devdocs - Domain references (full list of core commands)](https://devdocs.prestashop-project.org/9/development/architecture/domain/references/).
- Core canonical layout: [`src/Core/Domain/Customer`](https://github.com/PrestaShop/PrestaShop/tree/develop/src/Core/Domain/Customer) and [`src/Core/Domain/Product`](https://github.com/PrestaShop/PrestaShop/tree/develop/src/Core/Domain/Product) under `PrestaShop/PrestaShop`.
- Bus contract: [`CommandBusInterface`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/CommandBus/CommandBusInterface.php).
- Run `bin/console prestashop:list:commands-and-queries` against any PrestaShop install to discover the existing commands and copy their conventions.
