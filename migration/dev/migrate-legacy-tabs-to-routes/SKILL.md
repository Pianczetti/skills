---
name: migrate-legacy-tabs-to-routes
description: Replace manual Tab::installControllerClass() or raw INSERT INTO ps_tab with the modern getTabs() method returning route-aware tab definitions. Use when a module registers Back Office menu entries through legacy tab installation and needs route-based navigation.
---

## Requirements

- Module name.
- Legacy tab installation code (location in `install()` method or SQL file).
- New Symfony route names for each tab.
- Parent menu entry (`IMPROVE`, `SELL`, `CONFIGURE`, or a specific parent class name).

## Steps

1. Locate the legacy tab installation in the module's `install()` method:

    ```bash
    grep -n 'Tab::' modules/mymodule/mymodule.php
    grep -n 'installControllerClass\|ps_tab' modules/mymodule/
    ```

2. Remove the legacy tab creation code from `install()` and `uninstall()`:

    ```php
    // Remove this pattern:
    $tab = new Tab();
    $tab->class_name = 'AdminMymoduleStuff';
    $tab->id_parent = Tab::getIdFromClassName('IMPROVE');
    $tab->module = 'mymodule';
    $tab->name = array_fill_keys(Language::getIDs(), 'Stuff');
    $tab->add();
    ```

3. Implement `getTabs()` in the module main class. PrestaShop automatically installs and uninstalls tabs declared here:

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
            [
                'class_name' => 'AdminMymoduleSettings',
                'route_name' => 'mymodule_settings_index',
                'visible' => true,
                'name' => 'Settings',
                'parent_class_name' => 'AdminMymoduleStuff',
                'wording' => 'Settings',
                'wording_domain' => 'Modules.Mymodule.Admin',
            ],
        ];
    }
    ```

4. Ensure each `route_name` is declared in `config/routes.yml` with a matching `_legacy_controller` default equal to `class_name`:

    ```yaml
    mymodule_stuff_index:
      path: /modules/mymodule/stuff
      methods: [GET]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\StuffController::indexAction'
        _legacy_controller: 'AdminMymoduleStuff'
        _legacy_link: 'AdminMymoduleStuff'
    ```

5. For hidden tabs (e.g. child actions like edit/create), set `'visible' => false` but still declare them so the permission system registers the class_name.

6. Remove any manual `Tab::getIdFromClassName()` / `$tab->delete()` from `uninstall()`. The framework handles cleanup when `getTabs()` is used.

7. Reinstall the module to verify tabs appear in the BO menu:

    ```bash
    bin/console prestashop:module uninstall mymodule
    bin/console prestashop:module install mymodule
    ```

8. Verify the breadcrumb and sidebar highlight correctly point to the active tab by navigating each route.

## Do

- Use `wording` + `wording_domain` for translatable tab names.
- Match `class_name` to `_legacy_controller` in routes for permission continuity.
- Nest child tabs by setting `parent_class_name` to the parent tab's `class_name`.

## Don't

- Don't mix `getTabs()` with manual `Tab::` calls; use one approach exclusively.
- Don't create tabs without a corresponding route; orphan tabs lead to 404 errors.
- Don't omit `wording_domain`; without it the tab name is not extractable for translation.

## Related skills

- `migrate-admin-controller` - the controller each tab points to.
- `module-add-admin-controller-modern` - greenfield controller + tab creation.
- `migrate-legacy-links` - ensuring menu links use route-based URLs.
