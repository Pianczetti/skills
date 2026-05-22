---
name: migrate-hook-exec-to-dispatcher
description: Replace all Hook::exec() static calls with constructor-injected HookDispatcherInterface and dispatchWithParameters(). Use when a module dispatches hooks via the legacy static helper and needs to adopt the modern testable dispatcher.
---

## Requirements

- Module name and vendor namespace.
- List of `Hook::exec()` call sites (file and line).
- Hook names being dispatched and their parameter arrays.

## Steps

1. Grep the module for all `Hook::exec(` calls:

    ```bash
    grep -rn 'Hook::exec(' modules/mymodule/
    ```

2. For each service or class that dispatches hooks, inject `PrestaShop\PrestaShop\Core\Hook\HookDispatcherInterface` via the constructor:

    ```php
    namespace MyVendor\Mymodule\Service;

    use PrestaShop\PrestaShop\Core\Hook\HookDispatcherInterface;

    class OrderExporter
    {
        public function __construct(
            private readonly HookDispatcherInterface $hookDispatcher,
        ) {}
    }
    ```

3. Replace each `Hook::exec('hookName', $params)` with `$this->hookDispatcher->dispatchWithParameters('hookName', $params)`:

    ```php
    // Before
    Hook::exec('actionMymoduleExport', ['order_id' => $orderId]);

    // After
    $this->hookDispatcher->dispatchWithParameters('actionMymoduleExport', ['order_id' => $orderId]);
    ```

4. If the hook call lives in the module's main class (which is not a DI-managed service), inject the dispatcher via the Symfony container in a lazy fashion:

    ```php
    private function getHookDispatcher(): HookDispatcherInterface
    {
        return $this->get('prestashop.core.hook.dispatcher');
    }
    ```

5. Register the service in `config/services.yml` if not already autowired:

    ```yaml
    services:
      MyVendor\Mymodule\Service\OrderExporter:
        autowire: true
    ```

6. For hooks that collect return values (`Hook::exec(..., null, null, true)`), use `dispatchWithParameters()` and read from the `HookDispatcherInterface` return. The dispatcher returns the aggregated module responses identically to `Hook::exec()`.

7. Verify by triggering the hook scenario in the BO or front office and confirming listening modules still receive the dispatch.

## Do

- Inject `HookDispatcherInterface` via constructor for testability.
- Keep the hook name string identical to the one registered in `registerHook()`.
- Use the `prestashop.core.hook.dispatcher` service id when fetching from the container manually.

## Don't

- Don't keep `Hook::exec()` calls alongside the new dispatcher; remove them in the same commit.
- Don't rename hooks during migration; that breaks third-party listeners.
- Don't dispatch hooks from inside a CQRS handler; hooks belong at the application/controller layer.

## Related skills

- `module-register-hooks` - registering and listening to hooks the modern way.
- `migrate-objectmodel-to-cqrs` - moving mutation logic out of hook callbacks.
