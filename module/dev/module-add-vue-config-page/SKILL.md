---
name: module-add-vue-config-page
description: Build a Vue 3 SPA configuration page for a PrestaShop 9 module, bundled with Vite, mounted from a modern Symfony admin controller, and backed by Admin API resources. Use when the configuration UI is too rich for Symfony Forms (drag-and-drop, live preview, multi-step wizards, dashboards).
---

## Requirements

Ask the user:
* Vue version (PrestaShop 9 ships Vue 3 across the BO; pick Vue 3 unless an existing module already standardised on Vue 2).
* Bundler. Prefer Vite + TypeScript for new modules; keep Webpack only when extending an existing Webpack pipeline.
* Bundle entry name (`mymodule-config`) - drives the JS/CSS file names committed under `views/{js,css}/dist/`.
* Whether the SPA reads/writes settings through Admin API resources (preferred, see `module-add-admin-api-resource`) or through a dedicated Symfony controller.
* The mount point ID on the page (e.g. `mymodule-config-app`).

## Steps

1. Scaffold the dev tree under `_dev/`. The PrestaShop module zip ships only the built artefacts; `_dev/` is a developer-only workspace ignored by the merchant install:

    ```
    mymodule/
      _dev/
        package.json
        vite.config.ts
        tsconfig.json
        src/
          main.ts
          App.vue
          components/
      views/
        js/dist/      <- COMMIT THE BUILT BUNDLE
        css/dist/     <- COMMIT THE BUILT STYLESHEET
        templates/admin/configure.html.twig
    ```

2. `_dev/package.json` pins Vue 3 and Vite. Use a `build` script that outputs to the module's `views/{js,css}/dist/` with stable file names so the Twig template can reference them without manifest lookup:

    ```json
    {
      "name": "mymodule-config",
      "private": true,
      "scripts": {
        "dev": "vite",
        "build": "vite build"
      },
      "dependencies": { "vue": "^3.4.0" },
      "devDependencies": {
        "@vitejs/plugin-vue": "^5.0.0",
        "typescript": "^5.4.0",
        "vite": "^5.2.0",
        "vue-tsc": "^2.0.0"
      }
    }
    ```

3. `_dev/vite.config.ts` writes to `views/js/dist/` and `views/css/dist/` with deterministic names:

    ```ts
    import { defineConfig } from 'vite';
    import vue from '@vitejs/plugin-vue';
    import { resolve } from 'node:path';

    export default defineConfig({
      plugins: [vue()],
      build: {
        outDir: resolve(__dirname, '..'),
        emptyOutDir: false,
        cssCodeSplit: false,
        rollupOptions: {
          input: resolve(__dirname, 'src/main.ts'),
          output: {
            entryFileNames: 'views/js/dist/mymodule-config.js',
            chunkFileNames: 'views/js/dist/[name]-[hash].js',
            assetFileNames: (info) =>
              info.name?.endsWith('.css')
                ? 'views/css/dist/mymodule-config.css'
                : 'views/[ext]/dist/[name][extname]',
          },
        },
      },
    });
    ```

4. `_dev/src/main.ts` mounts the SPA. Read the initial state from a `data-config` JSON blob written by Twig, so the first paint does not require an API call:

    ```ts
    import { createApp } from 'vue';
    import App from './App.vue';

    const root = document.querySelector<HTMLElement>('#mymodule-config-app');
    if (root) {
      const initialConfig = JSON.parse(root.dataset.config ?? '{}');
      const apiBaseUrl = root.dataset.apiBase ?? '';
      createApp(App, { initialConfig, apiBaseUrl }).mount(root);
    }
    ```

5. Mount the SPA from a modern Symfony admin controller (see `module-add-admin-controller-modern`). Render a Twig template that injects the bundle, the CSRF token, the Admin API base URL, and the initial state:

    ```twig
    {% extends '@PrestaShop/Admin/layout.html.twig' %}
    {% block content %}
      <link rel="stylesheet" href="{{ asset('modules/mymodule/views/css/dist/mymodule-config.css') }}">
      <div
        id="mymodule-config-app"
        data-config="{{ initial_config|json_encode|e('html_attr') }}"
        data-api-base="{{ path('api_admin_mymodule_settings_index') }}"
      ></div>
      <script type="module" src="{{ asset('modules/mymodule/views/js/dist/mymodule-config.js') }}"></script>
    {% endblock %}
    ```

6. Expose the read/write endpoints as Admin API resources (declarative `#[CQRSGet]`, `#[CQRSPartialUpdate]` attributes) so the SPA reuses the OAuth2-protected surface that other API clients use. See `module-add-admin-api-resource` (FEAT-005). For a quick prototype, a private Symfony controller that returns `JsonResponse` and is gated by `#[IsGranted('update', subject: '_legacy_controller')]` is acceptable, but migrate to Admin API resources before release.

7. Build the bundle before commit and ship the artefacts in the module zip:

    ```bash
    cd _dev && npm install && npm run build
    cd .. && git add views/js/dist views/css/dist
    ```

   `_dev/node_modules/` and `_dev/dist/` (if any) belong in `.gitignore`. The committed `views/{js,css}/dist/*` are the merchant-facing payload.

## Do

- Commit the built `views/js/dist/*.js` and `views/css/dist/*.css`. Merchants install the module zip and must NOT need Node, npm, or a build step.
- Pin Vue 3 and Vite. PrestaShop 9 standardises on Vue 3 in the BO; Vue 2 is end-of-life.
- Mount via a modern Symfony admin controller (see `module-add-admin-controller-modern`); never via a legacy `getContent()` echo.
- Hydrate the SPA from server-rendered JSON in a `data-config` attribute. It removes the first-load flash and works offline-first if needed.
- Back the SPA with Admin API resources (`#[CQRSGet]`, `#[CQRSPartialUpdate]`, OAuth2 scopes) so the same data layer powers integrations and tests. See the [Admin API documentation](https://devdocs.prestashop-project.org/9/admin-api/).

## Don't

- Don't require merchants to run `npm install` or `npm run build` after downloading the module zip. The zip must be drop-in.
- Don't load the SPA from a CDN (`unpkg`, `jsdelivr`). Back Office must work offline, and CSP rules will block external scripts.
- Don't bypass the Admin API to write configuration directly from the SPA. Use the OAuth2-protected resources or a controller that wraps a CQRS command.
- Don't render the Vue page from a legacy `AdminController`. Use a modern `PrestaShopAdminController` and Twig (see `module-add-admin-controller-modern`).
- Don't ship the `_dev/` source folder in the published module zip - exclude it via `.zipignore` or your release tooling.

## Canonical examples

- [Vue 3 official guide](https://vuejs.org/) and [Vite documentation](https://vite.dev/).
- [devdocs - Modern admin controllers](https://devdocs.prestashop-project.org/9/modules/concepts/controllers/admin-controllers/).
- [devdocs - Displaying content in the front-office (asset registration patterns)](https://devdocs.prestashop-project.org/9/modules/creation/displaying-content-in-front-office/).
- [devdocs - Admin API](https://devdocs.prestashop-project.org/9/admin-api/) and [`PrestaShop/ps_apiresources`](https://github.com/PrestaShop/ps_apiresources).
- Front-office build pipeline reference: [`PrestaShop/hummingbird`](https://github.com/PrestaShop/hummingbird) (TypeScript + bundler + Storybook tooling, transferable to a Vite-based BO SPA).
- Sample modules repository for general patterns: [`PrestaShop/example-modules`](https://github.com/PrestaShop/example-modules).
