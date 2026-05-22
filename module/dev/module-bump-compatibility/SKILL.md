---
name: module-bump-compatibility
description: Update an existing PrestaShop module to support a newer minor or major (e.g. 8.x to 9.0, 9.0 to 9.1) - bump `ps_versions_compliancy`, migrate legacy patterns to modern equivalents, audit overrides and dependency constraints, and round-trip the install on the target. Use when the user wants to declare support for a new PS version.
---

## Requirements

Ask the user:
* Current `ps_versions_compliancy` from `<modulename>.php` (the `min` and `max` PS versions the module currently advertises) and the target `max` (e.g. `9.0.99`, `9.1.99`).
* The current `$this->version` and the new version they want to ship as part of the bump (a minor or patch bump is conventional).
* What changed in the target PS version. Read the corresponding "Changes in <version>" page under the [Core Updates index](https://devdocs.prestashop-project.org/9/modules/core-updates/) before touching code; major versions drop classes and rename services, minors usually only deprecate.
* A clean target shop on the new version available for the install/uninstall round-trip. The bump cannot be validated without one.
* Whether the module ships overrides (under `override/`) - PS 9 dropped many Core class methods, every override needs a signature audit.
* Whether the module ships its own `composer.json` - any `prestashop/*` or `symfony/*` constraint may need to widen for the new minimum.

## Steps

1. Bump the declared range in `<modulename>.php`. Use `_PS_VERSION_` boundaries that include the target's last patch (the `.99` convention) so the module installs on every patch release of the target minor:

    ```php
    public function __construct()
    {
        $this->name             = 'mymodule';
        $this->author           = 'MyVendor';
        $this->version          = '2.4.0';                 // bump alongside ps_versions_compliancy
        $this->ps_versions_compliancy = ['min' => '9.0.0', 'max' => '9.1.99'];

        parent::__construct();
        // ...
    }
    ```

   Do not widen `min` past what you actually test against. Lying about the lower bound creates support tickets the moment a merchant tries to install.

2. Replace every `Hook::exec()` call with a `HookDispatcherInterface` injection. The legacy static call still exists in PS 9 but is on the deprecation path; the modern dispatcher is what tests and Symfony controllers can mock. Inject `PrestaShop\PrestaShop\Core\Hook\HookDispatcherInterface` and call `dispatchWithParameters()`. See `module-register-hooks` for the full DI wiring.

    ```php
    // Before
    Hook::exec('actionMyModuleEvent', ['order' => $order]);

    // After
    $this->hookDispatcher->dispatchWithParameters('actionMyModuleEvent', ['order' => $order]);
    ```

3. Replace any `HelperList`-backed Back Office listing with a Grid (`GridDefinitionFactory` + Doctrine `QueryBuilder` + the `grid_panel.html.twig` macro). `HelperList` still renders but does not hook into the modern filter/bulk/toggle pipeline. See `module-add-grid` for the full migration; for Marketplace submission this is now expected on every BO list.

4. Replace `$this->l('...')` (legacy on-the-fly translation) with the Symfony Translator and a proper translation domain. The new key shape is `Modules.<modulename>.<area>` (`Admin`, `Front`, `Shop`, etc.). Add the matching XLIFF entry under `translations/<locale>/Modules.<modulename>.<area>.xlf`. See `module-add-translations-new` for the full workflow:

    ```php
    // Before
    echo $this->l('Save');

    // After (in a controller)
    $this->trans('Save', [], 'Modules.Mymodule.Admin');
    // After (in Twig)
    {{ 'Save'|trans({}, 'Modules.Mymodule.Admin') }}
    ```

5. Hunt down legacy admin links and convert them to Symfony routes. The `legacy-link` linter walks controllers, templates and PHP for `index.php?controller=...&token=...` patterns and reports each one. Replace each match with a Symfony `url()` / `path()` call against a route declared in `config/routes.yml`:

    ```bash
    cd /path/to/prestashop
    bin/console prestashop:linter:legacy-link modules/mymodule
    ```

6. Audit every override. PS 9 removed many legacy Core class methods (notably on `AdminController` subclasses, `Tools`, `Validate`, `ObjectModel` accessors). For each file under `override/`, open the matching Core class on the target version and verify the overridden method still exists with the same signature. If it does not, prefer migrating to a service decorator (`#[AsDecorator]`) on the modern service rather than re-overriding a legacy class. Modern Symfony admin controllers cannot be overridden via `override/`; use service decoration. See `module-add-override` for the decision tree.

7. Audit `composer.json`. If the module pins `prestashop/*` (e.g. `prestashop/decimal`, `prestashop/circuit-breaker`) or `symfony/*` packages, widen the constraints to also satisfy what the target PS version ships:

    ```bash
    cd modules/mymodule
    composer outdated --direct
    composer why-not symfony/console "^7.1"   # check what blocks the upgrade
    composer update --no-install              # regenerate lock against the new constraints
    ```

   Re-run `composer validate --strict` after every constraint change.

8. Round-trip the install on a fresh shop running the **target** PrestaShop version. The local linters and unit tests will not catch every regression; the install path is the only place where missing classes, dropped DI tags and renamed services surface:

    ```bash
    cd /path/to/prestashop-target
    bin/console prestashop:module install   mymodule
    bin/console prestashop:module uninstall mymodule
    # both must return 0; ps_module / ps_hook_module must end with zero rows for mymodule
    ```

   If the schema changed between the current and the target version (new column, new index, dropped table), ship the change as `upgrade/upgrade-<new-version>.php` and bump `$this->version` in the same commit. The upgrade script function is named `upgrade_module_<version_with_underscores>($module)` and is triggered by `bin/console prestashop:module upgrade mymodule`. See `module-add-migration`.

9. Re-run the full validation pipeline before tagging. Composer validate, unit tests, naming-convention linter, legacy-link linter, install/uninstall round-trip, and the official Marketplace Validator. See `module-validate` for the canonical sequence.

10. Update the module CHANGELOG with the version bump, the supported PS range, and a one-line note for every breaking deprecation you migrated (so merchants reading the changelog understand what changed in their database).

## Do

- Bump `ps_versions_compliancy['max']` and `$this->version` in the same commit. Reviewers and the BO upgrade flow expect them to move together.
- Read the [Core Updates page for the target version](https://devdocs.prestashop-project.org/9/modules/core-updates/) before changing code. It lists every dropped class, renamed service and DI tag rename you need to migrate.
- Run `bin/console prestashop:linter:legacy-link` and migrate every hit to a Symfony route. PS 9 actively removes legacy admin endpoints; old `index.php?controller=...` URLs break silently.
- Round-trip install/uninstall on a fresh shop running the **target** PS version, not the current one. Many regressions only surface during install, when DI containers compile.
- Audit overrides one by one against the target's Core source. PS 9 dropped enough methods that a previously-working override may now silently extend nothing.
- Re-run the full `module-validate` pipeline after every change in this skill. The bump is not done until composer validate, linters, tests, install round-trip and the Validator all pass on the target.

## Don't

- Don't widen `ps_versions_compliancy['min']` without testing against that lower bound. Saying you support 8.0 when you only test on 9.0 turns into a flood of support tickets.
- Don't keep `Hook::exec()` calls "for now" while bumping. Migrate them in the same PR; otherwise the next minor will move them onto an actively-removed code path.
- Don't keep `HelperList`-based BO lists in a modern bump. Grids are the expected surface; reviewers will reject mixed listings.
- Don't keep `$this->l('...')` calls. The legacy translator still resolves but does not produce extractable keys; your translations stop syncing with Crowdin / the project translation pipeline.
- Don't keep an override that only re-declares a method with no behaviour change. It still counts as a conflict surface and may break on the next core method removal.
- Don't ship the bump without an entry in `upgrade/upgrade-<new-version>.php` if any database schema, hook registration or configuration default changed. The upgrade path is the only thing that runs on existing installs.
- Don't run `doctrine:schema:update --force` to "fix" Doctrine drift on the target. PrestaShop's [Doctrine page](https://devdocs.prestashop-project.org/9/modules/concepts/doctrine/) explicitly warns against it; ship a migration instead.

## Canonical examples

- [devdocs - Modules / Core updates](https://devdocs.prestashop-project.org/9/modules/core-updates/) - the index of "Changes in <version>" pages every bump must read first.
- [devdocs - Modules / Concepts](https://devdocs.prestashop-project.org/9/modules/concepts/) - Doctrine, services, hooks, multistore reference for the modern equivalents you migrate to.
- [devdocs - Modules / Creation / Enabling auto-update](https://devdocs.prestashop-project.org/9/modules/creation/enabling-auto-update/) - upgrade-<version>.php contract triggered by `prestashop:module upgrade`.
- [devdocs - Migration guide](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/) - the canonical playbook for moving legacy admin pages, controllers, hooks and grids to the modern stack.
- [`HookDispatcherInterface`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Hook/HookDispatcherInterface.php) and [`CommandBusInterface`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/CommandBus/CommandBusInterface.php) - the modern FQCNs you wire up when removing `Hook::exec()` and `ObjectModel`-based mutations.

## Related skills

- `module-register-hooks` - migrate `Hook::exec()` to `HookDispatcherInterface`.
- `module-add-grid` - replace `HelperList`-backed BO listings.
- `module-add-translations-new` - replace `$this->l(...)` with Symfony Translator and `Modules.<modulename>.<area>` keys.
- `module-add-symfony-route` - replace legacy admin links.
- `module-add-override` - decide whether each `override/` file should stay, become a service decorator, or be deleted.
- `module-add-migration` - ship `upgrade/upgrade-<version>.php` when the schema changed.
- `module-validate` - the validation pipeline you re-run after the bump.
- `module-package-zip` - rebuild the distributable zip with the new `$this->version`.
