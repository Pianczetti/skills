---
name: migrate-legacy-admin-page
description: Step-by-step orchestrator for migrating a legacy admin page (ObjectModel + AdminXxxController) inside a PrestaShop 9 module to modern Symfony/CQRS architecture with feature-flag gating, Behat and Playwright test gates, and a phased delivery (listing slice, form slice). Use when the user says "migrate the Xxx admin page" or the module carries a legacy admin controller that must move to the PS 9 Symfony stack.
---

## When to use this skill

Trigger on any of:
- "Migrate the Xxx admin page in module Y"
- "Rewrite AdminXxxController to Symfony"
- "Add CQRS for the Xxx domain in module Y"
- "Modernize the module's back-office"

This skill **orchestrates** existing skills. Each phase below names the sibling skills to invoke; the procedural detail lives in those skills.

## Requirements

Ask the user:
* Module technical name, root path, vendor namespace.
* Which legacy admin controller(s) to migrate (e.g. `AdminMymoduleVouchersController`).
* Target: full migration (listing + form) or partial (listing only / CQRS only)?
* Whether bulk actions exist in the legacy list.
* Multistore: none / simple shop association / per-shop content?
* Running PS 9 test instance with the module installed (needed for Behat and Playwright gates).

## Phase 0 - Audit

Before writing any new code, build a complete map of what the legacy page does:

1. Run grep scans against the legacy controller and ObjectModel:

    ```bash
    echo "=== Fields ===" && grep -n 'fields_list\|fields_form\|$definition' modules/mymodule/classes/Voucher.php modules/mymodule/controllers/admin/AdminMymoduleVouchersController.php
    echo "=== Actions ===" && grep -n 'ajaxProcess\|processDelete\|processBulk\|processStatus\|processPosition' modules/mymodule/controllers/admin/AdminMymoduleVouchersController.php
    echo "=== Hooks ===" && grep -rn 'Hook::exec\|hookAction\|hookDisplay' modules/mymodule/
    echo "=== Sub-resources ===" && grep -rn 'ObjectModel\|class.*extends' modules/mymodule/classes/
    echo "=== Raw SQL ===" && grep -rn 'Db::getInstance()' modules/mymodule/
    ```

2. Produce a **migration manifest** (markdown table) listing:
   - Every DB field with type, nullable, translatable, multistore tier.
   - Every grid column (existing `fields_list` entry).
   - Every action (CRUD, bulk, toggle, position, AJAX).
   - Every hook dispatched or listened to.
   - Sub-resources that need their own commands.

3. Choose a **milestone strategy**:

   | Strategy | When |
   |----------|------|
   | Single sprint | Simple entity (< 10 fields, no sub-resources, no multistore). |
   | Listing first, form later | Complex entity; listing unblocks bulk actions immediately. |
   | CQRS first, then UI | Team wants the API layer solid before building the interface. |

   Document the decision in the PR description.

## Phase 1 - Feature flag

Register a feature flag so the legacy controller stays active during development and the new Symfony routes only activate when the flag is enabled.

**Invoke:** `module-add-feature-flag`

The flag name becomes the `_legacy_feature_flag` route attribute in all new routes. This is the routing mechanism that toggles between old and new pages.

## Phase 2 - Domain layer (Commands, Queries, Value Objects)

Build the domain layer under `src/Domain/<Aggregate>/` - no implementation, only contracts. This is framework-agnostic PHP: no Doctrine, no Symfony, no ObjectModel.

**Invoke in order:**
1. `module-add-cqrs-command` - AddXxxCommand, EditXxxCommand, DeleteXxxCommand, ToggleXxxStatusCommand, + {Aggregate}Id value object, + {Aggregate}Exception hierarchy.
2. `module-add-cqrs-bulk-command` (conditional: if manifest lists bulk actions) - BulkDeleteXxxCommand, BulkToggleXxxStatusCommand.
3. `module-add-cqrs-query` - GetXxxForEditing query, EditableXxx DTO with readonly typed getters.

Gate: every field in the manifest is covered by a command or query parameter/DTO field.

## Phase 3 - Adapter layer (Repository, Handlers, DI)

Implement the contracts from Phase 2:

**Invoke:**
1. `module-add-doctrine-entity` - entity + repository (persistence layer replacing ObjectModel).
2. `module-add-service` (as needed) - register additional services (file uploaders, decorators).
3. Implement command/query handlers using the repository. Follow the patterns from `module-add-cqrs-command` and `module-add-cqrs-query` (the handler extends nothing special for single commands, and `AbstractBulkCommandHandler` for bulk).

Gate: `bin/console cache:clear` compiles without error.

## Phase 4 - Behat tests (behaviour gate)

Cover the domain layer with Behat scenarios BEFORE building the UI. This is the quality gate: if the domain logic is broken, building a grid/form on top will produce confusing failures.

**Invoke:** `module-add-behat-tests`

Cover: CRUD lifecycle, constraint errors, bulk actions (partial failure), sub-resources if any.

