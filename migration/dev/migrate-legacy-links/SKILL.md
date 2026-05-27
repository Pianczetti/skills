---
name: migrate-legacy-links
description: Replace hardcoded index.php?controller=AdminXxx&token=... URLs and getAdminLink() calls with Symfony route-based URL generation. Use when a module contains legacy admin link patterns that break under the modern routing system.
---

## Requirements

- Module name.
- List of legacy controller names referenced in the module code and templates.
- Corresponding Symfony route names (from `config/routes.yml`).

## Steps

1. Run the legacy-link linter to discover all occurrences:

    ```bash
    bin/console prestashop:linter:legacy-link modules/mymodule
    ```

2. Grep for additional patterns the linter may miss:

    ```bash
    grep -rn 'index.php?controller=' modules/mymodule/
    grep -rn 'getAdminLink(' modules/mymodule/
    grep -rn "AdminLink(" modules/mymodule/
    ```

3. In PHP controllers and services, replace `$this->context->link->getAdminLink('AdminXxx')` with `$this->generateUrl('route_name')` or inject `Symfony\Component\Routing\RouterInterface`:

    ```php
    // Before
    $url = $this->context->link->getAdminLink('AdminMymoduleStuff');

    // After (in a Symfony controller)
    $url = $this->generateUrl('mymodule_stuff_index');

    // After (in a service)
    $url = $this->router->generate('mymodule_stuff_index');
    ```

4. In Twig templates, replace link helper calls with `path()`:

    ```twig
    {# Before #}
    <a href="{{ link.getAdminLink('AdminMymoduleStuff') }}">Stuff</a>

    {# After #}
    <a href="{{ path('mymodule_stuff_index') }}">Stuff</a>
    ```

5. In Smarty templates (`.tpl`), replace inline PHP link generation with the `{url}` helper or pass the URL from the controller:

    ```smarty
    {* Before *}
    <a href="{$link->getAdminLink('AdminMymoduleStuff')}">Stuff</a>

    {* After - pass from controller *}
    <a href="{$stuff_url}">Stuff</a>
    ```

6. For JavaScript files, replace hardcoded `index.php?controller=...&ajax=1` URLs with a route URL passed via a `data-*` attribute or a global JS variable set in the template:

    ```twig
    <div data-ajax-url="{{ path('mymodule_stuff_ajax') }}"></div>
    ```

7. Re-run the linter to confirm zero remaining hits:

    ```bash
    bin/console prestashop:linter:legacy-link modules/mymodule
    ```

## Do

- Ensure every route referenced has `_legacy_link` set in `config/routes.yml` for backward compatibility with `getAdminLink()` consumers outside the module.
- Pass route parameters explicitly: `$this->generateUrl('mymodule_stuff_edit', ['id' => $id])`.

## Don't

- Don't keep `index.php?controller=...&token=` URLs anywhere; PS 9 drops legacy token validation for Symfony-routed pages.
- Don't use `Tools::getAdminTokenLite()` for Symfony routes; CSRF is handled by the framework.
- Don't generate URLs by string concatenation; always use the router.

## Related skills

- `module-add-symfony-route` - declaring routes in `config/routes.yml`.
- `migrate-admin-controller` - the controller migration that defines the routes.
- `migrate-legacy-tabs-to-routes` - Tab entries that reference routes.
