---
name: module-add-admin-controller-modern
description: Add a modern Symfony Back Office controller (PrestaShopAdminController) to a PrestaShop 9 module. Use when the user wants a back-office page, action, or AJAX endpoint served by their module.
---

## Requirements

Ask the user:
* Controller class name in PascalCase ending in `Controller`, e.g. `SettingsController`.
* Route name in `snake_case`, prefixed with the module name, e.g. `mymodule_settings_index`.
* Route path under `/modules/<modulename>/...`, e.g. `/modules/mymodule/settings`.
* HTTP methods (default `GET`, add `POST` for form submission).
* Required permission expression (`read`, `create`, `update`, `delete`) checked against the legacy controller name passed in the request.

## Steps

1. Create the class under `src/Controller/Admin/<Name>Controller.php` extending `PrestaShopBundle\Controller\Admin\PrestaShopAdminController` (NOT the legacy `AdminController`):

    ```php
    <?php
    namespace MyVendor\Mymodule\Controller\Admin;

    use PrestaShopBundle\Controller\Admin\PrestaShopAdminController;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;

    class SettingsController extends PrestaShopAdminController
    {
        public function indexAction(Request $request): Response
        {
            return $this->render(
                '@Modules/mymodule/views/templates/admin/settings/index.html.twig',
                ['title' => 'Mymodule settings']
            );
        }
    }
    ```

2. Register the controller as a service in `config/services.yml` so the container can inject `controller.service_arguments`. With the autowired `resource:` block from `module-create` this is already done; if you registered services manually, add:

    ```yaml
    services:
      MyVendor\Mymodule\Controller\Admin\SettingsController:
        public: true
        tags: ['controller.service_arguments']
    ```

3. Declare the route in `config/routes.yml`. PrestaShop loads this file automatically for every enabled module:

    ```yaml
    mymodule_settings_index:
      path: /modules/mymodule/settings
      methods: [GET]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\SettingsController::indexAction'
        _legacy_controller: 'AdminMymoduleSettings'
        _legacy_link: 'AdminMymoduleSettings'
      options:
        _access_role: 'ROLE_ADMIN'
    ```

   For permission-gated actions, add a `_security` requirement on the route or the `#[IsGranted]` PHP attribute on the action:

    ```php
    use Symfony\Component\Security\Http\Attribute\IsGranted;
    #[IsGranted('read', subject: '_legacy_controller')]
    public function indexAction(Request $request): Response { /* ... */ }
    ```

4. Create the Twig template `views/templates/admin/settings/index.html.twig` extending the BO layout (Twig namespace `@Modules/<modulename>/...` is auto-registered):

    ```twig
    {% extends '@PrestaShop/Admin/layout.html.twig' %}
    {% block content %}
      <h1>{{ 'Settings'|trans({}, 'Modules.Mymodule.Admin') }}</h1>
    {% endblock %}
    ```

5. Add a Back Office menu entry by registering a Tab in `install()` (use `Tab::addTab(...)` or the modern `TabRepository`) pointing at the legacy controller name (`AdminMymoduleSettings` above) so the breadcrumb and permission system work.

6. Generate the URL in Twig with `{{ path('mymodule_settings_index') }}` or in PHP with `$this->generateUrl('mymodule_settings_index')`.

## Do

- Extend `PrestaShopAdminController`. It exposes BO-aware helpers (`generateUrl`, `addFlash`, CSRF tokens, multi-shop context).
- Use Twig (`.html.twig`) for BO templates and the `@Modules/<modulename>/...` namespace.
- Check permissions with `#[IsGranted]` or `_security` route options. Always pair the route with a `_legacy_controller` name so the legacy permission system can authorize requests.
- Inject services through the constructor; the `controller.service_arguments` tag wires action arguments.

## Don't

- Don't `extends AdminController` for new code. That is the legacy path and is excluded from PS 9 conventions; treat any `extends AdminController` you see as legacy code to migrate.
- Don't render with Smarty (`.tpl`) from a modern admin controller; use Twig.
- Don't hardcode `index.php?controller=...` URLs in templates. Use `path()` with the route name.
- Don't put DB queries or third-party API calls in actions; delegate to a service.

## Canonical examples

- [devdocs - Admin controllers](https://devdocs.prestashop-project.org/9/modules/concepts/controllers/admin-controllers/).
- [devdocs - Modern controller and routing](https://devdocs.prestashop-project.org/9/development/architecture/modern/controller-routing/).
- [devdocs - Migration guide: controller and routing](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/controller-routing/).
- Working sample with module routes + admin controllers: [`example-modules/demomoduleroutes`](https://github.com/PrestaShop/example-modules/tree/master/demomoduleroutes).
