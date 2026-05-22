---
name: migrate-helperlist-to-grid
description: Replace a legacy HelperList ($fields_list, getList(), renderList()) with a modern Grid (GridDefinitionFactory + Doctrine QueryBuilder + Filters + Twig macro). Use when a module renders a Back Office listing through HelperList and needs the modern filter/bulk/toggle pipeline.
---

## Requirements

- Module name and vendor namespace.
- Grid id in `snake_case` (e.g. `mymod_voucher`).
- Columns from the existing `$fields_list` array with types and labels.
- Filterable columns and their form types.
- Bulk actions and row actions currently wired through the legacy list.
- SQL table and column names backing the list.

## Steps

1. Create `src/Grid/Definition/Factory/<Name>GridDefinitionFactory.php` extending `AbstractGridDefinitionFactory`. Order columns: `BulkActionColumn` first, data columns, `ActionColumn` last. Every column id is snake_case matching the SQL alias:

    ```php
    namespace MyVendor\Mymodule\Grid\Definition\Factory;

    use PrestaShop\PrestaShop\Core\Grid\Definition\Factory\AbstractGridDefinitionFactory;
    use PrestaShop\PrestaShop\Core\Grid\Column\ColumnCollection;
    use PrestaShop\PrestaShop\Core\Grid\Column\Type\Common\BulkActionColumn;
    use PrestaShop\PrestaShop\Core\Grid\Column\Type\Common\DataColumn;
    use PrestaShop\PrestaShop\Core\Grid\Column\Type\Common\ActionColumn;

    final class ItemGridDefinitionFactory extends AbstractGridDefinitionFactory
    {
        public const GRID_ID = 'mymod_item';

        protected function getId(): string { return self::GRID_ID; }
        protected function getName(): string { return $this->trans('Items', [], 'Modules.Mymodule.Admin'); }

        protected function getColumns(): ColumnCollection
        {
            return (new ColumnCollection())
                ->add((new BulkActionColumn('bulk'))->setOptions(['bulk_field' => 'id_item']))
                ->add((new DataColumn('id_item'))->setName($this->trans('ID', [], 'Admin.Global'))->setOptions(['field' => 'id_item']))
                ->add((new ActionColumn('actions'))->setName($this->trans('Actions', [], 'Admin.Global')));
        }
    }
    ```

2. Create `src/Grid/Query/<Name>QueryBuilder.php` extending `AbstractDoctrineQueryBuilder`. `getCountQueryBuilder()` returns the same base query without ORDER BY, LIMIT or OFFSET. Always use `setParameter()`:

    ```php
    namespace MyVendor\Mymodule\Grid\Query;

    use PrestaShop\PrestaShop\Core\Grid\Query\AbstractDoctrineQueryBuilder;
    use PrestaShop\PrestaShop\Core\Grid\Query\DoctrineSearchCriteriaApplicatorInterface;
    use PrestaShop\PrestaShop\Core\Grid\Search\SearchCriteriaInterface;

    final class ItemQueryBuilder extends AbstractDoctrineQueryBuilder
    {
        public function __construct(
            \Doctrine\DBAL\Connection $connection,
            string $dbPrefix,
            private readonly DoctrineSearchCriteriaApplicatorInterface $criteriaApplicator,
        ) {
            parent::__construct($connection, $dbPrefix);
        }

        public function getSearchQueryBuilder(SearchCriteriaInterface $searchCriteria): \Doctrine\DBAL\Query\QueryBuilder
        {
            $qb = $this->getBaseQuery($searchCriteria->getFilters())
                ->select('i.id_item AS id_item');
            $this->criteriaApplicator->applyPagination($searchCriteria, $qb);
            return $qb;
        }

        public function getCountQueryBuilder(SearchCriteriaInterface $searchCriteria): \Doctrine\DBAL\Query\QueryBuilder
        {
            return $this->getBaseQuery($searchCriteria->getFilters())->select('COUNT(i.id_item)');
        }

        private function getBaseQuery(array $filters): \Doctrine\DBAL\Query\QueryBuilder
        {
            return $this->connection->createQueryBuilder()->from($this->dbPrefix . 'mymod_item', 'i');
        }
    }
    ```

3. Create `src/Grid/Filters/<Name>Filters.php`:

    ```php
    namespace MyVendor\Mymodule\Grid\Filters;

    use PrestaShop\PrestaShop\Core\Search\Filters;

    final class ItemFilters extends Filters
    {
        public static function getDefaults(): array
        {
            return ['limit' => 25, 'offset' => 0, 'orderBy' => 'id_item', 'sortOrder' => 'ASC', 'filters' => []];
        }
    }
    ```

4. Register services in `config/services.yml` using the core parent services:

    ```yaml
    services:
      MyVendor\Mymodule\Grid\Definition\Factory\ItemGridDefinitionFactory:
        parent: 'prestashop.core.grid.definition.factory.abstract_grid_definition'
        public: true
      MyVendor\Mymodule\Grid\Query\ItemQueryBuilder:
        parent: 'prestashop.core.grid.abstract_query_builder'
        autowire: true
    ```

5. Render in the controller with `presentGrid()` and include `@PrestaShop/Admin/Common/Grid/grid_panel.html.twig` in the template.

6. Remove the legacy `$fields_list` array, `getList()`, `renderList()`, and any `HelperList` instantiation from the old controller.

7. Validate by loading the listing in the BO: sort, filter, paginate, execute a bulk action.

## Do

- Match SQL aliases to column ids byte-for-byte (e.g. `i.id_item AS id_item` maps to column id `id_item`).
- Use `parent: prestashop.core.grid.abstract_query_builder` which injects Connection, dbPrefix, and the criteria applicator.

## Don't

- Don't put ORDER BY, LIMIT or OFFSET in `getCountQueryBuilder()`.
- Don't concatenate user input into SQL; always `setParameter()`.
- Don't return Doctrine entities from the query builder; the Grid expects flat associative rows.

## Related skills

- `module-add-grid` - greenfield Grid creation.
- `migrate-admin-controller` - the controller this grid plugs into.
- `migrate-objectmodel-to-cqrs` - replacing CRUD actions triggered from bulk/row actions.
