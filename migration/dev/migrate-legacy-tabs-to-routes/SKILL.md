---
name: migrate-legacy-tabs-to-routes
description: Replace `Tab::installControllerClass()` / manual Tab object creation for admin menu entries with Symfony route-based tab registration using `_legacy_controller` in `config/routes.yml`. Use when a module registers admin tabs via legacy Tab API and must adopt the modern routing-based menu system.
---

## Requirements

- Identify legacy tab installation code in `install()` / `uninstall()` methods.
- Confirm the parent tab class name (e.g. `AdminParentOrders`, `SELL`).
- Map each tab to a Symfony route with `_legacy_controller` and `_legacy_link`.

## Steps

1. **Identify the legacy pattern:**

    ```php
    // Legacy install() - manual Tab creation
    public function install()
    {
        $tab = new Tab();
        $tab->class_name = 'AdminMymoduleVouchers';
        $tab->module = $this->name;
        $tab->id_parent = (int) Tab::getIdFromClassName('AdminParentOrders');
        $tab->active = 1;
        foreach (Language::getLanguages(true) as $lang) {
            $tab->name[$lang['id_lang']] = 'Vouchers';
        }
        $tab->add();

        return parent::install();
    }
    ```

2. **Declare the route** in `config/routes.yml` with `_legacy_controller`:

    ```yaml
    mymodule_voucher_index:
      path: /modules/mymodule/vouchers
      methods: [GET]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\VoucherController::indexAction'
        _legacy_controller: 'AdminMymoduleVouchers'
        _legacy_link: 'AdminMymoduleVouchers'
    ```

3. **Modernize the tab registration** in `install()`. Use the `$tabs` property (PS 8+) for declarative registration:

    ```php
    class Mymodule extends Module
    {
        public function getTabs(): array
        {
            return [
                [
                    'class_name' => 'AdminMymoduleVouchers',
                    'route_name' => 'mymodule_voucher_index',
                    'name' => 'Vouchers',          // or multilang array
                    'parent_class_name' => 'AdminParentOrders',
                    'wording' => 'Vouchers',
                    'wording_domain' => 'Modules.Mymodule.Admin',
                ],
            ];
        }
    }
    ```

    PrestaShop calls `getTabs()` during install and auto-creates the Tab entries, linking them to the route.

4. **Remove manual Tab creation/deletion** from `install()` and `uninstall()`. The framework handles both when `getTabs()` is declared.

5. **Preserve permissions.** The `class_name` in `getTabs()` must match `_legacy_controller` in the route so the existing role assignments (`ROLE_MOD_TAB_ADMINMYMODULEVOUCHERS_READ`, etc.) continue to work.

6. **Verify:**

    ```bash
    bin/console cache:clear
    bin/console debug:router | grep mymodule
    ```

    Confirm the sidebar shows the tab and clicking it loads the Symfony controller.

## Do

- Use `getTabs()` (declarative) instead of manual Tab object creation.
- Keep `class_name` identical to the `_legacy_controller` route default for permission continuity.
- Set `wording` and `wording_domain` for translatable menu labels.

## Don't

- Don't create Tab objects manually in `install()` when `getTabs()` is available (PS 8+).
- Don't use a different `class_name` in `getTabs()` than in the route; permissions will break.
- Don't forget `uninstall()` cleanup if you mix manual and declarative approaches.

## Canonical examples

- [devdocs - Admin controllers: tabs](https://devdocs.prestashop-project.org/9/modules/concepts/controllers/admin-controllers/#tab-registration)
- [devdocs - Migration guide: controller and routing](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/controller-routing/)

## Related skills

- `module-add-symfony-route` - declaring the Symfony route.
- `module-add-admin-controller-modern` - the controller the tab points to.
- `migrate-admin-controller` - full controller migration workflow.
