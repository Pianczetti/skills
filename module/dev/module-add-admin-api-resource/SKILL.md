---
name: module-add-admin-api-resource
description: Expose a PrestaShop 9 module aggregate through the modern Admin API by declaring an API Platform resource that wires into the CQRS bus via #[CQRSGet], #[CQRSCreate], #[CQRSPartialUpdate] and #[CQRSDelete]. Use when the user wants a server-to-server REST endpoint authenticated with OAuth2 client credentials.
---

## Requirements

Ask the user:
* Resource name in PascalCase singular (e.g. `Voucher`, `Subscription`). The class lives at `src/ApiPlatform/Resources/<Resource>/<Resource>.php`.
* URI prefix in `kebab-case` plural (e.g. `/vouchers`). API Platform mounts it under `/admin-api/...` automatically.
* Allowed operations: any combination of `GET`, `POST`, `PATCH`, `DELETE`. Each maps to a CQRS attribute below.
* OAuth2 scope names in `lower_snake_case`, prefixed with `module_<modulename>_<resource>_` to avoid collisions with core scopes (`customer_read`, `product_write`, etc.). Two scopes per resource is the standard split: `module_mymod_voucher_read` and `module_mymod_voucher_write`.
* Existing CQRS Command/Query FQCNs that the operations will dispatch. Each operation references one. If they do not exist yet, create them first via [`module-add-cqrs-command`](../module-add-cqrs-command/SKILL.md) and [`module-add-cqrs-query`](../module-add-cqrs-query/SKILL.md) - the API resource is a thin DTO; all business logic lives in the handlers.

## Steps

