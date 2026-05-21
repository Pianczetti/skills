---
name: module-add-grid
description: Add a Back Office listing as a modern PrestaShop Grid (GridDefinitionFactory + Doctrine QueryBuilder + Twig macro), wired through service tags and rendered with the core grid template. Use when the user wants a sortable, filterable, paginated table in the Back Office instead of a HelperList.
---

## Requirements

Ask the user:
* Grid id in `snake_case`, prefixed with the module short code (e.g. `mymod_voucher`). The constant `GRID_ID` and the form filter share this id.
* Data source: a Doctrine DBAL `Connection` query against your module tables (the standard path), or a custom data factory if you need post-processing.
* Columns. For each column collect: id (snake_case), label, the SQL alias that produces it, the column type (`DataColumn`, `BadgeColumn`, `DateTimeColumn`, `ToggleColumn`, `ActionColumn`, ...).
* Filters - which columns are filterable, with which form types (`TextType`, `NumberType`, `DateRangeType`, `YesAndNoChoiceType`, ...).
* Bulk actions (delete, enable, disable, custom).
* Row actions (edit, delete, link).
* Whether rows are reorderable (positionable) - if yes, add a `PositionColumn` and a position-update AJAX route.
* Whether any boolean column should toggle inline - if yes, you need a dedicated AJAX route returning JSON.

## Steps

