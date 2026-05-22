---
name: theme-asset-pipeline
description: Wire a PrestaShop 9 theme to a Node-based bundler (Hummingbird ships Webpack; new themes can pick Vite) with SCSS, TypeScript, ESLint, Stylelint, Prettier and a test runner, then declare the compiled outputs in `config/theme.yml` so PrestaShop loads them at the right priority. Use whenever a theme grows past plain `assets/css/theme.css` and needs a real build.
---

## Requirements

Ask the user:
* Bundler choice: **Webpack 5** (matches Hummingbird, lowest friction when forking it) or **Vite** (lighter config, faster dev server). Default to whatever the parent theme already uses; Webpack for Hummingbird forks, Vite for fresh themes. Do not mix the two.
* Entry points: one bundle per major page family is the canonical layout (`theme`, `checkout`, `account`, ...). Ask which page families need their own bundle so the SCSS / TS split lines up with `config/theme.yml`.
* Where the sources live and where the build emits. The convention is `src/{js,scss}/...` -> `assets/{js,css}/...`. Keep the toolchain at the **theme root** (Hummingbird does not ship a `_dev/` subdirectory; the `package.json`, `.nvmrc`, `webpack.config.js` / `vite.config.ts` and `tsconfig.json` all live next to `config/theme.yml`).
* Whether the theme will keep a `jest` / `vitest` unit test stage and a Storybook stage (delegate to `theme-tdd-component` and `theme-storybook`).

## Steps

1. Pin Node 20 LTS and ship a `.nvmrc` so contributors and CI agree on the runtime. Hummingbird's `engines.node` is `>= 20`:

    ```bash
    echo "20" > .nvmrc
    ```

    ```jsonc
    // package.json (excerpt)
    {
      "name": "mytheme",
      "version": "1.0.0",
      "engines": { "node": ">= 20" },
      "scripts": {
        "dev":        "webpack serve --progress --mode=development",
        "watch":      "webpack --progress --watch --mode=development",
        "build":      "webpack --progress --mode=production",
        "lint":       "eslint --ext .js,.ts ./src/js",
        "lint:fix":   "eslint --ext .js,.ts ./src/js --fix",
        "stylelint":  "stylelint \"src/scss/**/*.scss\"",
        "format":     "prettier --check \"src/scss/**/*.scss\"",
        "format:fix": "prettier --write \"src/scss/**/*.scss\"",
        "test":       "jest"
      }
    }
    ```

    Replace `webpack ...` with `vite` / `vite build` if you picked Vite. Keep the script *names* (`dev`, `build`, `lint`, `format`, `test`) stable - skills, CI templates and Storybook scripts all assume them.

2. Lay out the source tree at the theme root:

    ```
    mytheme/
    ├── .nvmrc
    ├── package.json
    ├── tsconfig.json              # strict: true, target ES2022, module ESNext
    ├── webpack.config.js          # OR vite.config.ts
    ├── postcss.config.js          # autoprefixer + postcss-preset-env
    ├── .eslintrc.js / eslint.config.js
    ├── .prettierrc.js
    ├── stylelint config (in package.json or .stylelintrc)
    ├── src/
    │   ├── js/                    # TypeScript entries, organised by feature
    │   │   ├── theme.ts
    │   │   ├── checkout.ts
    │   │   ├── account.ts
    │   │   └── components/<block>.ts
    │   └── scss/
    │       ├── theme.scss         # @use 'components/_*' partials
    │       ├── checkout.scss
    │       └── components/_<block>.scss
    └── assets/                    # build output, COMMITTED to git
        ├── js/<bundle>.js
        └── css/<bundle>.css
    ```

3. Configure each entry to emit a stable filename so PHP / `theme.yml` references stay valid. Do not enable content hashing on production assets - PrestaShop references them by name, and the marketplace zip ships compiled output:

    ```js
    // webpack.config.js (excerpt)
    module.exports = (env, argv) => ({
      entry: {
        theme:    './src/js/theme.ts',
        checkout: './src/js/checkout.ts',
        account:  './src/js/account.ts',
      },
      output: {
        path: path.resolve(__dirname, 'assets/js'),
        filename: '[name].js',          // no [contenthash] - stable names
        clean: false,                   // keep checked-in art outside dist/
      },
      // ts-loader / esbuild-loader for .ts, sass-loader + mini-css-extract for .scss
    });
    ```

    For Vite, set `build.rollupOptions.output.entryFileNames` and `assetFileNames` to `[name].js` / `[name].css` and `build.outDir` to `assets/`.

