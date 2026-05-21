---
name: theme-storybook
description: Scaffold Storybook 8+ at the PrestaShop 9 theme root so BEM components render against fixture data outside the front office, with the same SCSS the production build uses, optional Chromatic visual review, and a static build that publishes to GitHub Pages or the marketplace listing. Use whenever the theme has more than a handful of BEM components and a designer or QA needs to review them in isolation.
---

## Requirements

Ask the user:
* Whether the theme is a Hummingbird fork (Storybook is already wired - skip the init step) or fresh (run `storybook init`).
* Framework choice. PrestaShop themes use **`@storybook/html`** with the Webpack 5 builder (matches Hummingbird). Vite-based themes can use `@storybook/html-vite` instead. Vue or Web Components frameworks are not appropriate - the front office renders HTML + Smarty.
* Where stories live. Convention: `stories/<area>/<block>.stories.ts`, mirroring the SCSS partial layout (`stories/theme/`, `stories/ui/`, `stories/images/`).
* Whether to publish a static build (GitHub Pages, S3) or run Chromatic for visual-regression review on every PR.

## Steps

1. Initialise Storybook at the theme root (NOT inside a `_dev/` subdirectory - the toolchain lives next to `config/theme.yml`):

    ```bash
    npx storybook@latest init --type=html --builder=webpack5
    # or, on a Vite-based theme:
    npx storybook@latest init --type=html --builder=vite
    ```

    The init drops `.storybook/`, a few sample stories, and updates `package.json` with two scripts. Keep them named exactly:

    ```jsonc
    {
      "scripts": {
        "storybook":       "storybook dev -p 6006",
        "storybook:build": "storybook build"
      }
    }
    ```

2. Wire Storybook to the theme's existing SCSS so stories render with production styling. Import the same entry the front office loads, plus a token decorator:

    ```ts
    // .storybook/preview.ts
    import type { Preview } from '@storybook/html';
    import '../src/scss/theme.scss';                    // same SCSS as production

    const preview: Preview = {
      parameters: {
        backgrounds: { default: 'page', values: [{ name: 'page', value: '#fff' }] },
        viewport:    { defaultViewport: 'responsive' },
        a11y:        { config: { rules: [{ id: 'color-contrast', enabled: true }] } },
      },
      globalTypes: {
        theme: {
          name: 'Theme',
          defaultValue: 'light',
          toolbar: { items: ['light', 'dark'] },
        },
      },
      decorators: [
        (Story, ctx) => {
          document.documentElement.dataset.psTheme = String(ctx.globals.theme);
          return Story();
        },
      ],
    };
    export default preview;
    ```

    Add `@storybook/addon-a11y` to the addons list in `.storybook/main.{js,ts}` so each story runs axe-core automatically.

3. Write one story file per BEM block under `stories/<area>/<block>.stories.ts`. The render function returns a string of HTML matching the markup from `theme-add-bem-component`:

    ```ts
    // stories/theme/product-card.stories.ts
    import type { Meta, StoryObj } from '@storybook/html';

    type Args = {
      name: string;
      price: string;
      featured: boolean;
      outOfStock: boolean;
    };

    const render = (args: Args) => `
      <article class="product-card${args.featured ? ' product-card--featured' : ''}${args.outOfStock ? ' product-card--out-of-stock' : ''}"
               data-ps-component="product-card">
        <div class="product-card__media"><img src="/preview.png" alt=""></div>
        <h3 class="product-card__title">${args.name}</h3>
        <span class="product-card__price">${args.price}</span>
        <button class="btn btn-primary product-card__cta" data-ps-action="quick-view">Quick view</button>
      </article>
    `;

    const meta: Meta<Args> = {
      title: 'Catalog/Product card',
      render,
      args:  { name: 'Hummingbird printed t-shirt', price: '€23.90', featured: false, outOfStock: false },
    };
    export default meta;

    export const Default:    StoryObj<Args> = {};
    export const Featured:   StoryObj<Args> = { args: { featured: true } };
    export const OutOfStock: StoryObj<Args> = { args: { outOfStock: true } };
    ```

    One story per modifier and one per significant element layout. Stories ship with **fixture** data, never live PrestaShop data.