1. Create the GridDefinitionFactory under `src/Grid/Definition/Factory/<Name>GridDefinitionFactory.php` extending `AbstractGridDefinitionFactory`. Order the columns strictly: `BulkActionColumn` first, then `PositionColumn` (only when positionable), then data columns, then `ActionColumn` last. Every column id is snake_case AND matches the SQL alias produced by the query builder:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Grid\Definition\Factory;

    use PrestaShop\PrestaShop\Core\Grid\Action\Bulk\BulkActionCollection;
    use PrestaShop\PrestaShop\Core\Grid\Action\Bulk\Type\ButtonBulkAction;
    use PrestaShop\PrestaShop\Core\Grid\Action\GridActionCollection;
    use PrestaShop\PrestaShop\Core\Grid\Action\Row\RowActionCollection;
    use PrestaShop\PrestaShop\Core\Grid\Action\Row\Type\LinkRowAction;
    use PrestaShop\PrestaShop\Core\Grid\Column\ColumnCollection;
    use PrestaShop\PrestaShop\Core\Grid\Column\Type\Common\ActionColumn;
    use PrestaShop\PrestaShop\Core\Grid\Column\Type\Common\BulkActionColumn;
    use PrestaShop\PrestaShop\Core\Grid\Column\Type\Common\DataColumn;
    use PrestaShop\PrestaShop\Core\Grid\Column\Type\Common\DateTimeColumn;
    use PrestaShop\PrestaShop\Core\Grid\Column\Type\Common\ToggleColumn;
    use PrestaShop\PrestaShop\Core\Grid\Definition\Factory\AbstractGridDefinitionFactory;
    use PrestaShop\PrestaShop\Core\Grid\Filter\Filter;
    use PrestaShop\PrestaShop\Core\Grid\Filter\FilterCollection;
    use PrestaShopBundle\Form\Admin\Type\SearchAndResetType;
    use PrestaShopBundle\Form\Admin\Type\YesAndNoChoiceType;
    use Symfony\Component\Form\Extension\Core\Type\TextType;

    final class VoucherGridDefinitionFactory extends AbstractGridDefinitionFactory
    {
        public const GRID_ID = 'mymod_voucher';

        protected function getId(): string
        {
            return self::GRID_ID;
        }

        protected function getName(): string
        {
            return $this->trans('Vouchers', [], 'Modules.Mymodule.Admin');
        }

        protected function getColumns(): ColumnCollection
        {
            return (new ColumnCollection())
                ->add((new BulkActionColumn('bulk'))
                    ->setOptions(['bulk_field' => 'id_voucher'])
                )
                ->add((new DataColumn('id_voucher'))
                    ->setName($this->trans('ID', [], 'Admin.Global'))
                    ->setOptions(['field' => 'id_voucher'])
                )
                ->add((new DataColumn('code'))
                    ->setName($this->trans('Code', [], 'Modules.Mymodule.Admin'))
                    ->setOptions(['field' => 'code'])
                )
                ->add((new DataColumn('discount_percent'))
                    ->setName($this->trans('Discount %', [], 'Modules.Mymodule.Admin'))
                    ->setOptions(['field' => 'discount_percent'])
                )
                ->add((new DateTimeColumn('expires_at'))
                    ->setName($this->trans('Expires', [], 'Modules.Mymodule.Admin'))
                    ->setOptions(['field' => 'expires_at', 'format' => 'Y-m-d H:i'])
                )
                ->add((new ToggleColumn('active'))
                    ->setName($this->trans('Enabled', [], 'Admin.Global'))
                    ->setOptions([
                        'field' => 'active',
                        'primary_field' => 'id_voucher',
                        'route' => 'mymodule_voucher_toggle_active',
                        'route_param_name' => 'voucherId',
                    ])
                )
                ->add((new ActionColumn('actions'))
                    ->setName($this->trans('Actions', [], 'Admin.Global'))
                    ->setOptions([
                        'actions' => (new RowActionCollection())
                            ->add((new LinkRowAction('edit'))
                                ->setIcon('edit')
                                ->setOptions([
                                    'route' => 'mymodule_voucher_edit',
                                    'route_param_name' => 'voucherId',
                                    'route_param_field' => 'id_voucher',
                                ])
                            ),
                    ])
                );
        }

        protected function getFilters(): FilterCollection
        {
            return (new FilterCollection())
                ->add((new Filter('id_voucher', TextType::class))->setTypeOptions(['required' => false])->setAssociatedColumn('id_voucher'))
                ->add((new Filter('code', TextType::class))->setTypeOptions(['required' => false])->setAssociatedColumn('code'))
                ->add((new Filter('active', YesAndNoChoiceType::class))->setAssociatedColumn('active'))
                ->add((new Filter('actions', SearchAndResetType::class))
                    ->setTypeOptions(['reset_route' => 'admin_common_reset_search_by_filter_id', 'reset_route_params' => ['filterId' => self::GRID_ID], 'redirect_route' => 'mymodule_voucher_index'])
                    ->setAssociatedColumn('actions')
                );
        }

        protected function getGridActions(): GridActionCollection
        {
            return new GridActionCollection();
        }

        protected function getBulkActions(): BulkActionCollection
        {
            return (new BulkActionCollection())
                ->add((new ButtonBulkAction('delete_selection'))
                    ->setName($this->trans('Delete selected', [], 'Admin.Actions'))
                    ->setOptions(['confirm_message' => $this->trans('Delete selected vouchers?', [], 'Modules.Mymodule.Admin')])
                );
        }
    }
    ```

2. Create the QueryBuilder under `src/Grid/Query/<Name>QueryBuilder.php` extending `AbstractDoctrineQueryBuilder`. Every selected column MUST be aliased to match the snake_case column id from step 1. `getCountQueryBuilder()` returns the SAME base query as `getSearchQueryBuilder()` minus `ORDER BY`, `LIMIT` and `OFFSET`. Always parameterise with `setParameter()`:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Grid\Query;

    use Doctrine\DBAL\Connection;
    use Doctrine\DBAL\Query\QueryBuilder;
    use PrestaShop\PrestaShop\Core\Grid\Query\AbstractDoctrineQueryBuilder;
    use PrestaShop\PrestaShop\Core\Grid\Query\DoctrineSearchCriteriaApplicatorInterface;
    use PrestaShop\PrestaShop\Core\Grid\Search\SearchCriteriaInterface;

    final class VoucherQueryBuilder extends AbstractDoctrineQueryBuilder
    {
        public function __construct(
            Connection $connection,
            string $dbPrefix,
            private readonly DoctrineSearchCriteriaApplicatorInterface $criteriaApplicator,
        ) {
            parent::__construct($connection, $dbPrefix);
        }

        public function getSearchQueryBuilder(SearchCriteriaInterface $searchCriteria): QueryBuilder
        {
            $qb = $this->getBaseQuery($searchCriteria->getFilters())
                ->select('v.id_voucher AS id_voucher')
                ->addSelect('v.code AS code')
                ->addSelect('v.discount_percent AS discount_percent')
                ->addSelect('v.expires_at AS expires_at')
                ->addSelect('v.active AS active')
            ;

            $qb->orderBy(
                $searchCriteria->getOrderBy() ?: 'id_voucher',
                $searchCriteria->getOrderWay() ?: 'ASC'
            );

            $this->criteriaApplicator
                ->applyPagination($searchCriteria, $qb)
                ->applyDeterministicSorting($searchCriteria, $qb, 'v', 'id_voucher')
            ;

            return $qb;
        }

        public function getCountQueryBuilder(SearchCriteriaInterface $searchCriteria): QueryBuilder
        {
            // No ORDER BY, no LIMIT, no OFFSET. Pagination MUST NOT leak into the count.
            return $this->getBaseQuery($searchCriteria->getFilters())->select('COUNT(v.id_voucher)');
        }

        private function getBaseQuery(array $filters): QueryBuilder
        {
            $qb = $this->connection->createQueryBuilder()
                ->from($this->dbPrefix . 'mymod_voucher', 'v');

            if (isset($filters['id_voucher']) && '' !== $filters['id_voucher']) {
                $qb->andWhere('v.id_voucher = :id_voucher')
                   ->setParameter('id_voucher', (int) $filters['id_voucher']);
            }
            if (!empty($filters['code'])) {
                $qb->andWhere('v.code LIKE :code')
                   ->setParameter('code', '%' . $filters['code'] . '%');
            }
            if (isset($filters['active']) && '' !== $filters['active']) {
                $qb->andWhere('v.active = :active')
                   ->setParameter('active', (bool) $filters['active'], \PDO::PARAM_BOOL);
            }

            return $qb;
        }
    }
    ```

