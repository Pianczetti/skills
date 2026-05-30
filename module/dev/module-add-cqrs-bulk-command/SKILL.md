---
name: module-add-cqrs-bulk-command
description: Scaffold a CQRS bulk-action Command and CommandHandler in a PrestaShop 9 module by extending AbstractBulkCommandHandler so a grid can delete, enable, disable or otherwise mutate many rows in one request while collecting per-item failures instead of aborting on the first error. Use when the user wires bulk actions into a Grid and needs the back-end to process a selection of ids.
---

## Requirements

Ask the user:
* Aggregate name in PascalCase singular (e.g. `Voucher`, `Subscription`). This becomes the namespace `Domain\<Aggregate>\...` and reuses the single-entity command tree from `module-add-cqrs-command`.
* Bulk action(s) needed: `BulkDelete`, `BulkUpdateStatus` (enable/disable), or a custom verb. Each action is its own command + handler.
* The id type each item carries. The grid passes scalar `int` ids by default; a domain that keys on a Value Object (e.g. `SearchTerm`) passes that instead. Keep it consistent with the single-entity commands.
* For a status action: confirm the command carries the **target** state (`bool $expectedStatus`), not a "flip current value" semantic.
* The repository / handler that performs the single-item operation (usually the same one used by `Delete<Aggregate>Command` / `UpdateStatus<Aggregate>Command`).

## Steps

1. Create the bulk commands next to the single-entity ones under `src/Domain/<Aggregate>/Command/`. A bulk command is a thin immutable DTO carrying the selection. Keep the id type scalar (`int[]`) to stay serializable:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Domain\Voucher\Command;

    final class BulkDeleteVouchersCommand
    {
        /** @param int[] $ids */
        public function __construct(private readonly array $ids)
        {
        }

        /** @return int[] */
        public function getIds(): array
        {
            return $this->ids;
        }
    }
    ```

    For a status toggle, carry the **target** state so the result is deterministic regardless of each row's current value:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Domain\Voucher\Command;

    final class BulkUpdateVoucherStatusCommand
    {
        /** @param int[] $ids */
        public function __construct(
            private readonly array $ids,
            private readonly bool $expectedStatus,
        ) {
        }

        /** @return int[] */
        public function getIds(): array
        {
            return $this->ids;
        }

        public function getExpectedStatus(): bool
        {
            return $this->expectedStatus;
        }
    }
    ```

2. Add a bulk exception under `src/Domain/<Aggregate>/Exception/`. It extends your module's `<Aggregate>Exception` base (see `module-add-cqrs-command`) AND implements the core `BulkCommandExceptionInterface`, so the HTTP layer can list every individual failure instead of a single message:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Domain\Voucher\Exception;

    use PrestaShop\PrestaShop\Core\Domain\Exception\BulkCommandExceptionInterface;

    class BulkVoucherException extends VoucherException implements BulkCommandExceptionInterface
    {
        /** @param \Throwable[] $exceptions */
        public function __construct(
            private readonly array $exceptions,
            string $message = 'Errors occurred during Voucher bulk action',
            int $code = 0,
            ?\Throwable $previous = null,
        ) {
            parent::__construct($message, $code, $previous);
        }

        /** @return \Throwable[] */
        public function getExceptions(): array
        {
            return $this->exceptions;
        }
    }
    ```

3. Write the handler interface alongside the single-entity ones under `src/Domain/<Aggregate>/CommandHandler/`. Bulk handlers always return `void` - the failures travel through the thrown bulk exception, never a return value:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Domain\Voucher\CommandHandler;

    use MyVendor\Mymodule\Domain\Voucher\Command\BulkDeleteVouchersCommand;

    interface BulkDeleteVouchersHandlerInterface
    {
        public function handle(BulkDeleteVouchersCommand $command): void;
    }
    ```

