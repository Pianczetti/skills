---
name: module-add-widget
description: Implement PrestaShop\PrestaShop\Core\Module\WidgetInterface in a PrestaShop 9 module so it can render reusable, cache-friendly content from any display hook. Use when the user wants the module to expose a block (sidebar, header, footer, product page extra content, etc.).
---

## Requirements

Ask the user:
* The display hook(s) the widget will be attached to (e.g. `displayHome`, `displayHeader`, `displayLeftColumn`, `displayProductExtraContent`).
* The data the widget needs to render (configuration keys, CMS pages, products, languages, ...).
* Whether the widget is shared across multiple shops (multistore-aware queries) or scoped to the current shop only.

## Steps

1. Make the main module class implement `PrestaShop\PrestaShop\Core\Module\WidgetInterface`. The interface declares two untyped methods, so the implementation must match its signature:

    ```php
    use PrestaShop\PrestaShop\Core\Module\WidgetInterface;

    class Mymodule extends Module implements WidgetInterface
    {
        public const HOOKS = ['displayHome', 'displayLeftColumn'];

        public function install(): bool
        {
            return parent::install() && $this->registerHook(self::HOOKS);
        }
    }
    ```

2. Implement `getWidgetVariables($hookName, array $configuration)` to return ONLY the data the template needs. Pull configuration through `$configuration` (passed by hook callers) and through `Configuration::get(...)` with the current shop constraint - never read `$_GET`/`$_POST` here:

    ```php
    public function getWidgetVariables($hookName = null, array $configuration = [])
    {
        return [
            'title' => Configuration::get('MYMODULE_TITLE'),
            'items' => $this->get(MyVendor\Mymodule\Service\ItemProvider::class)
                ->forShop((int) $this->context->shop->id),
            'hook' => $hookName,
        ];
    }
    ```

3. Implement `renderWidget($hookName, array $configuration)`. Assign the variables to Smarty and fetch the template under `views/templates/hook/`:

    ```php
    public function renderWidget($hookName = null, array $configuration = [])
    {
        $this->smarty->assign($this->getWidgetVariables($hookName, $configuration));

        return $this->fetch('module:' . $this->name . '/views/templates/hook/widget.tpl');
    }
    ```

4. Create the Smarty template `views/templates/hook/widget.tpl`. Keep it presentational: no DB calls, no `Configuration::get`, no raw SQL:

    ```smarty
    <div class="mymodule-widget" data-hook="{$hook|escape:'html':'UTF-8'}">
      <h3>{$title|escape:'html':'UTF-8'}</h3>
      <ul>
        {foreach from=$items item=item}
          <li><a href="{$item.url|escape:'html':'UTF-8'}">{$item.label|escape:'html':'UTF-8'}</a></li>
        {/foreach}
      </ul>
    </div>
    ```

5. PrestaShop wraps every widget call with the Smarty template cache. To force a refresh on configuration change, call `$this->_clearCache('widget.tpl')` from the FormDataProvider's `setData()` or from any `Configuration::updateValue` site.

6. To render the widget anywhere outside its registered hooks (e.g. another module, a CMS page), inject `PrestaShop\PrestaShop\Core\Hook\HookDispatcherInterface` and call `dispatchRenderingWithParameters('displayHome', ['idModule' => $idModule])`. The dispatcher returns a `RenderedHookInterface` whose `getContent()` is the rendered HTML. See `module-register-hooks`.

## Do

- Implement BOTH `getWidgetVariables` and `renderWidget`. Some core consumers call only `getWidgetVariables` (JSON APIs, alternative renderers).
- Keep widgets stateless. Pass everything the template needs through `$configuration` or read-only services. Do not store request-scoped data on the module instance.
- Match the interface signature exactly (`($hookName, array $configuration)`, no return types). Adding `?string`/`: string` breaks the contract on PHP < 8 strict mode.
- Register every display hook in `install()` and the matching listener method (`hookDisplayHome`, `hookDisplayLeftColumn`) on the module class. The listener typically forwards to `renderWidget`.
- Use `module:<modulename>/views/templates/hook/<file>.tpl` paths so PS resolves the override directory `themes/<theme>/modules/<modulename>/...` automatically.

## Don't

- Don't implement `WidgetInterface` from a service; PrestaShop only inspects the main module class.
- Don't put business logic in `renderWidget`. Build the data in a service, return a flat array from `getWidgetVariables`, and let the template stay declarative.
- Don't use `Hook::exec()` to dispatch your own renderable hooks from new code. Inject `HookDispatcherInterface` (see `module-register-hooks`).
- Don't read `Tools::getValue(...)` from a widget. Widgets render in many contexts (incl. CLI, AJAX includes, mail previews) where the request is not what you expect.

## Canonical examples

- [devdocs - Widgets](https://devdocs.prestashop-project.org/9/modules/concepts/widgets/).
- [`PrestaShop/PrestaShop` - WidgetInterface](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Module/WidgetInterface.php).
- [`PrestaShop/ps_languageselector`](https://github.com/PrestaShop/ps_languageselector) (front-office language picker, full WidgetInterface implementation).
- [`PrestaShop/ps_emailsubscription`](https://github.com/PrestaShop/ps_emailsubscription) (newsletter widget with form handling).