3. Register the factory, query builder, data factory and grid factory in `config/services.yml`. The two `parent:` services from core inherit the right tags (`prestashop.core.grid.definition.factory` for the factory, `prestashop.core.grid.query_builder` for the query builder), so you do not have to add them manually:

    ```yaml
    services:
      MyVendor\Mymodule\Grid\Definition\Factory\VoucherGridDefinitionFactory:
        parent: 'prestashop.core.grid.definition.factory.abstract_grid_definition'
        public: true

      MyVendor\Mymodule\Grid\Query\VoucherQueryBuilder:
        parent: 'prestashop.core.grid.abstract_query_builder'
        autowire: true

      mymodule.grid.data_provider.vouchers:
        class: '%prestashop.core.grid.data.factory.doctrine_grid_data_factory%'
        arguments:
          - '@MyVendor\Mymodule\Grid\Query\VoucherQueryBuilder'
          - '@prestashop.core.hook.dispatcher'
          - '@prestashop.core.grid.query.doctrine_query_parser'
          - !php/const MyVendor\Mymodule\Grid\Definition\Factory\VoucherGridDefinitionFactory::GRID_ID

      mymodule.grid.factory.vouchers:
        class: 'PrestaShop\PrestaShop\Core\Grid\GridFactory'
        public: true
        arguments:
          - '@MyVendor\Mymodule\Grid\Definition\Factory\VoucherGridDefinitionFactory'
          - '@mymodule.grid.data_provider.vouchers'
          - '@prestashop.core.grid.filter.form_factory'
          - '@prestashop.core.hook.dispatcher'
    ```

4. Create the Filters class under `src/Grid/Filters/<Name>Filters.php` so the controller can typehint the search criteria:

    ```php
    <?php
    namespace MyVendor\Mymodule\Grid\Filters;

    use PrestaShop\PrestaShop\Core\Search\Filters;

    final class VoucherFilters extends Filters
    {
        public static function getDefaults(): array
        {
            return [
                'limit' => 25,
                'offset' => 0,
                'orderBy' => 'id_voucher',
                'sortOrder' => 'ASC',
                'filters' => [],
            ];
        }
    }
    ```

5. Wire the controller. Use `#[Autowire(service: 'mymodule.grid.factory.vouchers')]` to grab the grid factory and `presentGrid()` to hand the grid to Twig:

    ```php
    <?php
    namespace MyVendor\Mymodule\Controller\Admin;

    use MyVendor\Mymodule\Grid\Definition\Factory\VoucherGridDefinitionFactory;
    use MyVendor\Mymodule\Grid\Filters\VoucherFilters;
    use PrestaShop\PrestaShop\Core\Grid\GridFactoryInterface;
    use PrestaShopBundle\Controller\Admin\PrestaShopAdminController;
    use Symfony\Component\DependencyInjection\Attribute\Autowire;
    use Symfony\Component\HttpFoundation\RedirectResponse;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\HttpFoundation\Response;

    class VoucherController extends PrestaShopAdminController
    {
        public function indexAction(
            VoucherFilters $filters,
            #[Autowire(service: 'mymodule.grid.factory.vouchers')]
            GridFactoryInterface $voucherGridFactory,
        ): Response {
            return $this->render('@Modules/mymodule/views/templates/admin/voucher/index.html.twig', [
                'voucherGrid' => $this->presentGrid($voucherGridFactory->getGrid($filters)),
            ]);
        }

        public function searchAction(
            Request $request,
            VoucherGridDefinitionFactory $definition,
        ): RedirectResponse {
            return $this->buildSearchResponse(
                $definition,
                $request,
                VoucherGridDefinitionFactory::GRID_ID,
                'mymodule_voucher_index'
            );
        }
    }
    ```

