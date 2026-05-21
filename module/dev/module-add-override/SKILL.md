---
name: module-add-override
description: Add a class or controller override under override/ in a PrestaShop 9 module as a LAST-RESORT extension mechanism. Use only when no hook, service decorator, CQRS command or event listener can express the change.
---

> **Overrides are a LAST RESORT.** They lock you to a specific PrestaShop version, conflict with other modules that override the same class (the order modules are installed determines who wins), and break under composer-managed cores where vendor classes are loaded before overrides. Try a HOOK, a SERVICE DECORATOR, a CQRS COMMAND or an EVENT LISTENER first. Only use this skill when none of those is feasible (e.g. the core method has no hook, is `final`, or you cannot intercept the call site).

## Requirements

Ask the user:
* Which Core class or controller they need to override (e.g. `Cart`, `Product`, `AdminOrdersController`, `OrderController`).
* The exact list of methods to override - keep it as small as possible, ideally one.
* Whether they have already considered: a hook (`bin/console prestashop:update:hooks-documentation` lists every dispatch site), a service decorator, or a CQRS command. Document why each was rejected.
* Whether the change applies to legacy classes (`classes/`), legacy controllers (`controllers/admin`, `controllers/front`) or admin tabs.

## Steps

1. Create the override file under the module. PrestaShop's autoloader copies (and merges) `override/` into the shop's own `override/` folder on `install()`, and removes it on `uninstall()`. The folder layout MIRRORS the core layout - one `override/classes/<Class>.php` per overridden class:

    ```
    mymodule/
    └── override/
        ├── classes/
        │   └── Cart.php                       # mirrors classes/Cart.php
        ├── controllers/
        │   ├── admin/AdminOrdersController.php
        │   └── front/OrderController.php
        └── classes/order/Order.php
    ```

2. Each override class is named exactly like the core class and extends its `*Core` counterpart. Override only the methods you need; let everything else fall through:

    ```php
    <?php
    // mymodule/override/classes/Cart.php
    class Cart extends CartCore
    {
        public function getOrderTotal(
            $with_taxes = true,
            $type = Cart::BOTH,
            $products = null,
            $id_carrier = null,
            $use_cache = true
        ) {
            $total = parent::getOrderTotal($with_taxes, $type, $products, $id_carrier, $use_cache);

            // example: round every cart total to the nearest 0.05
            return (float) (round($total * 20) / 20);
        }
    }
    ```

   ALWAYS call `parent::<method>(...)` first unless you mean to fully replace the behaviour - that is the most common cause of regressions.

3. For modern Symfony admin controllers, you CANNOT override via `override/`. The `override/` mechanism only works for legacy `classes/` and `controllers/`. To extend a modern controller you must either:
    - Subscribe to a Symfony event dispatched by the core controller, or
    - Decorate the underlying service via a `CompilerPass`, or
    - Replace the route in `config/routes.yml` with one of yours that calls into the core controller.

4. Document the conflict-resolution surface. The merge happens in install order: if module A overrides `Cart::getOrderTotal()` and module B overrides the same method later, module B's class extends module A's class and module A's body wins as the parent body. If the merge fails (incompatible method signatures, duplicate method declarations) PS aborts the install with a clear error in the log. List every override you ship in the module README so a merchant can detect collisions:

    ```
    ## Overrides shipped by this module
    - classes/Cart.php :: getOrderTotal
    ```

5. Clear the cache after every override change. PrestaShop generates `cache/class_index.php` from the union of core + overrides; stale cache will silently keep loading the previous version:

    ```bash
    bin/console cache:clear
    rm -f var/cache/*/class_index.php
    ```

6. Test the uninstall path. PS removes the merged override on `uninstall()`, but only if no other module has the same override - in which case the file stays merged with the surviving overrides. Reinstall and confirm the file content is correct.

## Do

- Prefer a HOOK first. Run `bin/console prestashop:update:hooks-documentation` to see if a `display*` or `action*` hook already covers the call site.
- Prefer a SERVICE DECORATOR for any modern Symfony service. Implement `Symfony\Component\DependencyInjection\Compiler\CompilerPassInterface` or use `#[AsDecorator]` to wrap the core service without touching its class.
- Prefer a CQRS COMMAND HANDLER for domain mutations. Subscribe via `tactician.handler` or write a domain event listener.
- Keep overrides surgical: one method, one class, with `parent::` call + comment explaining why a hook was not enough.
- Add the override to the module README with the file path and method name so collisions can be diagnosed.

## Don't

- Don't override a class you only need to inject behaviour into. Write a hook listener (`module-register-hooks`) or a service decorator instead.
- Don't override `final` core classes - the merge will fail at install time. Look for the equivalent CQRS command or refactor the call site upstream.
- Don't override modern Symfony controllers via `override/`. The mechanism does not apply; use events or a custom route.
- Don't ship overrides that do nothing but `parent::` - empty overrides only create future merge conflicts with other modules.
- Don't forget `cache:clear` after editing an override; PS caches the merged class map.

## Canonical examples

- [devdocs - Overrides](https://devdocs.prestashop-project.org/9/modules/concepts/overrides/) - the complete contract, including merge order and the install/uninstall lifecycle.
- [`PrestaShop/example-modules` - demooverrideobjectmodel](https://github.com/PrestaShop/example-modules/tree/master/demooverrideobjectmodel) - canonical override of an `ObjectModel` (extends `CartCore`), kept intentionally small.
- For the recommended alternatives the override skill steers users towards:
  - Hooks: [`module-register-hooks`](../module-register-hooks/SKILL.md) and [devdocs - Hooks](https://devdocs.prestashop-project.org/9/modules/concepts/hooks/).
  - Service decorators: [Symfony - Service decoration](https://symfony.com/doc/current/service_container/service_decoration.html).
  - CQRS commands: [`module-add-cqrs-command`](../module-add-cqrs-command/SKILL.md).
