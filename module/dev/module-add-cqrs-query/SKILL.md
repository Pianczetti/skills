---
name: module-add-cqrs-query
description: Scaffold a CQRS Query and QueryHandler in a PrestaShop 9 module to read data through the Tactician bus instead of pulling Doctrine entities or ObjectModels directly into a controller. Use when the user wants to fetch data for a Back Office page, an API endpoint, or a presenter.
---

## Requirements

Ask the user:
* Query name in the form `Get<X>For<Purpose>` (e.g. `GetVoucherForEditing`, `GetVouchersListForGrid`, `SearchCustomersForOrderCreation`). The pattern keeps queries discoverable and intent-revealing.
* Filter fields with PHP types (search criteria, IDs, language ID, shop ID).
* Result DTO shape: which scalar fields and Value Objects must the consumer receive, and is it a single object, a collection, or a paginated chunk.
* Whether the consumer needs translatable fields - if yes, the query receives `langId` and the handler joins `<entity>_lang`.

## Steps

1. Create the directory layout under your module root. Queries, handlers and result DTOs each live in their own sub-namespace:

    ```
    src/Domain/<Aggregate>/
    ├── Query/<Name>Query.php
    ├── QueryHandler/<Name>QueryHandlerInterface.php
    ├── QueryHandler/<Name>Handler.php
    └── QueryResult/<Name>.php
    ```

