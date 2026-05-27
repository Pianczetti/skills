---
name: module-legacy-migrator
description: Migrates legacy PrestaShop modules to modern PS 9 architecture pattern by pattern.
---

# Module Legacy Migrator

## Role
Takes an existing legacy PrestaShop module (using ObjectModel, HelperForm, HelperList, AdminController, `$this->l()`, `Hook::exec()`) and rewrites it to use modern PS 9 architecture. Performs an initial audit, creates a migration plan, and executes each pattern migration in sequence while preserving existing behavior.

## Context
Legacy modules written for PrestaShop 1.6/1.7/8.x use patterns that are deprecated or unsupported in PS 9: `ObjectModel` for persistence, `HelperForm`/`HelperList` for back-office UI, `AdminController` for admin pages, `$this->l()` for translations, `Hook::exec()` for hook dispatching, and raw SQL via `Db::getInstance()`. Modern PS 9 replaces these with Doctrine ORM, Symfony Forms + Grid, `PrestaShopAdminController`, XLIFF translations, `HookDispatcherInterface`, and CQRS. The migrator rewrites each pattern while maintaining functional equivalence.

## Skills Used
- **module-bump-compatibility** - update `ps_versions_compliancy` and composer constraints to target PS 9
- **migrate-admin-controller** - rewrite legacy `AdminController` to modern Symfony `PrestaShopAdminController`
- **migrate-helperform-to-symfony-form** - replace `HelperForm` with Symfony Form + FormHandler + FormDataProvider
- **migrate-helperlist-to-grid** - replace `HelperList` with Grid (GridDefinitionFactory + Doctrine QueryBuilder)
- **migrate-objectmodel-to-cqrs** - extract ObjectModel mutations into CQRS commands and queries
- **migrate-objectmodel-to-doctrine** - replace ObjectModel persistence with Doctrine ORM entities
- **migrate-hook-exec-to-dispatcher** - replace `Hook::exec()` with `HookDispatcherInterface`
- **migrate-legacy-translations** - replace `$this->l()` with Symfony Translator and XLIFF catalogs
- **migrate-legacy-links** - replace `Link::getAdminLink()` / `Link::getModuleLink()` with Symfony Router
- **migrate-legacy-ajax** - replace `ajaxProcess*` methods with proper Symfony AJAX controllers
- **migrate-override-to-decorator** - replace class overrides with service decorators or event listeners
- **migrate-legacy-sql-to-doctrine-qb** - replace raw `Db::getInstance()` queries with Doctrine QueryBuilder
- **migrate-legacy-tabs-to-routes** - replace Tab registration with Symfony route declarations
- **migrate-module-full-rewrite** - orchestrate a complete module rewrite when incremental migration is impractical
- **module-validate** - validate the migrated module passes all checks
- **module-add-unit-tests** - add tests for newly created handlers and services

## Workflow
1. Audit the existing module: identify all legacy patterns in use (ObjectModel classes, AdminControllers, HelperForm/HelperList usage, translation calls, hook dispatching, raw SQL, overrides, Tab registrations).
2. Run **module-bump-compatibility** to update version constraints and identify breaking changes.
3. Plan the migration order based on dependencies: persistence first (ObjectModel to Doctrine), then CQRS layer, then controllers, then UI, then translations.
4. For each ObjectModel, run **migrate-objectmodel-to-doctrine** to create the Doctrine entity and repository.
5. For each write operation on the old ObjectModel, run **migrate-objectmodel-to-cqrs** to extract CQRS commands.
6. For each read operation, extract CQRS queries.
7. Replace raw SQL (`Db::getInstance()`) with Doctrine QueryBuilder via **migrate-legacy-sql-to-doctrine-qb**.
8. For each legacy AdminController, run **migrate-admin-controller** to create a modern Symfony controller.
9. For each HelperForm, run **migrate-helperform-to-symfony-form** to create a proper Form type with data provider.
10. For each HelperList, run **migrate-helperlist-to-grid** to create a Grid definition.
11. Replace Tab registrations with Symfony routes via **migrate-legacy-tabs-to-routes**.
12. Replace `$this->l()` calls with the Symfony Translator via **migrate-legacy-translations**.
13. Replace `Hook::exec()` with `HookDispatcherInterface` via **migrate-hook-exec-to-dispatcher**.
14. Replace `ajaxProcess*` methods with Symfony AJAX controllers via **migrate-legacy-ajax**.
15. Replace class overrides with service decorators via **migrate-override-to-decorator**.
16. Replace legacy link generation with the Symfony Router via **migrate-legacy-links**.
17. Run **module-add-unit-tests** to cover the new CQRS handlers and services.
18. Run **module-validate** to verify the migrated module installs, uninstalls, and passes all checks.

## Rules
- Never delete the old code until the new code is verified to work. Keep the old files in a `_legacy/` folder during migration.
- Maintain functional equivalence: every feature the legacy module had must work identically after migration.
- Migrate one pattern at a time. Each step must leave the module in an installable state.
- Preserve all existing hook registrations. The module must continue to respond to the same hooks.
- Preserve database schema compatibility. Existing shop data must not be lost. Add migration scripts if columns or tables change.
- Test each migration step before proceeding to the next.
- Document any behavior changes or breaking changes in a MIGRATION.md file.

## Output Expectations
- A fully modernized module that installs cleanly on PrestaShop 9.
- All legacy patterns replaced with modern equivalents.
- A MIGRATION.md documenting what changed, any breaking changes, and upgrade steps for existing users.
- Unit tests covering all new CQRS handlers and services.
- `module-validate` passes with zero errors.
- Existing shop data is preserved through the upgrade path.
