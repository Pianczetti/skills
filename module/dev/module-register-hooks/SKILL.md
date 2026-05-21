---
name: module-register-hooks
description: Register hook listeners and dispatch custom hooks from a PrestaShop 9 module the modern way, using HookDispatcherInterface instead of the legacy Hook::exec(). Use when the user wants the module to react to core events or expose its own extension points.
---

## Requirements

Ask the user:
* The list of core hook names the module subscribes to (e.g. `actionFrontControllerSetMedia`, `displayHeader`, `displayProductExtraContent`, `actionObjectOrderAddAfter`). Use the canonical core list (`bin/console prestashop:update:hooks-documentation` writes it under `docs/hooks/`).
* Whether the module also dispatches its own hooks. If yes, capture the custom hook name (camelCase, prefixed with `action`/`display` and the module name, e.g. `actionMymoduleAfterImport`).
* Whether any custom hook is renderable (returns HTML to be aggregated) or a pure event.

## Steps

1. List ALL hook names in a single property of the main module class so install/uninstall iterate the same source of truth:

    ```php
    public const HOOKS = [
        'actionFrontControllerSetMedia',
        'displayHeader',
        'displayProductExtraContent',
        'actionObjectOrderAddAfter',
    ];

    public function install(): bool
    {
        return parent::install() && $this->registerHook(self::HOOKS);
    }

    public function uninstall(): bool
    {
        return parent::uninstall() && $this->unregisterHook(self::HOOKS);
    }
    ```

   `Module::registerHook()` accepts an array since PS 1.7, so a single call replaces a loop.

2. Implement each listener as a public method named `hook<HookName>(array $params)` on the module class. Keep the listener thin and delegate work to a service:

    ```php
    public function hookActionFrontControllerSetMedia(array $params): void
    {
        $this->context->controller->registerStylesheet(
            'mymodule-front',
            'modules/' . $this->name . '/views/css/front.css',
            ['priority' => 200]
        );
    }

    public function hookDisplayProductExtraContent(array $params): array
    {
        return $this->get(MyVendor\Mymodule\Service\ExtraContentRenderer::class)->render($params);
    }
    ```

3. To dispatch a custom hook from your own code, inject `PrestaShop\PrestaShop\Core\Hook\HookDispatcherInterface` (service ID `prestashop.core.hook.dispatcher`). Never call `Hook::exec()` from new code:

    ```php
    use PrestaShop\PrestaShop\Core\Hook\HookDispatcherInterface;

    final class ImportRunner
    {
        public function __construct(private readonly HookDispatcherInterface $hookDispatcher) {}

        public function run(array $rows): void
        {
            // Pure event, listeners cannot return rendered output.
            $this->hookDispatcher->dispatchWithParameters('actionMymoduleAfterImport', [
                'imported' => count($rows),
            ]);

            // Renderable hook, aggregates listener HTML.
            $rendered = $this->hookDispatcher->dispatchRenderingWithParameters(
                'displayMymoduleSummary',
                ['rows' => $rows]
            );
        }
    }
    ```

4. Document the new hook so other developers and the AI tooling can discover it. From the PrestaShop project root:

    ```bash
    bin/console prestashop:update:hooks-documentation
    bin/console prestashop:list:commands-and-queries  # for CQRS context discovery
    ```

   `prestashop:update:hooks-documentation` extracts every hook call into `docs/hooks/`, including the ones declared by enabled modules.

5. For the front-office, prefer the dedicated `display*` hooks tied to a Smarty template (see `module-add-widget`). For Back Office grids and forms, use the dynamic `action<GridId>GridDefinitionModifier` / `actionBuildMailLayoutVariables` hooks instead of editing core templates.

## Do

- Maintain the hook list in ONE place (`self::HOOKS`) so install, uninstall, and developer overviews stay in sync.
- Inject `HookDispatcherInterface` for every dispatch site. Constructor injection only, no `Context::getContext()->container->get(...)`.
- Use `dispatchRenderingWithParameters()` when listeners must return HTML; use `dispatchWithParameters()` for fire-and-forget events.
- Name custom hooks with the `action`/`display` prefix and your module name (e.g. `actionMymoduleSync`, `displayMymoduleBlock`) to avoid collisions.

## Don't

- Don't call `Hook::exec('myHook', $params)` in new code. The legacy static helper is incompatible with the modern dispatcher and bypasses event listeners registered via Symfony tags.
- Don't register hooks dynamically from runtime code (controllers, services, hook listeners). Registration must happen in `install()` so the database state matches the source.
- Don't make listeners do heavy work synchronously. Delegate to a service and dispatch a queued job for slow operations.
- Don't return a `Response` from a `display*` hook listener. Return a string (rendered Smarty/Twig) or an array of stringable values.

## Canonical examples

- [devdocs - Hooks (concept)](https://devdocs.prestashop-project.org/9/modules/concepts/hooks/).
- [devdocs - Migration guide: hooks](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/hooks/).
- [`PrestaShop/PrestaShop` - HookDispatcherInterface](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Hook/HookDispatcherInterface.php).
- Working sample: [`example-modules/demovieworderhooks`](https://github.com/PrestaShop/example-modules/tree/master/demovieworderhooks) (multiple BO display/action hooks on the order view).
- Working sample: [`example-modules/demoproductextracontent`](https://github.com/PrestaShop/example-modules/tree/master/demoproductextracontent) (front-office `displayProductExtraContent`).
