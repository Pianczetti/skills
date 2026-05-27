---
name: module-modifier
description: Modifies existing modern PrestaShop 9 modules to add features, fix bugs, or extend functionality.
---

# Module Modifier

## Role
Takes an existing modern PrestaShop 9 module and applies targeted modifications: adding new features, fixing bugs, extending existing functionality, or wiring new integrations. Understands the module's existing structure and conventions and applies changes that are consistent with its architecture.

## Context
The module being modified already follows PS 9 modern conventions: PSR-4 layout, Symfony services, CQRS for state management, Doctrine for persistence, modern admin controllers, Grid for listings, and the new translation system. The modifier must respect existing patterns, naming conventions, and architectural decisions. It must not introduce legacy patterns or break existing functionality.

## Skills Used
- **module-add-front-controller** - add new customer-facing pages or AJAX endpoints
- **module-add-admin-controller-modern** - add new back-office pages
- **module-add-config-page-modern** - add or extend configuration screens
- **module-add-cqrs-command** - add new write operations
- **module-add-cqrs-query** - add new read operations
- **module-add-doctrine-entity** - add new entities or extend existing ones
- **module-add-grid** - add new listing pages
- **module-add-symfony-route** - declare new routes
- **module-add-service** - register new services
- **module-register-hooks** - register additional hook listeners
- **module-add-widget** - add new display widgets
- **module-add-mail-template** - add email templates
- **module-add-translations-new** - wire new translatable strings
- **module-add-migration** - add upgrade scripts for schema changes
- **module-make-multistore-aware** - add multistore support to new features
- **module-add-unit-tests** - test new handlers and services
- **module-add-feature-flag** - gate new features behind a toggle
- **module-add-cli-command** - add console commands for maintenance tasks
- **module-validate** - verify the modified module still passes all checks

## Workflow
1. Read and understand the existing module structure: main class, entities, CQRS layer, controllers, services, hooks, configuration.
2. Identify what needs to change based on the user's request: new feature, bug fix, extension, or integration.
3. Plan the changes: which files to modify, which new files to create, which skills to invoke.
4. If the change introduces a new entity or alters a schema, run **module-add-doctrine-entity** and **module-add-migration** for the upgrade script.
5. If the change requires new state mutations, run **module-add-cqrs-command**.
6. If the change requires new data retrieval, run **module-add-cqrs-query**.
7. Register any new services with **module-add-service**.
8. If the change needs a new back-office page, run **module-add-symfony-route** and **module-add-admin-controller-modern**.
9. If the change needs a new front-office endpoint, run **module-add-front-controller**.
10. If the change needs a new listing, run **module-add-grid**.
11. Wire new hooks with **module-register-hooks** if needed.
12. If the feature is risky or partial, wrap it with **module-add-feature-flag**.
13. Add translations for all new user-facing strings with **module-add-translations-new**.
14. Write unit tests for new code with **module-add-unit-tests**.
15. Bump the module version in `config.xml` and the main class, and add an upgrade script via **module-add-migration** if the database schema changed.
16. Run **module-validate** to verify the module still installs, uninstalls, and passes all checks.

## Rules
- Read the existing module code before making any changes. Never blindly add code that conflicts with existing patterns.
- Follow the existing naming conventions in the module (namespace structure, service naming, route naming).
- Every schema change requires an upgrade script. Never modify install SQL without a corresponding `upgrade-<version>.php`.
- Do not break existing functionality. Run existing tests before and after changes.
- Keep new features consistent with the module's existing architecture. If it uses a repository pattern, add repositories. If it uses value objects, add value objects.
- Never introduce legacy patterns into a modern module.
- Always bump the module version when adding features.

## Output Expectations
- Modified module with the requested changes applied cleanly.
- All existing tests still pass.
- New tests cover the added functionality.
- If a version bump occurred, an upgrade script handles schema migration.
- `module-validate` passes with zero errors.
- The module installs, uninstalls, and enables/disables without errors on PS 9.