4. Write the handler. Extend `PrestaShop\PrestaShop\Core\Domain\AbstractBulkCommandHandler`, mark it `#[AsCommandHandler]` (auto-registered exactly like a normal command handler - see `module-add-cqrs-command`), and implement the three abstract hooks:

    * `handle()` calls `$this->handleBulkAction($ids, <ExceptionToCatch>::class, $command)`. The second argument is the **single-item** exception type that is safe to collect and continue past; any other throwable aborts the whole batch and bubbles up.
    * `handleSingleAction()` performs the per-row work (delegate to the same repository the single-entity handler uses).
    * `buildBulkException()` wraps the collected throwables into your `Bulk<Aggregate>Exception`.
    * `supports()` guards the id type.

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Domain\Voucher\CommandHandler;

    use MyVendor\Mymodule\Domain\Voucher\Command\BulkDeleteVouchersCommand;
    use MyVendor\Mymodule\Domain\Voucher\Exception\BulkVoucherException;
    use MyVendor\Mymodule\Domain\Voucher\Exception\VoucherException;
    use MyVendor\Mymodule\Repository\VoucherRepository;
    use PrestaShop\PrestaShop\Core\CommandBus\Attributes\AsCommandHandler;
    use PrestaShop\PrestaShop\Core\Domain\AbstractBulkCommandHandler;
    use PrestaShop\PrestaShop\Core\Domain\Exception\BulkCommandExceptionInterface;

    #[AsCommandHandler]
    final class BulkDeleteVouchersHandler extends AbstractBulkCommandHandler implements BulkDeleteVouchersHandlerInterface
    {
        public function __construct(private readonly VoucherRepository $repository)
        {
        }

        public function handle(BulkDeleteVouchersCommand $command): void
        {
            // VoucherException = the type we collect-and-continue on; everything else aborts.
            $this->handleBulkAction($command->getIds(), VoucherException::class, $command);
        }

        protected function handleSingleAction(mixed $id, mixed $command): void
        {
            $this->repository->delete((int) $id);
        }

        protected function buildBulkException(array $caughtExceptions): BulkCommandExceptionInterface
        {
            return new BulkVoucherException($caughtExceptions, 'Errors occurred during Voucher bulk delete action');
        }

        protected function supports($id): bool
        {
            return is_numeric($id);
        }
    }
    ```

    A status handler is the same shape; `handleSingleAction()` reads the target state off the command instead of flipping the current value:

    ```php
    protected function handleSingleAction(mixed $id, mixed $command): void
    {
        /** @var \MyVendor\Mymodule\Domain\Voucher\Command\BulkUpdateVoucherStatusCommand $command */
        $this->repository->updateStatus((int) $id, $command->getExpectedStatus());
    }
    ```

5. Registration is automatic. With the default `services.yml` from `module-create` (`autowire`, `autoconfigure`, `resource:` glob over `src/`), the `#[AsCommandHandler]` attribute is picked up by `CommandAndQueryRegisterPass`, which derives `handles:` from the typed `handle()` parameter. No manual tag is required.

6. Dispatch from the controller action wired to the grid bulk action. Read the selected ids from the request, build the bulk command, and translate a thrown `BulkCommandExceptionInterface` into a partial-success message that names every failure:

    ```php
    <?php
    namespace MyVendor\Mymodule\Controller\Admin;

    use MyVendor\Mymodule\Domain\Voucher\Command\BulkDeleteVouchersCommand;
    use PrestaShop\PrestaShop\Core\CommandBus\CommandBusInterface;
    use PrestaShop\PrestaShop\Core\Domain\Exception\BulkCommandExceptionInterface;
    use PrestaShopBundle\Controller\Admin\PrestaShopAdminController;
    use Symfony\Component\DependencyInjection\Attribute\Autowire;
    use Symfony\Component\HttpFoundation\RedirectResponse;
    use Symfony\Component\HttpFoundation\Request;

    class VoucherController extends PrestaShopAdminController
    {
        public function bulkDeleteAction(
            Request $request,
            #[Autowire(service: 'prestashop.core.command_bus')]
            CommandBusInterface $commandBus,
        ): RedirectResponse {
            $ids = array_map('intval', $request->request->all('voucher_bulk'));

            try {
                $commandBus->handle(new BulkDeleteVouchersCommand($ids));
                $this->addFlash('success', $this->trans('The selection has been deleted.', [], 'Admin.Notifications.Success'));
            } catch (BulkCommandExceptionInterface $e) {
                // One flash per failed item; the batch still deleted everything it could.
                $this->addFlash('error', $this->trans(
                    '%count% item(s) could not be deleted.',
                    ['%count%' => count($e->getExceptions())],
                    'Modules.Mymodule.Admin'
                ));
            }

            return $this->redirectToRoute('mymodule_voucher_index');
        }
    }
    ```

