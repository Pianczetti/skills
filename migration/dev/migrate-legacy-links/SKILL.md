---
name: migrate-legacy-links
description: Replace legacy `index.php?controller=AdminXxx&token=...` links with Symfony route-based URL generation (`$this->generateUrl()`, `{{ path() }}`, `Routing.generate()`). Use when a module still constructs admin URLs using query-string controller names and security tokens.
---

## Requirements

- Identify all files generating legacy admin links (PHP, Twig, Smarty, JS).
- Confirm each target controller has a declared Symfony route with `_legacy_controller` and `_legacy_link`.
- If routes do not exist yet, create them first (see `module-add-symfony-route`).

## Steps

1. **Inventory legacy links.** Run the built-in linter or grep:

    ```bash
    bin/console prestashop:linter:legacy-link
    # or manually:
    grep -rn 'index\.php?controller=Admin' modules/mymodule/
    grep -rn 'token=' modules/mymodule/
    ```

2. **Ensure each target page has a route** in `config/routes.yml`:

    ```yaml
    mymodule_voucher_index:
      path: /modules/mymodule/vouchers
      methods: [GET]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\VoucherController::indexAction'
        _legacy_controller: 'AdminMymoduleVouchers'
        _legacy_link: 'AdminMymoduleVouchers'
    ```

3. **Replace PHP link generation.** In controllers or services:

    ```php
    // BEFORE
    $link = $this->context->link->getAdminLink('AdminMymoduleVouchers', true, [], ['id' => $id]);

    // AFTER
    $link = $this->generateUrl('mymodule_voucher_index', ['id' => $id]);
    ```

    In non-controller services, inject `Symfony\Component\Routing\RouterInterface`:

    ```php
    $link = $this->router->generate('mymodule_voucher_index', ['id' => $id]);
    ```

4. **Replace Twig link generation:**

    ```twig
    {# BEFORE #}
    <a href="{{ legacy_url('AdminMymoduleVouchers', {'id': item.id}) }}">

    {# AFTER #}
    <a href="{{ path('mymodule_voucher_index', {'id': item.id}) }}">
    ```

5. **Replace Smarty link generation:**

    ```smarty
    {* BEFORE *}
    <a href="{$link->getAdminLink('AdminMymoduleVouchers')}">

    {* AFTER - use the Symfony URL via a controller-assigned variable *}
    <a href="{$voucher_url}">
    ```

    Assign the URL in the controller before rendering:
    ```php
    $this->context->smarty->assign('voucher_url', $this->generateUrl('mymodule_voucher_index'));
    ```

6. **Replace JavaScript link generation.** Mark the route as exposed and use FOSJsRouting:

    ```yaml
    # config/routes.yml
    mymodule_voucher_index:
      options:
        expose: true
    ```

    ```js
    // BEFORE
    const url = baseAdminDir + 'index.php?controller=AdminMymoduleVouchers&token=' + token;

    // AFTER
    const url = Routing.generate('mymodule_voucher_index');
    ```

7. **Clear cache** and verify no legacy links remain:

    ```bash
    bin/console cache:clear
    bin/console prestashop:linter:legacy-link
    ```

## Do

- Declare `_legacy_link` in route defaults so `Link::getAdminLink()` falls through to the Symfony router automatically during partial migrations.
- Use route names consistently across PHP, Twig and JS to keep a single source of truth.

## Don't

- Don't hardcode `/admin-xxx/` paths; the admin directory is randomized at install.
- Don't pass `token=` manually; Symfony CSRF protection replaces the legacy token mechanism.
- Don't remove legacy controller names from route defaults until all link sources are migrated.

## Canonical examples

- [devdocs - Migration guide: controller and routing](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/controller-routing/).
- [devdocs - Modern controller and routing](https://devdocs.prestashop-project.org/9/development/architecture/modern/controller-routing/).
- Working sample: [`example-modules/demomoduleroutes`](https://github.com/PrestaShop/example-modules/tree/master/demomoduleroutes).

## Related skills

- `module-add-symfony-route` - declaring the route that replaces the legacy link.
- `migrate-admin-controller` - migrating the controller the link points to.