4. Use TypeScript with `strict: true`. Do not introduce jQuery into a new theme - the PS 9 storefront ships `core.js` only for legacy compatibility. Couple JS to the DOM through `data-ps-component` / `data-ps-element` (see `theme-data-ps-bindings`), not through CSS class selectors:

    ```jsonc
    // tsconfig.json
    {
      "compilerOptions": {
        "target": "ES2022",
        "module": "ESNext",
        "strict": true,
        "noUncheckedIndexedAccess": true,
        "moduleResolution": "Bundler",
        "esModuleInterop": true,
        "isolatedModules": true,
        "skipLibCheck": true,
        "types": ["bootstrap"]
      },
      "include": ["src/js/**/*.ts"]
    }
    ```

5. Declare the compiled outputs in `config/theme.yml`. The `assets.css` and `assets.js` arrays are the documented contract (no `assets.rtl_css` key - RTL siblings are filename-discovered, see `theme-rtl-support`). Each entry takes `id`, `path`, `media` (`all` / `print` / `screen`) and `priority` (lower numbers load first):

    ```yaml
    assets:
      use_parent_assets: false
      css:
        - { id: theme-main,     path: assets/css/theme.css,    media: all,    priority: 200 }
        - { id: theme-print,    path: assets/css/print.css,    media: print,  priority: 800 }
      js:
        - { id: theme-main,     path: assets/js/theme.js,                     priority: 200 }
        - { id: theme-checkout, path: assets/js/checkout.js,                  priority: 200 }
    ```

6. Commit the built artefacts (`assets/js/*.js`, `assets/css/*.css`). Merchants install the theme without running `npm`; the marketplace zip ships compiled output. Do not commit sourcemaps, `node_modules/` or the bundler cache directory.

7. Verify:
    * `nvm use && npm ci && npm run build` succeeds, leaves a clean `git status` (or only updates the committed `assets/` outputs).
    * `npm run lint`, `npm run stylelint` and `npm run format` all pass.
    * Browse the front office with Developer Mode on and confirm the Symfony "Performance" panel lists each compiled `theme.js` / `theme.css` with the priorities declared in `theme.yml`.

## Do

- Vanilla TypeScript, BEM SCSS, no jQuery in new themes. Reach for `data-ps-component` for JS hooks (see `theme-data-ps-bindings`).
- Keep the toolchain at the theme root, not in a `_dev/` subdirectory. Hummingbird collapsed that split a long time ago.
- Pin Node via `.nvmrc` and lock dependency majors via `package-lock.json`. Commit the lockfile.
- Stable, hash-free output filenames so PHP and `config/theme.yml` references stay valid across rebuilds.

## Don't

- Don't ship sourcemaps in the marketplace zip. Treat `*.map` as build-time artefacts; gitignore them or strip them in the export step.
- Don't add jQuery to a fresh PS 9 theme. If a third-party module needs it, the module ships its own bundle.
- Don't reference `[contenthash]` filenames in `theme.yml` - PrestaShop has no manifest indirection.
- Don't keep a parallel `_dev/` mirror of the build setup. Drift between the root and `_dev/` configs is the single biggest source of broken Hummingbird forks.

## Canonical examples

- [devdocs - Asset management](https://devdocs.prestashop-project.org/9/themes/concepts/asset-management/) - the `assets.css` / `assets.js` schema, `media`, `priority`, `needRtl`.
- [devdocs - Hummingbird development workflow](https://devdocs.prestashop-project.org/9/themes/hummingbird/development-workflow/) - the `dev` / `watch` / `build` / `lint` / `format` / `test` script set.
- [devdocs - Hummingbird requirements](https://devdocs.prestashop-project.org/9/themes/getting-started/requirements/) - Node 20, npm and supported PS versions.
- [`PrestaShop/hummingbird` - `package.json`](https://github.com/PrestaShop/hummingbird/blob/develop/package.json) - reference scripts, devDependencies and `engines.node`.
- [`PrestaShop/hummingbird` - `webpack.config.js`](https://github.com/PrestaShop/hummingbird/blob/develop/webpack.config.js) - canonical Webpack 5 config for a PS 9 theme.
- [`PrestaShop/hummingbird` - `tsconfig.json`](https://github.com/PrestaShop/hummingbird/blob/develop/tsconfig.json) - `strict: true` baseline.
- [`PrestaShop/hummingbird` - `.nvmrc`](https://github.com/PrestaShop/hummingbird/blob/develop/.nvmrc) - Node version pin.
- [Vite documentation](https://vite.dev/guide/) - alternative bundler reference for fresh themes.

## Related skills

- `theme-create-from-hummingbird` - bootstraps the whole Webpack pipeline by cloning Hummingbird.
- `theme-add-bem-component` - the SCSS / BEM conventions consumed by this pipeline.
- `theme-data-ps-bindings` - how the TypeScript side binds to the DOM without coupling to CSS classes.
- `theme-rtl-support` - how RTL siblings are produced from the same build.
- `theme-tdd-component` - Jest / Vitest unit tests, Storybook visual tests and Playwright E2E inside this pipeline.
- `theme-export` - what to keep (and what to strip) from the build output when zipping the theme.
