---
name: migrate-legacy-ajax
description: Replace ajaxProcess* methods in a legacy AdminController (triggered via index.php?ajax=1&action=xxx) with dedicated Symfony routes returning JsonResponse. Use when a module handles AJAX through the legacy dispatch mechanism and needs proper REST endpoints.
---

## Requirements

- Module name and vendor namespace.
- List of `ajaxProcess*` methods and their expected request/response shapes.
- Whether the AJAX actions mutate state (POST) or only read (GET).
- JavaScript files that call the legacy AJAX endpoints.

## Steps

1. Identify all `ajaxProcess*` methods in the legacy controller:

    ```bash
    grep -n 'ajaxProcess' modules/mymodule/controllers/admin/AdminMymoduleController.php
    ```

2. For each `ajaxProcessXxx` method, create a dedicated action in the modern Symfony controller that returns `JsonResponse`:

    ```php
    namespace MyVendor\Mymodule\Controller\Admin;

    use PrestaShopBundle\Controller\Admin\PrestaShopAdminController;
    use Symfony\Component\HttpFoundation\JsonResponse;
    use Symfony\Component\HttpFoundation\Request;

    class StuffAjaxController extends PrestaShopAdminController
    {
        public function toggleStatusAction(Request $request): JsonResponse
        {
            $id = (int) $request->request->get('id');
            // dispatch CQRS command...
            return new JsonResponse(['status' => true, 'message' => 'Status toggled']);
        }
    }
    ```

3. Declare the route in `config/routes.yml` with explicit HTTP method:

    ```yaml
    mymodule_stuff_toggle_status:
      path: /modules/mymodule/stuff/{id}/toggle-status
      methods: [POST]
      defaults:
        _controller: 'MyVendor\Mymodule\Controller\Admin\StuffAjaxController::toggleStatusAction'
        _legacy_controller: 'AdminMymoduleStuff'
      requirements:
        id: '\d+'
    ```

4. Update the JavaScript to call the new endpoint via `fetch()` instead of `$.ajax()` against `index.php?ajax=1`:

    ```javascript
    // Before
    $.ajax({
      url: currentIndex + '&ajax=1&action=toggleStatus&id=' + id + '&token=' + token,
      success: function(data) { /* ... */ }
    });

    // After
    const url = document.querySelector('[data-toggle-url]').dataset.toggleUrl;
    fetch(url.replace('{id}', id), {
      method: 'POST',
      headers: { 'X-Requested-With': 'XMLHttpRequest' },
    }).then(r => r.json()).then(data => { /* ... */ });
    ```

5. Pass the route URL to the template via a `data-*` attribute:

    ```twig
    <div data-toggle-url="{{ path('mymodule_stuff_toggle_status', {id: '__ID__'}) }}"></div>
    ```

6. Remove the `ajaxProcess*` methods from the legacy controller file (or delete the entire legacy controller if all actions have been migrated).

7. Test each AJAX call in the browser DevTools Network panel: confirm 200 responses with JSON payloads and no 302 redirects to the login page.

## Do

- Return `JsonResponse` with `status` and `message` keys; the Grid JS expects this shape for toggle/bulk actions.
- Use explicit `methods: [POST]` or `[GET]` per route; never allow both unless the semantics differ.
- Protect mutating endpoints with `#[IsGranted('update', subject: '_legacy_controller')]`.

## Don't

- Don't keep `index.php?ajax=1` URLs in JavaScript; they depend on legacy token generation.
- Don't return HTML fragments from AJAX endpoints; return JSON and let the client update the DOM.
- Don't skip CSRF validation on POST endpoints; use `$this->isCsrfTokenValid()` or rely on the BO token cookie.

## Related skills

- `migrate-admin-controller` - the main controller migration.
- `migrate-legacy-links` - replacing URL patterns.
- `module-add-cqrs-command` - the CQRS commands dispatched from AJAX actions.