2. Write the Query as an immutable DTO. Queries never mutate state, so the constructor only stores the criteria:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Domain\Voucher\Query;

    final class GetVoucherForEditing
    {
        public function __construct(
            public readonly int $voucherId,
            public readonly int $langId,
        ) {}
    }
    ```

3. Define the QueryHandler interface and its implementation. The handler reads from Doctrine repositories or DBAL but never returns the underlying entity:

    ```php
    <?php
    namespace MyVendor\Mymodule\Domain\Voucher\QueryHandler;

    use MyVendor\Mymodule\Domain\Voucher\Query\GetVoucherForEditing;
    use MyVendor\Mymodule\Domain\Voucher\QueryResult\EditableVoucher;

    interface GetVoucherForEditingHandlerInterface
    {
        public function handle(GetVoucherForEditing $query): EditableVoucher;
    }
    ```

    ```php
    <?php
    namespace MyVendor\Mymodule\Domain\Voucher\QueryHandler;

    use MyVendor\Mymodule\Domain\Voucher\Exception\VoucherNotFoundException;
    use MyVendor\Mymodule\Domain\Voucher\Query\GetVoucherForEditing;
    use MyVendor\Mymodule\Domain\Voucher\QueryResult\EditableVoucher;
    use MyVendor\Mymodule\Repository\VoucherRepository;

    final class GetVoucherForEditingHandler implements GetVoucherForEditingHandlerInterface
    {
        public function __construct(private readonly VoucherRepository $repository) {}

        public function handle(GetVoucherForEditing $query): EditableVoucher
        {
            $voucher = $this->repository->find($query->voucherId)
                ?? throw new VoucherNotFoundException(sprintf('Voucher #%d not found.', $query->voucherId));

            return new EditableVoucher(
                $voucher->getId(),
                $voucher->getCode(),
                $voucher->getDiscountPercent(),
                $voucher->getExpiresAt()->format(\DateTimeInterface::ATOM),
            );
        }
    }
    ```

4. Write the Result DTO with primitives and Value Objects only. No Doctrine entity, no `ObjectModel`, no callable, no resource:

    ```php
    <?php
    namespace MyVendor\Mymodule\Domain\Voucher\QueryResult;

    final class EditableVoucher
    {
        public function __construct(
            public readonly int $voucherId,
            public readonly string $code,
            public readonly int $discountPercent,
            public readonly string $expiresAt,
        ) {}
    }
    ```

5. Register the handler in `config/services.yml` with the `tactician.handler` tag pointing at the Query FQCN:

    ```yaml
    services:
      MyVendor\Mymodule\Domain\Voucher\QueryHandler\GetVoucherForEditingHandler:
        autowire: true
        public: true
        tags:
          - { name: 'tactician.handler', command: 'MyVendor\Mymodule\Domain\Voucher\Query\GetVoucherForEditing' }
    ```

6. Dispatch from a modern admin controller. Use the same bus interface as for commands; PrestaShop wires the query bus under `prestashop.core.query_bus`:

    ```php
    <?php
    namespace MyVendor\Mymodule\Controller\Admin;

    use MyVendor\Mymodule\Domain\Voucher\Exception\VoucherNotFoundException;
    use MyVendor\Mymodule\Domain\Voucher\Query\GetVoucherForEditing;
    use MyVendor\Mymodule\Domain\Voucher\QueryResult\EditableVoucher;
    use PrestaShop\PrestaShop\Core\CommandBus\CommandBusInterface;
    use PrestaShopBundle\Controller\Admin\PrestaShopAdminController;
    use Symfony\Component\DependencyInjection\Attribute\Autowire;
    use Symfony\Component\HttpFoundation\Response;

    class VoucherController extends PrestaShopAdminController
    {
        public function editAction(
            int $voucherId,
            #[Autowire(service: 'prestashop.core.query_bus')]
            CommandBusInterface $queryBus,
        ): Response {
            try {
                /** @var EditableVoucher $voucher */
                $voucher = $queryBus->handle(new GetVoucherForEditing($voucherId, $this->getLanguageContext()->getId()));
            } catch (VoucherNotFoundException $e) {
                throw $this->createNotFoundException($e->getMessage(), $e);
            }

            return $this->render('@Modules/mymodule/views/templates/admin/voucher/edit.html.twig', ['voucher' => $voucher]);
        }
    }
    ```

7. Discover existing queries before writing a new one. From the PrestaShop project root:

    ```bash
    bin/console prestashop:list:commands-and-queries
    ```

   The command lists every Tactician handler registered in the container, with the Command/Query FQCN it accepts. Reuse a core query (e.g. `GetCustomerForViewing`, `SearchProducts`) when one exists rather than duplicating it in your module.

## Do

- Treat queries as read-only. If you find yourself emitting events or writing to the database in a handler, you wrote a Command, rename it.
- Keep result DTOs framework-agnostic. They are payloads for templates, JSON responses and JavaScript callers; they must not leak `EntityManager` lifecycles.
- Inject repositories, not the `EntityManager`. Repositories make the read intent explicit and easier to mock in unit tests.
- Throw a domain `<Aggregate>NotFoundException` (extending `<Aggregate>Exception`) when the criteria match nothing. The HTTP layer converts it to a 404.

## Don't

- Don't return Doctrine entities, `ObjectModel` instances, or arrays of either. The DTO contract is the entire output, including its structure.
- Don't bypass the bus by calling the handler directly from the controller. The bus enforces middleware (logging, transactions, validation).
- Don't reuse a Command for a read action. The bus only knows one handler per type, and Commands carry mutation intent in their name.
- Don't add side-effects to the constructor of a Query. The Query is a value, not a process.

## Canonical examples

- [devdocs - CQRS](https://devdocs.prestashop-project.org/9/development/architecture/domain/cqrs/).
- [devdocs - Data Transfer Objects](https://devdocs.prestashop-project.org/9/development/architecture/domain/data-transfer-objects/).
- [devdocs - Domain references (browse `Get*` and `Search*` queries per aggregate)](https://devdocs.prestashop-project.org/9/development/architecture/domain/references/).
- Core canonical layout: [`src/Core/Domain/Customer`](https://github.com/PrestaShop/PrestaShop/tree/develop/src/Core/Domain/Customer) - look at `Query/`, `QueryHandler/`, `QueryResult/`.
- Bus contract: [`CommandBusInterface`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/CommandBus/CommandBusInterface.php).
