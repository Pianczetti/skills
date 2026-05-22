---
name: migrate-helperlist-to-grid
description: Replace a HelperList-based Back Office listing with the modern Grid component (GridDefinitionFactory + Doctrine QueryBuilder + filters + actions + Twig macro). Use when a module renders a table via HelperList and must migrate to the PS 9 Grid architecture.
---

## Requirements

- Identify the legacy `$this->fields_list` array and the `getList()` / `renderList()` calls.
- Map each column to a Grid column type (`DataColumn`, `DateTimeColumn`, `ToggleColumn`, `ActionColumn`).
- List all bulk actions and row actions currently wired through `$this->bulk_actions` and `$this->actions`.
- Confirm the Grid ID (snake_case, module-prefixed, e.g. `mymod_voucher`).

## Steps

1. **Extract the legacy column definitions.** Typical legacy pattern:

    ```php
    // LEGACY - to be removed
    $this->fields_list = [
        'id_voucher' => ['title' => $this->l('ID'), 'type' => 'int'],
        'code'       => ['title' => $this->l('Code'), 'type' => 'text'],
        'active'     => ['title' => $this->l('Active'), 'type' => 'bool', 'active' => 'status'],
    ];
    $this->bulk_actions = ['delete' => ['text' => $this->l('Delete selected')]];
    ```

2. **Create the GridDefinitionFactory** (`src/Grid/Definition/Factory/<Name>GridDefinitionFactory.php`). Map each legacy column to the modern type:

    | Legacy `type` | Modern Column |
    |---|---|
    | `int`, `text` | `DataColumn` |
    | `bool` + `active` | `ToggleColumn` |
    | `date` | `DateTimeColumn` |
    | row actions | `ActionColumn` with `RowActionCollection` |

    ```php
    use PrestaShop\PrestaShop\Core\Grid\Definition\Factory\AbstractGridDefinitionFactory;

    final class VoucherGridDefinitionFactory extends AbstractGridDefinitionFactory
    {
        public const GRID_ID = 'mymod_voucher';
        protected function getId(): string { return self::GRID_ID; }
        // columns, filters, bulk actions - see module-add-grid for full code
    }
    ```

3. **Create the QueryBuilder** (`src/Grid/Query/<Name>QueryBuilder.php`) extending `AbstractDoctrineQueryBuilder`. Replace the legacy `$this->_listSql` / `ObjectModel::getList()` with a parameterised DBAL query. Every selected alias must match the column id exactly.

4. **Wire services** in `config/services.yml` using `parent: prestashop.core.grid.definition.factory.abstract_grid_definition` and `parent: prestashop.core.grid.abstract_query_builder`.

5. **Create the Filters class** (`src/Grid/Filters/<Name>Filters.php`) with default pagination and sort order.

6. **Update the controller** to inject the GridFactory and call `$this->presentGrid()`:

    ```php
    public function indexAction(
        VoucherFilters $filters,
        #[Autowire(service: 'mymodule.grid.factory.vouchers')]
        GridFactoryInterface $gridFactory,
    ): Response {
        return $this->render('@Modules/mymodule/views/templates/admin/voucher/index.html.twig', [
            'grid' => $this->presentGrid($gridFactory->getGrid($filters)),
        ]);
    }
    ```

7. **Render with the core Twig macro** - no custom table HTML:

    ```twig
    {% extends '@PrestaShop/Admin/layout.html.twig' %}
    {% block content %}
      {% embed '@PrestaShop/Admin/Common/Grid/grid_panel.html.twig' with {'grid': grid} %}{% endembed %}
    {% endblock %}
    ```

8. **Delete legacy code**: remove `$this->fields_list`, `$this->bulk_actions`, `getList()`, `renderList()`, and any `initToolbar()` overrides.

## Do

- Keep column ids in snake_case and identical to the SQL aliases from the query builder.
- Always parameterise filter values with `setParameter()`.
- Use `SearchAndResetType` as the last filter so the "Reset" button targets the right route.

## Don't

- Don't put `ORDER BY`, `LIMIT`, or `OFFSET` in `getCountQueryBuilder()`.
- Don't concatenate user input into SQL; use `:param` + `setParameter()`.
- Don't omit `_legacy_controller` from routes; the permission system requires it.

## Canonical examples

- [devdocs - Grid component](https://devdocs.prestashop-project.org/9/development/components/grid/)
- [devdocs - Migration guide](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/)

## Related skills

- `module-add-grid` - full recipe for building a Grid from scratch.
- `migrate-admin-controller` - migrating the controller that hosts the listing.
