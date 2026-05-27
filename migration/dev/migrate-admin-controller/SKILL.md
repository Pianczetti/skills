---
name: migrate-admin-controller
description: Convert a legacy AdminXxxController (extending ModuleAdminController) to a modern Symfony controller extending PrestaShopAdminController with routing, security attributes, and Twig rendering. Use when a module ships a legacy admin controller that must move to the PS 9 Symfony stack.
---

## Requirements

- Module name and vendor namespace.
- Legacy controller class name (e.g. `AdminMymoduleStuffController`).
- New route name and path (e.g. `mymodule_stuff_index`, `/modules/mymodule/stuff`).
- Actions the controller exposes (index, create, edit, delete, toggle).
- Whether the controller renders a Grid, a Form, or free-form Twig content.

## Steps

1. Create the modern controller under `src/Controller/Admin/<Name>Controller.php` extending `PrestaShopBundle\Controller\Admin\PrestaShopAdminController`. Add `#[IsGranted('read', subject: '_legacy_controller')]` on the index action:

    ```php
    namespace MyVendor\Mymodule\Controller\Admin;

    use PrestaShopBundle\Controller\Admin\PrestaShopAdminController;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Security\Http\Attribute\IsGranted;

    class StuffController extends PrestaShopAdminController
    {
        #[IsGranted('read', subject: '_legacy_controller')]
        public function indexAction(Request $request): Response
        {
            return $this->render('@Modules/mymodule/views/templates/admin/stuff/index.html.twig');
        }
    }
    ```

2. Declare routes in `config/routes.yml` with `_legacy_controller` and `_legacy_link` defaults so the permission system and breadcrumb still resolve:

    ```yaml
    mymodule_stuff_index:
      path: /modules/mymodule/stuff
      methods: [GET]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\StuffController::indexAction'
        _legacy_controller: 'AdminMymoduleStuff'
        _legacy_link: 'AdminMymoduleStuff'
    ```

3. Create the Twig template `views/templates/admin/stuff/index.html.twig` extending `@PrestaShop/Admin/layout.html.twig`. Migrate the legacy `renderList()` / `renderForm()` output into Twig blocks, using the Grid macro or Symfony Form helpers as needed.

4. Register the Tab via the module's `getTabs()` method so the BO menu entry points at the new route:

    ```php
    public function getTabs(): array
    {
        return [
            [
                'class_name' => 'AdminMymoduleStuff',
                'route_name' => 'mymodule_stuff_index',
                'visible' => true,
                'name' => 'Stuff',
                'parent_class_name' => 'IMPROVE',
                'wording' => 'Stuff',
                'wording_domain' => 'Modules.Mymodule.Admin',
            ],
        ];
    }
    ```

5. Delete the legacy file `controllers/admin/AdminMymoduleStuffController.php`.

6. Search for any remaining `index.php?controller=AdminMymoduleStuff` or `$this->context->link->getAdminLink('AdminMymoduleStuff')` references and replace them with `$this->generateUrl('mymodule_stuff_index')` (PHP) or `{{ path('mymodule_stuff_index') }}` (Twig).

7. Run `bin/console prestashop:module uninstall mymodule && bin/console prestashop:module install mymodule` to verify Tab registration and route compilation.

## Do

- Keep `_legacy_controller` identical to the old controller class name so existing employee permissions carry over without reconfiguration.
- Use `#[IsGranted]` attributes per action (`read`, `create`, `update`, `delete`).

## Don't

- Don't leave the old `controllers/admin/` file in place; two controllers responding to the same Tab class name causes routing conflicts.
- Don't render Smarty `.tpl` from a modern controller; use Twig exclusively.
- Don't omit `_legacy_link` from routes; the legacy link generator needs it for backward-compatible URL generation.

## Related skills

- `module-add-admin-controller-modern` - the greenfield version of this migration.
- `migrate-legacy-links` - systematic replacement of hardcoded legacy URLs.
- `migrate-legacy-tabs-to-routes` - Tab registration via `getTabs()`.
