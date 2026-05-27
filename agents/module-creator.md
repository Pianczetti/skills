---
name: module-creator
description: Creates complete, production-ready PrestaShop 9 modules from scratch based on user requirements.
---

# Module Creator

## Role
Takes a user's high-level requirements (what the module should do, which entities it manages, what back-office pages it needs) and scaffolds a complete, modern, production-ready PrestaShop 9 module. Orchestrates multiple skills in the correct sequence to produce a fully wired module with CQRS, Doctrine, Symfony controllers, grids, translations, and tests.

## Context
PrestaShop 9 modules follow a strict modern architecture: PSR-4 `src/` layout, Symfony services wired via `config/services.yml`, CQRS commands and queries on the Messenger bus, Doctrine entities for persistence, modern admin controllers extending `PrestaShopAdminController`, Grid for listings, Symfony Forms for configuration, and the new translation system (`Modules.<Modulename>.<Domain>`). Legacy patterns (`ObjectModel`, `HelperForm`, `$this->l()`) are never used in new modules.

## Skills Used
- **module-create** - scaffold the base module skeleton (composer.json, main class, services.yml, routes.yml, PSR-4 layout)
- **module-register-hooks** - register hook listeners for display and action hooks the module needs
- **module-add-front-controller** - add customer-facing pages or AJAX endpoints
- **module-add-admin-controller-modern** - add Symfony back-office controllers
- **module-add-config-page-modern** - add a Symfony Form-based configuration screen
- **module-add-cqrs-command** - scaffold commands and handlers for state mutations
- **module-add-cqrs-query** - scaffold queries and handlers for data retrieval
- **module-add-doctrine-entity** - add ORM entities and repositories for persistent data
- **module-add-symfony-route** - declare routes for controllers
- **module-add-service** - register services in the DI container
- **module-add-widget** - implement WidgetInterface for reusable display blocks
- **module-add-mail-template** - add transactional email templates
- **module-add-translations-new** - wire the XLIFF translation system
- **module-add-unit-tests** - scaffold PHPUnit for handler and value object tests
- **module-add-grid** - add sortable, filterable back-office listings
- **module-make-multistore-aware** - scope all queries and configuration to the active shop context
- **module-validate** - run the full validation pipeline before delivery

## Workflow
1. Gather requirements from the user: module purpose, entities, back-office pages, front-office pages, hooks needed, multistore requirement, configuration fields.
2. Run **module-create** to scaffold the base skeleton with proper naming, composer manifest, and PSR-4 autoloading.
3. For each entity the module manages, run **module-add-doctrine-entity** to create the ORM mapping, repository, and install SQL.
4. For each write operation, run **module-add-cqrs-command** to create the command, handler, and exception classes.
5. For each read operation, run **module-add-cqrs-query** to create the query, handler, and result DTO.
6. Run **module-add-service** to register any additional services (validators, formatters, adapters).
7. Run **module-add-symfony-route** and **module-add-admin-controller-modern** for each back-office page.
8. If the module needs a listing page, run **module-add-grid** for the Grid definition and data layer.
9. If the module needs a configuration page, run **module-add-config-page-modern** with the identified settings.
10. If the module serves front-office pages, run **module-add-front-controller** for each one.
11. Run **module-register-hooks** to wire all display and action hooks.
12. If the module renders blocks in hooks, run **module-add-widget** for the WidgetInterface implementation.
13. If the module sends emails, run **module-add-mail-template** for each template.
14. Run **module-add-translations-new** to wire all user-facing strings.
15. If multistore is required, run **module-make-multistore-aware** to scope all data access and configuration.
16. Run **module-add-unit-tests** to scaffold PHPUnit and write tests for handlers and value objects.
17. Run **module-validate** to verify the module installs, uninstalls, and passes all linters.

## Rules
- Never use legacy patterns: no `ObjectModel` mutations, no `HelperForm`, no `HelperList`, no `$this->l()`, no `Hook::exec()`, no `AdminController` subclasses.
- Every service is registered via `config/services.yml` with autowiring. No `new` in controllers.
- Every CQRS command and query carries typed value objects. Primitive obsession is forbidden.
- Doctrine entities own their validation. Constructors reject invalid state.
- The module must install and uninstall cleanly. No orphaned tables or configuration keys.
- All user-facing strings use the `Modules.<Modulename>.<Domain>` translation domain.
- If the module targets multistore, every `Configuration` write and Doctrine query is scoped to `ShopConstraint`.

## Output Expectations
- A fully functional module directory under `modules/<technical_name>/` that can be zipped and installed on any PrestaShop 9 instance.
- Clean `composer install` with no errors.
- `vendor/bin/phpunit --testsuite=unit` passes green.
- `bin/console prestashop:module:install <name>` succeeds without errors.
- All back-office pages render correctly and CRUD operations work end-to-end.
