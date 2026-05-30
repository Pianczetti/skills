---
name: migrate-module-full-rewrite
description: Orchestrate a complete module migration from legacy patterns to PS 9 modern stack by auditing, planning, and executing pattern-by-pattern using sibling migration skills. Use when a module needs a full rewrite rather than incremental fixes.
---

## Requirements

- Module name, vendor namespace, and current PS version compatibility range.
- Target PS version (e.g. `9.0`, `9.1`).
- Access to the module source code and a running PS 9 test instance.
- Confirmation that a full rewrite (not incremental migration) is acceptable.

## Steps

1. **Audit** - scan for all legacy patterns:

    ```bash
    echo "=== Hook::exec ===" && grep -rn 'Hook::exec(' modules/mymodule/
    echo "=== HelperList ===" && grep -rn 'HelperList\|fields_list\|renderList' modules/mymodule/
    echo "=== HelperForm ===" && grep -rn 'HelperForm\|fields_form\|getContent' modules/mymodule/
    echo "=== ObjectModel mutations ===" && grep -rn '->add()\|->update()\|->delete()\|->save()' modules/mymodule/
    echo "=== Legacy translations ===" && grep -rn '\$this->l(' modules/mymodule/
    echo "=== Legacy links ===" && grep -rn 'index.php?controller=\|getAdminLink(' modules/mymodule/
    echo "=== Legacy AJAX ===" && grep -rn 'ajaxProcess' modules/mymodule/
    echo "=== Raw SQL ===" && grep -rn 'Db::getInstance()' modules/mymodule/
    echo "=== Overrides ===" && find modules/mymodule/override/ -name '*.php' 2>/dev/null
    echo "=== Legacy tabs ===" && grep -rn 'Tab::' modules/mymodule/
    ```

2. **Plan** - execute in this order to avoid cascading rework:
   1. `migrate-legacy-translations` (no structural dependencies)
   2. `migrate-hook-exec-to-dispatcher` (pure refactor)
   3. `migrate-objectmodel-to-doctrine` (persistence layer)
   4. `migrate-objectmodel-to-cqrs` (domain layer)
   5. `migrate-legacy-sql-to-doctrine-qb` (remaining raw SQL)
   6. `migrate-helperform-to-symfony-form` (forms)
   7. `migrate-helperlist-to-grid` (listings)
   8. `migrate-admin-controller` (controllers)
   9. `migrate-legacy-ajax` (AJAX endpoints)
   10. `migrate-legacy-links` (URL references)
   11. `migrate-legacy-tabs-to-routes` (menu entries)
   12. `migrate-override-to-decorator` (overrides)

3. **Execute** each migration skill in sequence. After each skill completes, run:

    ```bash
    bin/console cache:clear
    bin/console prestashop:module uninstall mymodule && bin/console prestashop:module install mymodule
    ```

4. **Validate** the fully migrated module:

    ```bash
    bin/console prestashop:module install mymodule
    bin/console prestashop:module uninstall mymodule
    composer validate --strict --working-dir=modules/mymodule
    bin/console prestashop:linter:legacy-link modules/mymodule
    vendor/bin/phpunit --configuration=modules/mymodule/phpunit.xml
    ```

5. Bump `ps_versions_compliancy` and `$this->version` using `module-bump-compatibility`.

6. Update CHANGELOG with migration notes for each pattern replaced.

## Do

- Follow the prescribed order; translations and hooks have no dependencies while controllers depend on forms and grids.
- Commit after each pattern migration so the history is bisectable.
- Run install/uninstall round-trip after each major step to catch DI compilation errors early.

## Don't

- Don't skip the audit step; unknown patterns discovered mid-migration cause rework.
- Don't migrate controllers before forms and grids are ready; the controller needs to render them.
- Don't ship a partial migration; if blocked on one pattern, document it and complete the rest.

## Related skills

- `migrate-legacy-admin-page` - phased orchestrator with feature-flag gating, Behat/Playwright test gates, and listing/form slices. Use that skill when the module has a legacy admin page and you want a structured, milestone-driven delivery. Use this skill (`migrate-module-full-rewrite`) when the module has no admin page or needs a pure pattern-by-pattern sweep.
- `migrate-admin-controller` - controller migration.
- `migrate-helperform-to-symfony-form` - form migration.
- `migrate-helperlist-to-grid` - grid migration.
- `migrate-objectmodel-to-cqrs` - CQRS migration.
- `migrate-objectmodel-to-doctrine` - persistence migration.
- `migrate-hook-exec-to-dispatcher` - hook migration.
- `migrate-legacy-translations` - translation migration.
- `migrate-legacy-links` - link migration.
- `migrate-legacy-ajax` - AJAX migration.
- `migrate-legacy-sql-to-doctrine-qb` - SQL migration.
- `migrate-legacy-tabs-to-routes` - tab migration.
- `migrate-override-to-decorator` - override migration.
- `module-bump-compatibility` - final version bump.
- `module-validate` - final validation pipeline.
