---
name: module-add-symfony-route
description: Declare a Symfony route in a PrestaShop 9 module's config/routes.yml. Use when the user wants to expose a back-office or AJAX endpoint served by a modern controller.
---

## Requirements

Ask the user:
* Route name in `snake_case`, prefixed with the module name (e.g. `mymodule_settings_save`). The prefix avoids collisions with core and other modules.
* HTTP methods accepted (`[GET]`, `[POST]`, `[GET, POST]`).
* Controller FQCN and action method (`MyVendor\Mymodule\Controller\Admin\SettingsController::saveAction`).
* Path under `/modules/<modulename>/...` so URLs are predictable.
* Whether the route should be exposed to JS via FOSJsRouting (`options: { expose: true }`).
* Permission expression: `read`, `create`, `update`, `delete` checked against the legacy controller name.

## Steps

1. Edit `config/routes.yml` at the module root. PrestaShop discovers and merges this file into the kernel router for every enabled module - no manual import needed.

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

    mymodule_settings_save:
      path: /modules/mymodule/settings
      methods: [POST]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\SettingsController::saveAction'
        _legacy_controller: 'AdminMymoduleSettings'
      options:
        expose: true
    ```

2. Always pair every route with `_legacy_controller` (and `_legacy_link` for index actions). The BO menu, breadcrumb and permission system are still keyed on legacy controller names; without it the user cannot be authorized.

3. Generate URLs:

    * Twig (preferred): `{{ path('mymodule_settings_save') }}` and `{{ url('mymodule_settings_save') }}`.
    * PHP inside a controller: `$this->generateUrl('mymodule_settings_save', ['id' => 42])`.
    * JavaScript: only if `options: { expose: true }`, then `Routing.generate('mymodule_settings_save')`.

4. Routes with parameters use Symfony's standard syntax with optional requirements:

    ```yaml
    mymodule_item_edit:
      path: /modules/mymodule/items/{itemId}
      methods: [GET]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\ItemController::editAction'
        _legacy_controller: 'AdminMymoduleItems'
      requirements:
        itemId: '\d+'
    ```

5. After editing `config/routes.yml`, clear the Symfony cache: `php bin/console cache:clear` (run from the PrestaShop root). Routes are otherwise stale.

## Do

- Prefix every route name with the module's technical name to avoid collisions.
- Keep paths under `/modules/<modulename>/` for discoverability in route lists (`bin/console debug:router | grep mymodule`).
- Use `_legacy_controller` so the admin permission system and menu can authorize the request.
- Use `methods:` to constrain verbs; declare a separate route per method when actions diverge.

## Don't

- Don't hardcode URLs as `index.php?controller=AdminFoo&action=bar` in PHP, Twig or JS. Always use `path()` / `generateUrl()` / `Routing.generate()` so refactors are safe.
- Don't import the routes file from anywhere - PrestaShop loads it automatically. Manually importing it produces duplicate-route errors.
- Don't omit `_legacy_controller`. Routes without it cannot be permission-checked and break the BO menu.
- Don't reuse a route name that already exists in core or another module; `cache:clear` will fail with a duplicate-route error.

## Canonical examples

- [devdocs - Admin controllers](https://devdocs.prestashop-project.org/9/modules/concepts/controllers/admin-controllers/).
- [devdocs - Migration guide: controller and routing](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/controller-routing/).
- [devdocs - Modern controller and routing](https://devdocs.prestashop-project.org/9/development/architecture/modern/controller-routing/).
- Working sample: [`example-modules/demomoduleroutes`](https://github.com/PrestaShop/example-modules/tree/master/demomoduleroutes).
