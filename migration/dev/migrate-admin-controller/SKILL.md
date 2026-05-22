---
name: migrate-admin-controller
description: Convert a legacy AdminXxxController (extends ModuleAdminController) to a modern Symfony controller extending PrestaShopAdminController with routing, security attributes, and Twig rendering. Use when a module has a Back Office page served by an AdminController that must be migrated to the PS 9 architecture.
---

## Requirements

- Identify the legacy controller class (e.g. `AdminMymoduleVouchersController extends ModuleAdminController`).
- Map every action: `initContent()`, `renderList()`, `renderForm()`, `postProcess()` to their modern equivalents (index action, form submit action, AJAX endpoints).
- Confirm the target route prefix (`/modules/<modulename>/...`) and the `_legacy_controller` name that preserves existing permissions.

## Steps

1. **Remove the legacy controller file** (`controllers/admin/AdminMymoduleVouchersController.php`). Before deleting, extract:
   - Template assignments from `initContent()` / `renderList()` / `renderForm()`.
   - `postProcess()` logic (form handling, CRUD).
   - `$this->fields_list` / `$this->fields_form` definitions (migrate to Grid / Symfony Form separately).

2. **Create the modern controller** under `src/Controller/Admin/<Name>Controller.php`:

    ```php
    <?php
    namespace MyVendor\Mymodule\Controller\Admin;

    use PrestaShopBundle\Controller\Admin\PrestaShopAdminController;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;
    use Symfony\Component\Security\Http\Attribute\IsGranted;

    class VoucherController extends PrestaShopAdminController
    {
        #[IsGranted('read', subject: '_legacy_controller')]
        public function indexAction(Request $request): Response
        {
            return $this->render('@Modules/mymodule/views/templates/admin/voucher/index.html.twig');
        }

        #[IsGranted('create', subject: '_legacy_controller')]
        public function createAction(Request $request): Response
        {
            // Form handling - see migrate-helperform-to-symfony-form
        }
    }
    ```

3. **Declare routes** in `config/routes.yml`:

    ```yaml
    mymodule_voucher_index:
      path: /modules/mymodule/vouchers
      methods: [GET]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\VoucherController::indexAction'
        _legacy_controller: 'AdminMymoduleVouchers'
        _legacy_link: 'AdminMymoduleVouchers'
    ```

4. **Create the Twig template** replacing the Smarty `.tpl` or the `renderList()` output:

    ```twig
    {% extends '@PrestaShop/Admin/layout.html.twig' %}
    {% block content %}
      {# Grid, form, or custom content here #}
    {% endblock %}
    ```

5. **Migrate permission checks.** Replace `$this->access('edit')` calls with `#[IsGranted('update', subject: '_legacy_controller')]` on each action.

6. **Update Tab registration.** The `install()` Tab entry must point to the same `_legacy_controller` value so the sidebar link and breadcrumb continue to work. No changes needed if the legacy controller name is preserved.

7. **Migrate `postProcess()` logic** to dedicated actions (one per HTTP verb/route) or delegate to CQRS commands (see `migrate-objectmodel-to-cqrs`).

## Do

- Keep the same `_legacy_controller` name so existing role/permission assignments are preserved.
- Use `#[IsGranted]` attributes for per-action security instead of manual `if (!$this->access(...))` checks.
- Inject services via constructor or action parameters; never use `$this->context->controller`.

## Don't

- Don't keep both the legacy controller file and the new Symfony controller registered simultaneously; they will conflict on the Tab route.
- Don't render Smarty `.tpl` files from a modern controller; always use Twig.
- Don't skip `_legacy_link` in the route defaults; the BO menu system needs it to highlight the active tab.

## Canonical examples

- [devdocs - Migration guide: controller and routing](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/controller-routing/)
- [devdocs - Admin controllers](https://devdocs.prestashop-project.org/9/modules/concepts/controllers/admin-controllers/)

## Related skills

- `module-add-admin-controller-modern` - full recipe for the modern controller pattern.
- `migrate-helperform-to-symfony-form` - migrating the form rendered by the legacy controller.
- `migrate-helperlist-to-grid` - migrating the list rendered by the legacy controller.
