---
name: migrate-legacy-ajax
description: Replace legacy AJAX patterns (`ajaxProcess*` methods in AdminController, `index.php?ajax=1&action=...`) with a dedicated Symfony route returning `JsonResponse`. Use when a module uses the legacy AJAX dispatch mechanism and must adopt modern routing.
---

## Requirements

- Identify all `ajaxProcess*` methods in legacy controllers.
- Map each AJAX action to a dedicated route name and HTTP method.
- Confirm caller JS files that hit these endpoints.

## Steps

1. **Identify the legacy pattern:**

    ```php
    // Legacy AdminController with magic AJAX dispatch
    class AdminMymoduleAjaxController extends ModuleAdminController
    {
        public function ajaxProcessSearchProducts()
        {
            $query = Tools::getValue('q');
            $results = Product::searchByName($this->context->language->id, $query);
            die(json_encode(['products' => $results]));
        }
    }
    ```

    Called from JS:
    ```js
    $.ajax({
      url: baseAdminDir + 'index.php?controller=AdminMymoduleAjax&ajax=1&action=searchProducts&token=' + token,
      success: function(data) { /* ... */ }
    });
    ```

2. **Create a modern controller action** returning `JsonResponse`:

    ```php
    <?php
    namespace MyVendor\Mymodule\Controller\Admin;

    use PrestaShopBundle\Controller\Admin\PrestaShopAdminController;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;
    use Symfony\Component\Security\Http\Attribute\IsGranted;

    class ProductSearchController extends PrestaShopAdminController
    {
        #[IsGranted('read', subject: '_legacy_controller')]
        public function searchAction(Request $request): JsonResponse
        {
            $query = $request->query->get('q', '');
            // Use a service/repository instead of Product::searchByName
            $results = $this->productRepository->searchByName($query);

            return $this->json(['products' => $results]);
        }
    }
    ```

3. **Declare the route** in `config/routes.yml`:

    ```yaml
    mymodule_product_search:
      path: /modules/mymodule/products/search
      methods: [GET]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\ProductSearchController::searchAction'
        _legacy_controller: 'AdminMymoduleAjax'
      options:
        expose: true
    ```

4. **Update the JS caller** to use `fetch` and the exposed route:

    ```js
    // BEFORE
    $.ajax({ url: baseAdminDir + 'index.php?controller=AdminMymoduleAjax&ajax=1&action=searchProducts&token=' + token });

    // AFTER
    const url = Routing.generate('mymodule_product_search', { q: query });
    const response = await fetch(url, {
      headers: { 'X-Requested-With': 'XMLHttpRequest' }
    });
    const data = await response.json();
    ```

5. **Remove the legacy controller** once all AJAX actions are migrated. If other non-AJAX actions remain, keep the controller but delete the `ajaxProcess*` methods.

6. **Clear cache:**

    ```bash
    bin/console cache:clear
    ```

## Do

- Return typed `JsonResponse` with appropriate HTTP status codes (200, 400, 404, 500).
- Use `#[IsGranted]` for permission checks instead of manual `if (!$this->access(...))`.
- Set `options: { expose: true }` on routes accessed from JS so FOSJsRouting exposes them.

## Don't

- Don't use `die(json_encode(...))` or `echo` + `exit` in modern controllers.
- Don't rely on `Tools::getValue()` in Symfony controllers; use `$request->query->get()` or `$request->request->get()`.
- Don't keep both legacy `ajaxProcess*` and the new route active simultaneously; they can conflict.

## Canonical examples

- [devdocs - Migration guide: controller and routing](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/controller-routing/)

## Related skills

- `module-add-admin-controller-modern` - full modern controller recipe.
- `module-add-symfony-route` - route declaration details.
- `migrate-admin-controller` - migrating the full controller (not just AJAX).