7. Wire the grid bulk action to that route. In the `GridDefinitionFactory` from `module-add-grid`, bulk actions that mutate state MUST use a POST-style action, and the route MUST be `methods: [POST]`:

    ```php
    use PrestaShop\PrestaShop\Core\Grid\Action\Bulk\Type\SubmitBulkAction;

    protected function getBulkActions(): BulkActionCollection
    {
        return (new BulkActionCollection())
            ->add((new SubmitBulkAction('delete_selection'))
                ->setName($this->trans('Delete selected', [], 'Admin.Actions'))
                ->setOptions([
                    'submit_route' => 'mymodule_voucher_bulk_delete',
                    'confirm_message' => $this->trans('Delete the selected vouchers?', [], 'Modules.Mymodule.Admin'),
                ])
            );
    }
    ```

    ```yaml
    # config/routes.yml
    mymodule_voucher_bulk_delete:
      path: /modules/mymodule/vouchers/bulk-delete
      methods: [POST]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\VoucherController::bulkDeleteAction'
        _legacy_controller: 'AdminMymoduleVouchers'
    ```

## Do

- Extend `AbstractBulkCommandHandler` and implement `handleSingleAction()`, `buildBulkException()` and `supports()`. This is the only sanctioned bulk pattern in PS 9 - it gives you collect-and-continue for free.
- Pass the **single-item** exception type to `handleBulkAction()` as the catchable type. Only failures of that type are collected; anything else aborts the batch so genuine bugs are not silently swallowed.
- Report every failure: build the bulk exception from the full `$caughtExceptions` array and surface `getExceptions()` count (or messages) to the merchant. A bulk action that "half-worked" must say so.
- Carry the target state in a status command (`bool $expectedStatus`). Bulk enable and bulk disable are two dispatches of the same command with `true` / `false`.
- Keep `handleSingleAction()` delegating to the same repository method the single-entity handler uses, so single and bulk paths stay consistent.

## Don't

- Don't loop over ids in the controller calling the single command N times. That bypasses the per-item error collection and produces one flash per row or a hard stop on the first failure.
- Don't abort the whole batch on the first failure. The base class already continues past the catchable exception type - don't re-throw inside `handleSingleAction()` for an expected, per-item error.
- Don't return the failures from the handler. Bulk handlers return `void`; failures travel through the thrown `BulkCommandExceptionInterface`.
- Don't catch a broad `\Throwable` / `\Exception` as the catchable type. Use the narrow domain exception so infrastructure errors (DB down, type errors) still abort and bubble up.
- Don't use a `LinkRowAction`-style GET for a bulk mutation, and don't forget `methods: [POST]` on the route. State-changing endpoints must be POST.

## Canonical examples

- [devdocs - CQRS](https://devdocs.prestashop-project.org/9/development/architecture/domain/cqrs/) and [Domain references](https://devdocs.prestashop-project.org/9/development/architecture/domain/references/) (browse any domain's `Bulk*Command` entries, e.g. Customer's `BulkDeleteCustomerCommand`, `BulkEnableCustomerCommand`).
- Base class: [`AbstractBulkCommandHandler`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Domain/AbstractBulkCommandHandler.php) and the [`BulkCommandExceptionInterface`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Domain/Exception/BulkCommandExceptionInterface.php) contract.
- Reference handlers in core: `src/Adapter/Alias/CommandHandler/BulkDeleteSearchTermsAliasesHandler.php` and `src/Adapter/Cart/CommandHandler/BulkDeleteCartHandler.php` under [`PrestaShop/PrestaShop`](https://github.com/PrestaShop/PrestaShop/tree/develop/src/Adapter).
- Run `bin/console prestashop:list:commands-and-queries` against any install to list the existing `Bulk*` commands and copy their conventions.

## Related skills

- `module-add-cqrs-command` - the single-entity command/handler tree this skill extends (shares the `<Aggregate>Exception` base and the repository).
- `module-add-grid` - the Grid that exposes the bulk action UI and posts the selection to the controller.
- `module-add-symfony-route` - the POST route declaration cheatsheet (`_legacy_controller`).
- `module-add-behat-tests` - cover the collect-and-continue behaviour with a bulk scenario.
- `module-add-doctrine-entity` - the repository whose delete / status method `handleSingleAction()` delegates to.