6. Render the grid in the Twig template by including the core grid macro. No custom HTML needed:

    ```twig
    {# views/templates/admin/voucher/index.html.twig #}
    {% extends '@PrestaShop/Admin/layout.html.twig' %}
    {% block content %}
      {% embed '@PrestaShop/Admin/Common/Grid/grid_panel.html.twig' with {'grid': voucherGrid} %}{% endembed %}
    {% endblock %}
    ```

7. Add the routes in `config/routes.yml`. At minimum an index and a search (POST on the same path):

    ```yaml
    mymodule_voucher_index:
      path: /modules/mymodule/vouchers
      methods: [GET]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\VoucherController::indexAction'
        _legacy_controller: 'AdminMymoduleVouchers'
        _legacy_link: 'AdminMymoduleVouchers'

    mymodule_voucher_search:
      path: /modules/mymodule/vouchers
      methods: [POST]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\VoucherController::searchAction'
        _legacy_controller: 'AdminMymoduleVouchers'
    ```

   For each `ToggleColumn` add a `POST /...{id}/toggle-<field>` route whose action returns a JSON `{status, message}` payload (the Grid front-end script expects this shape) and dispatches a `Toggle<X>Command` via the bus. Same pattern for bulk-action endpoints. Build them as standard CQRS commands - see `module-add-cqrs-command`.

## Do

- Order columns strictly: `BulkActionColumn` first, then `PositionColumn` (only if positionable), then data columns, then `ActionColumn` last. Reviewers and the BO Twig macros assume that order.
- Make every column id snake_case AND identical to the SQL alias from the query builder (`v.id_voucher AS id_voucher`). The `field:` option in the column points at this alias.
- Always parameterise filter values with `setParameter()` and a typed binding (`PDO::PARAM_BOOL`, `PARAM_INT`). String concatenation in SQL is a SQL-injection vector.
- Use `parent: prestashop.core.grid.definition.factory.abstract_grid_definition` and `parent: prestashop.core.grid.abstract_query_builder` for service wiring; they inject the right constructor args and apply the required tags automatically.

## Don't

- Don't use `HelperList` for new BO listings. Grids expose filtering, bulk actions, hooks and the modern column types; HelperList is the legacy path.
- Don't put `ORDER BY`, `LIMIT` or `OFFSET` inside `getCountQueryBuilder()`. Pagination MUST NOT leak into the count or the grid will report wrong totals.
- Don't concatenate user input into SQL (`->andWhere("code = '$code'")`). Use `:code` + `setParameter('code', $code)` even for "trusted" filters.
- Don't return Doctrine entities or `ObjectModel` objects from the query builder; the Grid expects flat associative rows keyed by the SQL alias.
- Don't omit `_legacy_controller` from the routes - the BO permission system and the bulk-action submit URL both need it.

## Canonical examples

- [devdocs - Grid component](https://devdocs.prestashop-project.org/9/development/components/grid/).
- Working sample wired like a module: [`example-modules/demo_grid`](https://github.com/PrestaShop/example-modules/tree/master/demo_grid). Hook-based extension: [`example-modules/demoextendgrid`](https://github.com/PrestaShop/example-modules/tree/master/demoextendgrid).
- Reference factory and query builder: [`CustomerGridDefinitionFactory`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Grid/Definition/Factory/CustomerGridDefinitionFactory.php), [`CustomerQueryBuilder`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Grid/Query/CustomerQueryBuilder.php).
- Abstract bases: [`AbstractGridDefinitionFactory`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Grid/Definition/Factory/AbstractGridDefinitionFactory.php), [`AbstractDoctrineQueryBuilder`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Grid/Query/AbstractDoctrineQueryBuilder.php).

## Related skills

- `module-add-cqrs-command` for the toggle / bulk-action handlers dispatched from the JSON routes above.
- `module-add-symfony-route` for the route declaration cheatsheet (`_legacy_controller`, `_legacy_link`).
