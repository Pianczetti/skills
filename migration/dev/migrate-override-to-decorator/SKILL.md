---
name: migrate-override-to-decorator
description: Replace override/ class files with Symfony service decorators or event listeners/subscribers. Use when a module ships overrides that conflict with PS 9 core changes and need a maintainable, upgrade-safe alternative.
---

## Requirements

- Module name and vendor namespace.
- List of files under `override/` with the classes and methods they override.
- The core service id each override targets (find via `bin/console debug:container <keyword>`).
- Whether each override modifies a service method (decorator) or reacts to an event (listener).

## Steps

1. Inventory overrides:

    ```bash
    find modules/mymodule/override/ -name '*.php' -exec echo {} \;
    ```

2. For each override, determine the migration path using this decision tree:
   - Override modifies a **service method** (business logic, query, rendering) -> **service decorator**.
   - Override reacts to a **lifecycle event** (before/after save, display, validation) -> **event listener or hook subscriber**.
   - Override adds a method that does not exist in the parent -> likely not an override pattern; extract to a standalone service.

3. For the **decorator** path, create a service implementing the same interface and use `#[AsDecorator]`:

    ```php
    namespace MyVendor\Mymodule\Decorator;

    use PrestaShop\PrestaShop\Core\Product\Search\ProductSearchProviderInterface;
    use Symfony\Component\DependencyInjection\Attribute\AsDecorator;

    #[AsDecorator('prestashop.core.product.search.provider')]
    class CustomSearchProvider implements ProductSearchProviderInterface
    {
        public function __construct(
            private readonly ProductSearchProviderInterface $inner,
        ) {}

        public function runQuery(/* ... */): ProductSearchResult
        {
            $result = $this->inner->runQuery(/* ... */);
            // custom logic
            return $result;
        }
    }
    ```

4. For the **listener** path, register a hook subscriber or Symfony event listener:

    ```php
    namespace MyVendor\Mymodule\EventListener;

    use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

    #[AsEventListener(event: 'kernel.response')]
    class CustomResponseListener
    {
        public function __invoke(\Symfony\Component\HttpKernel\Event\ResponseEvent $event): void
        {
            // modify response
        }
    }
    ```

5. Register in `config/services.yml` (if not using attributes with autoconfigure):

    ```yaml
    services:
      MyVendor\Mymodule\Decorator\CustomSearchProvider:
        decorates: 'prestashop.core.product.search.provider'
        arguments:
          $inner: '@.inner'
    ```

6. Delete the override file from `override/`. Update the module's `install()` / `uninstall()` to remove any `$this->addOverride()` / `$this->removeOverride()` calls.

7. Clear the class index cache and verify:

    ```bash
    bin/console cache:clear
    bin/console prestashop:module uninstall mymodule && bin/console prestashop:module install mymodule
    ```

8. If no decorator or listener can express the override (e.g. the target class has no service id and no events), document the limitation and keep the override as a last resort with a `@deprecated` comment pointing to the upstream issue requesting an extension point.

## Do

- Use `#[AsDecorator]` for clean decoration without manual YAML when possible.
- Always delegate to `$this->inner` for methods you do not need to modify.
- Test that the decorated service still passes the original consumers' expectations.

## Don't

- Don't keep the `override/` file alongside the decorator; only one mechanism should be active.
- Don't decorate a service that does not have a stable interface; it will break on core updates.
- Don't override controllers via `override/controllers/admin/`; use route-level replacement or event listeners.

## Related skills

- `module-add-override` - when override is truly the last resort.
- `module-add-service` - registering the decorator service.
- `migrate-hook-exec-to-dispatcher` - hooks as an alternative to overrides.
