---
name: migrate-module-full-rewrite
description: Orchestrate a complete legacy-to-modern module rewrite by sequencing individual migration skills in the correct dependency order. Use when a module has multiple legacy patterns and needs a systematic, incremental migration plan.
---

## Requirements

- Module technical name and path.
- Current minimum PS version supported; target PS version (9.x).
- List of legacy patterns present (run the audit in step 1).
- Decision: big-bang rewrite vs incremental (feature branch per pattern). Incremental is strongly recommended.

## Steps

1. **Audit the module.** Identify every legacy pattern present:

    ```bash
    # Translations
    grep -rn '\$this->l(' modules/mymodule/
    # ObjectModel
    grep -rn 'extends ObjectModel' modules/mymodule/
    # Raw SQL
    grep -rn 'Db::getInstance()' modules/mymodule/
    # Legacy controller
    grep -rn 'extends ModuleAdminController' modules/mymodule/
    # HelperForm / HelperList
    grep -rn 'HelperForm\|HelperList' modules/mymodule/
    # Hook::exec
    grep -rn 'Hook::exec(' modules/mymodule/
    # Legacy links
    grep -rn 'index\.php?controller=Admin' modules/mymodule/
    # Legacy AJAX
    grep -rn 'ajaxProcess' modules/mymodule/
    # Overrides
    ls modules/mymodule/override/ 2>/dev/null
    # Legacy tabs
    grep -rn 'new Tab()' modules/mymodule/
    ```

2. **Plan migration order.** Follow this dependency chain (earlier layers must be done first):

    | Order | Pattern | Skill |
    |-------|---------|-------|
    | 1 | Translations (`$this->l()`) | `migrate-legacy-translations` |
    | 2 | Hook dispatch (`Hook::exec()`) | `migrate-hook-exec-to-dispatcher` |
    | 3 | Persistence - ObjectModel to Doctrine | `migrate-objectmodel-to-doctrine` |
    | 4 | Persistence - Raw SQL to Doctrine QB | `migrate-legacy-sql-to-doctrine-qb` |
    | 5 | Domain logic - ObjectModel CRUD to CQRS | `migrate-objectmodel-to-cqrs` |
    | 6 | Forms - HelperForm to Symfony Form | `migrate-helperform-to-symfony-form` |
    | 7 | Lists - HelperList to Grid | `migrate-helperlist-to-grid` |
    | 8 | Controllers - AdminController to Symfony | `migrate-admin-controller` |
    | 9 | AJAX endpoints | `migrate-legacy-ajax` |
    | 10 | Links and URLs | `migrate-legacy-links` |
    | 11 | Tabs and menu items | `migrate-legacy-tabs-to-routes` |
    | 12 | Overrides to decorators/listeners | `migrate-override-to-decorator` |

3. **Execute pattern-by-pattern.** For each row in the plan:
   - Create a feature branch: `git checkout -b migrate/<pattern-name>`
   - Follow the corresponding skill's Steps.
   - Run the module test suite after each migration.
   - Merge to the main migration branch when green.

4. **Update `composer.json`** metadata:
   - Bump `"ps_versions_compliancy"` min to `9.0.0`.
   - Ensure `"autoload"` PSR-4 maps `src/` correctly.
   - Add Doctrine and Symfony dependencies if newly introduced.

5. **Validate the final state:**

    ```bash
    # Install round-trip
    bin/console prestashop:module:uninstall mymodule
    bin/console prestashop:module:install mymodule

    # Linters
    bin/console prestashop:linter:legacy-link
    php vendor/bin/phpstan analyse modules/mymodule/src -l 6

    # Verify no legacy patterns remain
    grep -rn 'extends ModuleAdminController\|HelperForm\|HelperList\|Hook::exec\|Db::getInstance\|\$this->l(' modules/mymodule/src/
    ```

6. **Update module documentation** (README, CHANGELOG) noting the minimum PS version bump and any breaking changes for merchants.

## Do

- Migrate incrementally: one pattern per commit/PR, always keeping the module installable.
- Run `bin/console prestashop:module:install` + `uninstall` after each major step to catch registration issues early.
- Keep the legacy code functional on a parallel branch until the migration is validated end-to-end.

## Don't

- Don't migrate everything in a single commit; partial regressions become impossible to bisect.
- Don't skip the translations step; `$this->l()` calls in modern code trigger deprecation warnings.
- Don't leave dead legacy files (old controllers, override/) in the final tree.

## Related skills

- `migrate-legacy-translations` - step 1 in the migration order.
- `migrate-hook-exec-to-dispatcher` - step 2.
- `migrate-objectmodel-to-doctrine` - step 3.
- `migrate-legacy-sql-to-doctrine-qb` - step 4.
- `migrate-objectmodel-to-cqrs` - step 5.
- `migrate-helperform-to-symfony-form` - step 6.
- `migrate-helperlist-to-grid` - step 7.
- `migrate-admin-controller` - step 8.
- `migrate-legacy-ajax` - step 9.
- `migrate-legacy-links` - step 10.
- `migrate-legacy-tabs-to-routes` - step 11.
- `migrate-override-to-decorator` - step 12.
