---
name: migrate-override-to-decorator
description: Replace `override/classes/` or `override/controllers/` files with a Symfony service decorator (`#[AsDecorator]`) or event listener. Use when a module ships overrides that cause merge conflicts and must be replaced with a modern extension point.
---

## Requirements

- Identify override files shipped by the module (`override/classes/*.php`, `override/controllers/**/*.php`).
- For each override, determine the behaviour change (method interception, value mutation, extra side-effect).
- Confirm PrestaShop version >= 9 where the target class is available as a Symfony service.

## Decision tree

| Situation | Approach |
|-----------|----------|
| Core exposes a hook at the call site | Register a hook listener (`module-register-hooks`) |
| Core class is a Symfony service with an interface | Use `#[AsDecorator]` wrapping the interface |
| Core dispatches a domain event (CQRS) | Write an event subscriber |
| No hook, no service, no event | Keep the override (last resort) and document collision risk |

## Steps

1. **Identify the legacy override:**

    ```php
    // override/classes/Cart.php
    class Cart extends CartCore
    {
        public function getOrderTotal($with_taxes = true, $type = Cart::BOTH, ...)
        {
            $total = parent::getOrderTotal($with_taxes, $type, ...);
            return (float) (round($total * 20) / 20); // round to 0.05
        }
    }
    ```

2. **Check if the core class is a registered Symfony service:**

    ```bash
    bin/console debug:container | grep -i cart
    # Look for a service implementing an interface you can decorate
    ```

3. **Option A - Service Decorator** (preferred when an interface exists):

    ```php
    <?php
    namespace MyVendor\Mymodule\Decorator;

    use PrestaShop\PrestaShop\Core\Cart\CartTotalCalculatorInterface;
    use Symfony\Component\DependencyInjection\Attribute\AsDecorator;

    #[AsDecorator(decorates: CartTotalCalculatorInterface::class)]
    final class RoundedCartTotalCalculator implements CartTotalCalculatorInterface
    {
        public function __construct(
            private readonly CartTotalCalculatorInterface $inner,
        ) {}

        public function getTotal(CartContext $context): float
        {
            $total = $this->inner->getTotal($context);
            return (float) (round($total * 20) / 20);
        }
    }
    ```

    Register in `config/services.yml` (autowire + autoconfigure handles the rest):
    ```yaml
    services:
      MyVendor\Mymodule\Decorator\RoundedCartTotalCalculator:
        autowire: true
        autoconfigure: true
    ```

4. **Option B - Event Listener** (when a hook or domain event fires):

    ```php
    <?php
    namespace MyVendor\Mymodule\EventListener;

    use PrestaShop\PrestaShop\Core\Hook\HookDispatcherInterface;
    use Symfony\Component\EventDispatcher\Attribute\AsEventListener;

    #[AsEventListener(event: 'actionCartGetOrderTotal')]
    final class RoundCartTotalListener
    {
        public function __invoke(array $params): void
        {
            $params['total'] = (float) (round($params['total'] * 20) / 20);
        }
    }
    ```

5. **Remove the override file** from `override/classes/Cart.php` and update `install()` / `uninstall()` so the module no longer copies/removes it.

6. **Clear cache and test:**

    ```bash
    bin/console cache:clear
    rm -f var/cache/*/class_index.php
    ```

## Do

- Prefer `#[AsDecorator]` when the core class exposes a clear interface; it is compile-time safe.
- Always delegate to the inner service for unmodified behaviour; never silently swallow parent logic.
- Document why you chose decorator vs listener vs hook in a code comment.

## Don't

- Don't decorate a class that has no interface; you will couple to the concrete implementation.
- Don't leave orphan override files in the module after migration; they will reinstall on next `install()`.
- Don't use a decorator for cross-cutting concerns (logging, caching) on many services; use a Compiler Pass or event subscriber instead.

## Canonical examples

- [devdocs - Overrides](https://devdocs.prestashop-project.org/9/modules/concepts/overrides/) - explains why overrides are last resort.
- [Symfony - Service decoration](https://symfony.com/doc/current/service_container/service_decoration.html)

## Related skills

- `module-add-override` - the override mechanism itself (last resort).
- `module-register-hooks` - hook-based alternative.
- `migrate-hook-exec-to-dispatcher` - modern hook dispatching.