Gate: `vendor/bin/behat -c modules/mymodule/tests/Behaviour/behat.yml` passes green.

## Phase 5 - Listing page (vertical slice)

Build the complete listing (grid + controller + route + Twig template + JS) as one coherent slice.

**Invoke in order:**
1. `module-add-grid` - GridDefinitionFactory + DoctrineQueryBuilder + filters.
2. `module-add-admin-controller-modern` - controller with list action + row/bulk actions.
3. `module-add-symfony-route` - routes with `_legacy_feature_flag` attribute.

The listing controller serves as the "index" action; the form controller (Phase 6) will extend the same controller class or create a sibling.

Gate: with the feature flag enabled, the grid renders correctly, filter/sort/paginate work, row and bulk actions dispatch to the correct routes.

## Phase 6 - Form page (vertical slice)

Build the create/edit form as a second slice:

**Invoke:**
1. `module-add-config-page-modern` (for settings-type pages) OR for a full CRUD form, invoke the Form skills from `module-add-config-page-modern` adapted to the IdentifiableObject pattern:
   - FormType with typed fields matching the command/DTO.
   - FormDataProvider reading from the query bus.
   - FormDataHandler dispatching add/edit commands.
2. Extend the controller with create/edit/delete actions.
3. Add Twig form template.

Gate: create, edit, delete cycle works end-to-end with validation errors displayed correctly.

## Phase 7 - Playwright tests (UI gate)

Cover the UI with browser-level acceptance tests:

**Invoke:** `module-add-playwright-tests`

Campaigns to write:
- `01_CRUD.ts` - create, verify in grid, edit, delete.
- `02_filterSort.ts` - filter by each column, sort ascending/descending.
- `03_bulkActions.ts` (conditional) - select rows, bulk delete/enable/disable.
- `04_position.ts` (conditional) - drag-and-drop reorder if position column exists.

Gate: all campaigns pass headless.

## Phase 8 - General availability

Once the migrated page has been stable (no P1 regressions for at least one minor release cycle):

1. Remove `_legacy_feature_flag` from routes.
2. Update the feature flag to `stable` (or remove it entirely).
3. Write an upgrade SQL script if the module shipped data that needs migration (via `module-add-migration`).
4. Document in CHANGELOG.

## Phase 9 - Legacy removal (next major)

In the module's next major version:
1. Delete the legacy `AdminXxxController`.
2. Delete the legacy `classes/Xxx.php` ObjectModel (if fully replaced by Doctrine entity).
3. Remove the feature-flag XML entry.
4. Bump `ps_versions_compliancy` minimum.

## Do

- Follow the phase order strictly. Each phase depends on the previous one's outputs.
- Commit after each phase so the history is bisectable and reviewable.
- Run `bin/console cache:clear` after every structural change to catch DI compilation errors early.
- Document the milestone strategy (single sprint vs. listing-first) in the first PR.
- Treat Behat (Phase 4) as a hard gate: do not start UI work if the domain layer has failing scenarios.

## Don't

- Don't skip the audit (Phase 0). Unknown fields and actions discovered mid-migration cause expensive rework.
- Don't build the form before the listing unless you have a specific reason (see milestone strategy).
- Don't remove the legacy controller until the new page is GA and has been stable for a full minor release.
- Don't merge without test gates: both Behat and Playwright must pass before shipping.
- Don't use `@trigger_error()` in the legacy controller as a deprecation mechanism - it causes log noise merchants cannot act on.

## Canonical examples

- Core Tax migration (simple: 3 fields, 1 grid, no tabs).
- Core Manufacturer migration (medium: 8 fields, sub-resource addresses, image upload).
- Core Employee migration (medium: 12 fields, profile permissions, tabs).
- [devdocs - Migration guide](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/) for the broader context.

## Related skills

- `migrate-module-full-rewrite` - pattern-by-pattern rewrite without the phased feature-flag delivery. Use that skill when the module has NO admin page (pure hooks/services) and just needs pattern upgrades.
- `migrate-admin-controller` - single-skill controller migration (invoked in Phase 5/6 above).
- `migrate-helperform-to-symfony-form` - form-level pattern swap (invoked conceptually in Phase 6).
- `migrate-helperlist-to-grid` - grid-level pattern swap (invoked conceptually in Phase 5).
- `migrate-objectmodel-to-cqrs` - CQRS pattern swap (invoked in Phase 2/3).
- `migrate-objectmodel-to-doctrine` - persistence swap (invoked in Phase 3).
- `module-add-feature-flag` - flag registration (Phase 1).
- `module-add-behat-tests` - Behat BDD tests (Phase 4).
- `module-add-playwright-tests` - Playwright E2E tests (Phase 7).
- `module-add-grid` - grid scaffolding (Phase 5).
- `module-add-admin-controller-modern` - controller scaffolding (Phase 5/6).
- `module-add-cqrs-command` - command scaffolding (Phase 2).
- `module-add-cqrs-bulk-command` - bulk command scaffolding (Phase 2).
- `module-add-cqrs-query` - query scaffolding (Phase 2).