4. Reuse the theme's design tokens (`--ps-*`, `--bs-*`) via the `themes` decorator. Do not hard-code colours or spacing in stories - if a story needs a value the tokens do not expose, that is a sign to add the token first (see `theme-add-bem-component`).

5. Run Storybook in dev:

    ```bash
    npm run storybook                          # http://localhost:6006
    ```

    And build the static site for CI / publication:

    ```bash
    npm run storybook:build                    # writes storybook-static/
    ```

    `storybook-static/` is a deployable artefact (GitHub Pages, S3) - **never** commit it to the theme zip; the `theme-export` exclusions strip it.

6. Optional - integrate Chromatic for visual-regression review:

    ```bash
    npx chromatic --project-token=<token>      # uploads the build, gates the PR
    ```

    Chromatic snapshots every story across browsers and breakpoints; reviewers approve diffs in the PR.

7. Verify:
    * `npm run storybook` boots; every block has at least a default story.
    * `npm run storybook:build` succeeds and produces `storybook-static/index.html`.
    * The a11y addon shows zero violations on every story (matches the gate from `theme-accessibility-audit`).
    * `git status` after a build shows no changes inside `assets/` (Storybook does not pollute production output).

## Do

- Keep Storybook at the theme root, alongside `package.json` and `webpack.config.js`. Do not move it into `_dev/`.
- Reuse production SCSS via a single `import '../src/scss/theme.scss'` in `.storybook/preview.ts`. Stories drift from production the moment they have their own SCSS.
- Use `@storybook/html` with the Webpack 5 builder for Hummingbird forks. Use `@storybook/html-vite` for fresh Vite-based themes. Match the bundler the rest of the theme uses.
- Add `@storybook/addon-a11y` to surface accessibility regressions per story.

## Don't

- Don't ship `storybook-static/` inside the marketplace zip. Add it to `.gitignore` and to the export exclusions.
- Don't use a Vue or Web Components Storybook framework. PrestaShop renders HTML + Smarty; pick `@storybook/html` so stories match the runtime.
- Don't pull live data into stories. Fixtures are deterministic; live data makes diffs noisy and breaks Chromatic.
- Don't duplicate the SCSS into a story-only entry point. One source of truth, imported from `preview.ts`.

## Canonical examples

- [Storybook documentation](https://storybook.js.org/) - framework reference (HTML, Vite / Webpack5, addon-a11y, Chromatic integration).
- [Chromatic](https://www.chromatic.com/) - visual-regression review service paired with Storybook.
- [`PrestaShop/hummingbird` - `.storybook/`](https://github.com/PrestaShop/hummingbird/tree/develop/.storybook) - canonical Storybook 8 config (HTML + Webpack5 builder, addon-a11y, theming decorator).
- [`PrestaShop/hummingbird` - `.storybook/preview.js`](https://github.com/PrestaShop/hummingbird/blob/develop/.storybook/preview.js) - reference `preview` file that imports the production SCSS.
- [`PrestaShop/hummingbird` - `stories/`](https://github.com/PrestaShop/hummingbird/tree/develop/stories) - real story files organised by area (`theme/`, `ui/`, `images/`).

## Related skills

- `theme-add-bem-component` - the BEM markup each story renders.
- `theme-data-ps-bindings` - the data attributes the rendered fixtures should carry so JS bindings work in stories too.
- `theme-asset-pipeline` - shares the bundler and SCSS with Storybook; never spin up a parallel pipeline.
- `theme-tdd-component` - Vitest / Jest unit tests for behaviour, Storybook for visual review, Playwright for E2E - one tool per layer.
- `theme-accessibility-audit` - the addon-a11y signal here feeds the global a11y gate.
