---
name: module-add-front-controller
description: Add a ModuleFrontController to expose a front-office page or AJAX endpoint from a PrestaShop 9 module. Use when the user wants a customer-facing URL served by their module.
---

## Requirements

Ask the user:
* Controller slug in `lower_snake_case` (becomes the file name and the URL `controller=` value), e.g. `display`.
* Whether the page is a full HTML view or an AJAX endpoint returning JSON.
* Whether customer auth is required (`$this->auth = true` redirects guests to login).
* Whether SSL must be forced (`$this->ssl = true`).

## Steps

1. Create `controllers/front/<slug>.php` (the file name is the controller slug, lowercase). The class name is `<ModuleName><Slug>ModuleFrontController` in PascalCase. PrestaShop discovers it by file name.

    ```php
    <?php
    class MymoduleDisplayModuleFrontController extends ModuleFrontController
    {
        public bool $auth = false;
        public bool $ssl = true;

        public function initContent(): void
        {
            parent::initContent();

            // Delegate business logic to a service registered in config/services.yml.
            // Retrieve it from the module container (PS 9 exposes `get()` on the module).
            $payload = $this->module->get('MyVendor\\Mymodule\\Service\\DisplayPayloadProvider')
                ->build($this->context);

            $this->context->smarty->assign(['payload' => $payload]);
            $this->setTemplate('module:mymodule/views/templates/front/display.tpl');
        }
    }
    ```

2. For an AJAX endpoint, set `$this->ajax = true` in the constructor or `init()` and override `displayAjax()` instead of `initContent()`. Return JSON with `$this->ajaxRender(json_encode($data))` and `die`.

    ```php
    public function postProcess(): void { $this->ajax = true; }
    public function displayAjax(): void
    {
        header('Content-Type: application/json');
        $this->ajaxRender(json_encode(['ok' => true]));
    }
    ```

3. Create the Smarty template `views/templates/front/<slug>.tpl`. Extend the theme layout so the page inherits header / footer:

    ```smarty
    {extends file='page.tpl'}
    {block name='page_content'}
      <h1>{l s='Hello' d='Modules.Mymodule.Shop'}</h1>
      <pre>{$payload|escape:'htmlall':'UTF-8'}</pre>
    {/block}
    ```

4. URL patterns the controller is reachable at:

    * Direct: `index.php?fc=module&module=<modulename>&controller=<slug>`.
    * Friendly URL (preferred): generate it with `Link::getModuleLink()`:

    ```php
    $url = $this->context->link->getModuleLink('mymodule', 'display', ['id' => 42], true);
    ```

5. If the controller emits CSS/JS, register them by hooking `actionFrontControllerSetMedia` from the main module class and calling `$this->context->controller->registerStylesheet(...)` / `registerJavascript(...)`. Don't echo `<script>` / `<link>` tags.

## Do

- Delegate business logic to an autowired service in `config/services.yml`. The controller's job is HTTP I/O (read request, set template vars, choose template).
- Use `Modules.<Modulename>.Shop` as the translation domain for shop-facing strings (`{l s='...' d='Modules.Mymodule.Shop'}`).
- Validate every input with `Tools::getValue('foo')` plus a typed cast or `Validate::is*()`; never trust raw `$_GET` / `$_POST`.

## Don't

- Don't put SQL, business rules, or third-party HTTP calls in `initContent()`. Inject a service.
- Don't return HTML strings from `displayAjax()`; return JSON or a structured payload.
- Don't use `$this->l(...)`. Use the Smarty `{l s='...' d='Modules.Mymodule.Shop'}` tag.
- Don't bypass `setTemplate()` by `echo`-ing markup directly: it breaks theme inheritance and child themes.

## PS9 Module Compatibility Notes

### Do NOT type-hint inherited properties

`ModuleFrontController` declares `$ssl`, `$auth`, `$authRedirection`, `$guestAllowed` WITHOUT types. PHP 8 forbids adding types to inherited properties:

```php
// WRONG - Fatal Error
public bool $ssl = true;
public bool $auth = false;

// CORRECT
public $ssl = true;
public $auth = false;
```

### Admin asset paths

In admin Twig templates, module assets need `../` prefix:
```twig
{# WRONG - 404 #}
{{ asset('modules/mymodule/views/css/style.css') }}
{# CORRECT #}
{{ asset('../modules/mymodule/views/css/style.css') }}
```

### AJAX URL building with existing query params

When routes already have `?_token=xxx`, use `URLSearchParams` to append parameters:
```javascript
var url = new URL(dataUrl, window.location.origin);
url.searchParams.set('date', date);
fetch(url.toString());
```

## Canonical examples

- [devdocs - Front controllers](https://devdocs.prestashop-project.org/9/modules/concepts/controllers/front-controllers/).
- [devdocs - Displaying content in front office](https://devdocs.prestashop-project.org/9/modules/creation/displaying-content-in-front-office/).
- [devdocs - Templating concepts](https://devdocs.prestashop-project.org/9/modules/concepts/templating/).
- Working sample: [`example-modules/demoextendtemplates`](https://github.com/PrestaShop/example-modules/tree/master/demoextendtemplates).