1. Place every resource under `src/ApiPlatform/Resources/<Resource>/`. PrestaShop scans this folder automatically (handled by `PrestaShopExtension`); you do NOT register the resource manually in `services.yml` or `api_platform.yml`. From the [API Resources docs](https://devdocs.prestashop-project.org/9/admin-api/resource_server/api-resources/):

    > resources in modules are in the `src/ApiPlatform/Resources` [...] The endpoints defined in modules resources are only usable when the module is installed and enabled. However, the scopes defined in the modules are scanned as long as the module is installed, this allows assigning them to clients before you enable the module and its endpoints.

    Layout for a resource called `Voucher`:

    ```
    src/ApiPlatform/Resources/Voucher/
    └── Voucher.php
    ```

2. Write the resource as a plain PHP DTO whose public properties are the API contract. Declare a single `#[ApiResource]` attribute with all CQRS operations grouped under `operations:`. The four CQRS attribute classes live in `PrestaShopBundle\ApiPlatform\Metadata\`:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\ApiPlatform\Resources\Voucher;

    use ApiPlatform\Metadata\ApiProperty;
    use ApiPlatform\Metadata\ApiResource;
    use MyVendor\Mymodule\Domain\Voucher\Command\AddVoucherCommand;
    use MyVendor\Mymodule\Domain\Voucher\Command\DeleteVoucherCommand;
    use MyVendor\Mymodule\Domain\Voucher\Command\EditVoucherCommand;
    use MyVendor\Mymodule\Domain\Voucher\Exception\VoucherException;
    use MyVendor\Mymodule\Domain\Voucher\Exception\VoucherNotFoundException;
    use MyVendor\Mymodule\Domain\Voucher\Query\GetVoucherForEditing;
    use PrestaShopBundle\ApiPlatform\Metadata\CQRSCreate;
    use PrestaShopBundle\ApiPlatform\Metadata\CQRSDelete;
    use PrestaShopBundle\ApiPlatform\Metadata\CQRSGet;
    use PrestaShopBundle\ApiPlatform\Metadata\CQRSPartialUpdate;
    use Symfony\Component\HttpFoundation\Response;

    #[ApiResource(
        operations: [
            new CQRSGet(
                uriTemplate: '/vouchers/{voucherId}',
                requirements: ['voucherId' => '\d+'],
                CQRSQuery: GetVoucherForEditing::class,
                scopes: ['module_mymod_voucher_read'],
            ),
            new CQRSCreate(
                uriTemplate: '/vouchers',
                CQRSCommand: AddVoucherCommand::class,
                CQRSQuery: GetVoucherForEditing::class,
                scopes: ['module_mymod_voucher_write'],
            ),
            new CQRSPartialUpdate(
                uriTemplate: '/vouchers/{voucherId}',
                requirements: ['voucherId' => '\d+'],
                read: false,
                CQRSCommand: EditVoucherCommand::class,
                CQRSQuery: GetVoucherForEditing::class,
                scopes: ['module_mymod_voucher_write'],
            ),
            new CQRSDelete(
                uriTemplate: '/vouchers/{voucherId}',
                requirements: ['voucherId' => '\d+'],
                CQRSCommand: DeleteVoucherCommand::class,
                scopes: ['module_mymod_voucher_write'],
            ),
        ],
        exceptionToStatus: [
            VoucherNotFoundException::class => Response::HTTP_NOT_FOUND,
            VoucherException::class => Response::HTTP_UNPROCESSABLE_ENTITY,
        ],
    )]
    class Voucher
    {
        #[ApiProperty(identifier: true, openapiContext: ['type' => 'integer', 'example' => 1])]
        public int $voucherId;

        public string $code;

        public int $discountPercent;

        public string $expiresAt;

        public bool $enabled;
    }
    ```

3. The API Platform framework calls the bus for you. PrestaShop ships three components that read the CQRS attributes and dispatch through the bus:

    * `PrestaShopBundle\ApiPlatform\Provider\QueryProvider` for single-resource reads (`#[CQRSGet]`).
    * `PrestaShopBundle\ApiPlatform\Provider\QueryListProvider` for collection reads (`#[CQRSGetCollection]`).
    * `PrestaShopBundle\ApiPlatform\Processor\CommandProcessor` for writes (`#[CQRSCreate]`, `#[CQRSPartialUpdate]`, `#[CQRSDelete]`).

    Do NOT write a Symfony controller for these endpoints. The HTTP layer is generated from the attributes, the operation references the CQRS Command/Query, and the bus + handlers do the rest.

4. Optional mapping when the API field names differ from the Command/Query fields. Both attributes accept `CQRSQueryMapping` and `CQRSCommandMapping` constants; declare them on the class:

    ```php
    public const QUERY_MAPPING = [
        '[expires_at]' => '[expiresAt]',
    ];

    public const COMMAND_MAPPING = [
        '[expires_at]' => '[expiresAt]',
    ];
    ```

    Reference them in the operation: `CQRSQueryMapping: self::QUERY_MAPPING, CQRSCommandMapping: self::COMMAND_MAPPING`.

5. Scopes are picked up automatically from the attributes. Once the module is installed, the merchant can grant them to an API client in the BO under `Advanced Parameters → Admin API → Add new API Client`. The scope names appear in the multi-select exactly as declared.

6. Authenticate with OAuth2 client credentials from a third-party caller. The token endpoint is the internal authorization server unless an external one is configured (see [authorization servers](https://devdocs.prestashop-project.org/9/admin-api/authorization_server/)):

    ```bash
    # 1. Fetch a JWT access token, requesting only the scopes you need for this call
    curl -sS -X POST https://shop.example/admin-api/oauth2/token \
      -H 'Content-Type: application/x-www-form-urlencoded' \
      -d 'grant_type=client_credentials' \
      -d 'client_id=YOUR_CLIENT_ID' \
      -d 'client_secret=YOUR_CLIENT_SECRET' \
      -d 'scope=module_mymod_voucher_read module_mymod_voucher_write'

    # 2. Call the resource with Authorization: Bearer <access_token>
    curl -sS https://shop.example/admin-api/vouchers/42 \
      -H 'Authorization: Bearer eyJhbGciOi...'
    ```

    A 403 with `"detail": "Invalid scope."` means the access token did not include the scope required by the operation.

7. In production, the API requires HTTPS with TLSv1.2 or higher. The "Force security in debug mode" toggle is only available with debug mode enabled (`Advanced Parameters → Admin API → Configuration`).

## Do

- Use the same scope-naming convention everywhere in the module: `module_<modulename>_<resource>_read` and `module_<modulename>_<resource>_write`. Two scopes per resource is enough; finer-grained scopes are rarely worth the matrix explosion.
- Keep the resource class field-list minimal. Properties not present in a request payload are ignored by the framework; you do not need a separate DTO per operation as long as the same shape makes sense.
- Map domain exceptions to HTTP status codes via `exceptionToStatus`. The framework otherwise returns 500 on every uncaught domain exception.
- Validate inputs with Symfony validator constraints on the DTO properties (`#[Assert\NotBlank(groups: ['Create'])]`) and reference the group via `validationContext` on `CQRSCreate`. The bus does not re-validate.

## Don't

- Don't write a Symfony controller for these endpoints. The CQRS attributes wire the bus automatically; a controller bypasses the API Platform serialisation, scope check and OpenAPI generation.
- Don't reuse a core scope (`customer_read`, `product_write`, ...) for a module endpoint. Module scopes must be namespaced with `module_<modulename>_` so merchants can audit them in the API Client form.
- Don't return Doctrine entities or `ObjectModel` instances from the underlying Query handler. The DTO is the contract; the handler returns a flat result that maps onto its public properties.
- Don't put the resource files outside `src/ApiPlatform/Resources/`. PrestaShopExtension's auto-scan only looks there; resources placed elsewhere are silently ignored.
- Don't authenticate with cookies or admin sessions. The Admin API is OAuth2-only and rejects every request that does not carry a valid `Authorization: Bearer` header.

## Canonical examples

- [devdocs - Admin API overview](https://devdocs.prestashop-project.org/9/admin-api/).
- [devdocs - API Resources (declares `CQRSGet`, `CQRSCreate`, `CQRSPartialUpdate`, `CQRSDelete` and `CQRSGetCollection`)](https://devdocs.prestashop-project.org/9/admin-api/resource_server/api-resources/).
- [devdocs - API Platform integration (Provider/Processor wiring)](https://devdocs.prestashop-project.org/9/admin-api/resource_server/api-platform/).
- [devdocs - OAuth2 / client credentials flow](https://devdocs.prestashop-project.org/9/admin-api/oauth/).
- [devdocs - How to use the Admin API (token + Bearer + scopes)](https://devdocs.prestashop-project.org/9/admin-api/how-to-use/).
- [devdocs - Multi-shop context for the API](https://devdocs.prestashop-project.org/9/admin-api/multi-shop/).
- [devdocs - Contributing new endpoints (full step-by-step)](https://devdocs.prestashop-project.org/9/admin-api/contribute-to-core-api/).
- [`PrestaShop/ps_apiresources`](https://github.com/PrestaShop/ps_apiresources) - the canonical module that ships every core API resource. Browse [`src/ApiPlatform/Resources/`](https://github.com/PrestaShop/ps_apiresources/tree/dev/src/ApiPlatform/Resources) to copy patterns.
- Full GET/POST/PATCH/DELETE example: [`Customer.php`](https://github.com/PrestaShop/ps_apiresources/blob/dev/src/ApiPlatform/Resources/Customer/Customer.php).
- Search-style collection: [`FoundCartRule.php`](https://github.com/PrestaShop/ps_apiresources/blob/dev/src/ApiPlatform/Resources/CartRule/FoundCartRule.php).
- Example-modules sample to scaffold from: [`example-modules/api_module`](https://github.com/PrestaShop/example-modules/tree/master/api_module).
