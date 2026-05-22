---
name: migrate-hook-exec-to-dispatcher
description: Replace all Hook::exec() static calls with HookDispatcherInterface injection and dispatchWithParameters(). Use when a module dispatches hooks via the legacy static helper and must adopt the PS 9 service-based dispatcher.
---

## Requirements

- Grep the module codebase for `Hook::exec(` calls.
- For each call, determine whether the hook is a pure event (fire-and-forget) or renderable (returns HTML).
- Confirm the hook names follow the naming convention (`action*` for events, `display*` for renderable).

## Steps

1. **Identify the legacy pattern:**

    ```php
    // LEGACY - static call, no DI, bypasses Symfony event listeners
    Hook::exec('actionMymoduleAfterImport', ['imported' => $count]);

    // LEGACY - renderable hook
    $html = Hook::exec('displayMymoduleBlock', ['product' => $product], null, true);
    ```

2. **Inject `HookDispatcherInterface`** into the service that dispatches the hook:

    ```php
    use PrestaShop\PrestaShop\Core\Hook\HookDispatcherInterface;

    final class ImportRunner
    {
        public function __construct(
            private readonly HookDispatcherInterface $hookDispatcher,
        ) {}
    }
    ```

3. **Replace event hooks** with `dispatchWithParameters()`:

    ```php
    // AFTER - service-based, testable, compatible with Symfony listeners
    $this->hookDispatcher->dispatchWithParameters(
        'actionMymoduleAfterImport',
        ['imported' => $count]
    );
    ```

4. **Replace renderable hooks** with `dispatchRenderingWithParameters()`:

    ```php
    // AFTER - aggregates HTML returned by all listeners
    $html = $this->hookDispatcher->dispatchRenderingWithParameters(
        'displayMymoduleBlock',
        ['product' => $product]
    );
    ```

5. **Update service registration.** If using autowiring (standard `services.yml`), no extra config is needed. The `HookDispatcherInterface` is already registered as `prestashop.core.hook.dispatcher`. For manual wiring:

    ```yaml
    MyVendor\Mymodule\Service\ImportRunner:
      arguments:
        $hookDispatcher: '@prestashop.core.hook.dispatcher'
    ```

6. **Handle the main module class.** In the main module file (which is not a service), you cannot constructor-inject. Use the container bridge:

    ```php
    // Inside the main module class only
    /** @var HookDispatcherInterface $dispatcher */
    $dispatcher = $this->get('prestashop.core.hook.dispatcher');
    $dispatcher->dispatchWithParameters('actionMymoduleAfterInstall', []);
    ```

7. **Remove all `Hook::exec()` calls.** Search with `grep -rn 'Hook::exec' src/ controllers/` and replace every occurrence.

## Do

- Use `dispatchWithParameters()` for `action*` hooks (no return value aggregated).
- Use `dispatchRenderingWithParameters()` for `display*` hooks (returns concatenated HTML from all listeners).
- Inject `HookDispatcherInterface` via constructor in services; use `$this->get(...)` only in the main module class.

## Don't

- Don't call `Hook::exec()` from new code; it bypasses Symfony-registered listeners and is not mockable in tests.
- Don't pass `null` for the module ID parameter thinking it scopes the hook; the dispatcher handles scoping internally.
- Don't mix static `Hook::exec()` and `HookDispatcherInterface` for the same hook in the same codebase; pick one path.

## Canonical examples

- [devdocs - Hooks](https://devdocs.prestashop-project.org/9/modules/concepts/hooks/)
- [`HookDispatcherInterface`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Hook/HookDispatcherInterface.php)

## Related skills

- `module-register-hooks` - full recipe for registering and dispatching hooks.
- `migrate-admin-controller` - the controller migration that often hosts hook dispatch sites.
