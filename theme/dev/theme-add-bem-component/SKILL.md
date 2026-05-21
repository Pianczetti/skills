---
name: theme-add-bem-component
description: Add a Block / Element / Modifier styled component to a PrestaShop 9 theme - one block per SCSS file, no descendant selectors, design tokens via CSS custom properties (`--ps-*` or `--bs-*`), and a typed TypeScript class wired through `data-ps-component`. Use whenever the user introduces a new visual building block (product card, badge, accordion, mega-menu).
---

## Requirements

Ask the user:
* Block name in **kebab-case** (e.g. `product-card`, `notification-banner`, `accordion`). The block name is the only "ambient" class - everything else nests under it via `__element` or `--modifier`.
* The list of elements (`__title`, `__price`, `__cta`, `__media`) and modifiers (`--featured`, `--out-of-stock`, `--compact`). Modifiers describe a *state* of the block, not a different block.
* Whether the component carries behaviour. If yes, plan the TypeScript counterpart now (see `theme-data-ps-bindings`); if no, the SCSS partial alone is the deliverable.
* Which design tokens drive the visuals (`--ps-color-primary`, `--bs-border-radius`, `--ps-spacing-3`). Reuse the theme's existing token vocabulary; introduce a new token only when no existing one fits.

## Steps

1. Create exactly **one** SCSS partial under `src/scss/components/_<block>.scss`. The file declares the block and nothing else - no global resets, no helpers, no other components:

    ```scss
    // src/scss/components/_product-card.scss
    .product-card {
      display: flex;
      flex-direction: column;
      gap: var(--ps-spacing-3);
      padding: var(--ps-spacing-4);
      border-radius: var(--bs-border-radius);
      background: var(--bs-body-bg);
      color: var(--bs-body-color);

      &__media   { aspect-ratio: 1 / 1; overflow: hidden; }
      &__title   { font: var(--ps-font-heading-sm); margin: 0; }
      &__price   { font-weight: 600; color: var(--ps-color-primary); }
      &__cta     { margin-top: auto; }

      &--featured  { border: 1px solid var(--ps-color-primary); }
      &--compact   { padding: var(--ps-spacing-2); gap: var(--ps-spacing-2); }
      &--out-of-stock {
        opacity: .55;

        .product-card__cta { pointer-events: none; }
      }
    }
    ```

    Use SCSS nesting (`&__element`, `&--modifier`) only as a typing shortcut. The compiled output is still flat single-class selectors.

2. Import the partial from the bundle that needs it. Do not register components globally - per-page bundles only pull what they render. From `src/scss/theme.scss`:

    ```scss
    @use 'components/product-card';
    @use 'components/badge';
    ```

3. Drive design decisions through CSS custom properties. The theme's token layer (defined once in `src/scss/_tokens.scss` or inherited from Hummingbird's `--bs-*` overrides) is the single source of truth:

    ```scss
    :root {
      --ps-color-primary:   #1a73e8;
      --ps-color-on-primary:#fff;
      --ps-spacing-2: .5rem;
      --ps-spacing-3: .75rem;
      --ps-spacing-4: 1rem;
      --ps-font-heading-sm: 600 1.125rem/1.4 var(--bs-font-sans-serif);
    }
    ```

    Never inline literals (`#1a73e8`, `12px`, `1rem`) in a component. If a value is one-off, promote it to a token first.

4. Write the markup with **flat** BEM classes - no descendant selectors, no chaining `.product-card .price`. Every meaningful sub-element gets its own `block__element` class:

    ```smarty
    {* templates/catalog/_partials/miniatures/product.tpl (excerpt) *}
    <article class="product-card{if $product.has_discount} product-card--featured{/if}"
             data-ps-component="product-card"
             data-ps-id="{$product.id_product}">
      <a class="product-card__media" href="{$product.url}" data-ps-element="media">
        <img src="{$product.cover.medium.url}" alt="{$product.cover.legend|escape:'html'}">
      </a>
      <h3 class="product-card__title">
        <a href="{$product.url}" data-ps-element="title">{$product.name}</a>
      </h3>
      <span class="product-card__price">{$product.price}</span>
      <button class="btn btn-primary product-card__cta"
              data-ps-action="quick-view"
              data-ps-element="cta">{l s='Quick view' d='Shop.Mytheme'}</button>
    </article>
    ```

    Theme-owned strings live in the `Shop.<Themename>` translation domain (e.g. `Shop.Mytheme`), never the module-style `Modules.<modulename>.*` domain.

5. Pair the block with one TypeScript class scoped via `data-ps-component`. The CSS class `product-card` is for styling only; the JS query selects on the data attribute. Full pattern in `theme-data-ps-bindings`:

    ```ts
    // src/js/components/product-card.ts
    export class ProductCard {
      constructor(private readonly root: HTMLElement) {
        this.root.addEventListener('click', (e) => {
          const action = (e.target as HTMLElement).closest<HTMLElement>('[data-ps-action]')?.dataset.psAction;
          if (action === 'quick-view') this.openQuickView();
        });
      }
      private openQuickView(): void { /* ... */ }
    }

    document.querySelectorAll<HTMLElement>('[data-ps-component="product-card"]')
      .forEach((el) => new ProductCard(el));
    ```

6. Add a Storybook story so the block can be reviewed in isolation against fixture data. See `theme-storybook` for the wiring; the convention is `stories/<area>/<block>.stories.ts`.

7. Verify:
    * `npm run stylelint` passes with the BEM plugin (Hummingbird's `stylelint-config-prestashop` already enforces the conventions).
    * `npm run build` produces `assets/css/theme.css` containing the flat `.product-card`, `.product-card__title`, `.product-card--featured` selectors only - no `.product-card .title` style descendants.
    * The Storybook story renders the block with each modifier and at least one element layout.

## Do

- One block per file under `src/scss/components/_<block>.scss`. Every block lives by itself; cross-block coupling lives in a layout partial.
- Class names follow `block`, `block__element`, `block--modifier` exactly. No deeper nesting (`block__element__sub` is a smell - that sub-element is its own block).
- Drive every measurable property through a `--ps-*` or `--bs-*` token. Themes rebrand by overriding the tokens, not by re-skinning components.
- Pair every interactive block with a typed TS class hooked through `data-ps-component`.

## Don't

- Don't write descendant selectors (`.product-card .title`, `.product-card a`). They couple two blocks and break the one-class contract.
- Don't reach into another block's internals (`.product-card .price-tag__value`). If two blocks need to share a piece, extract a third block.
- Don't use modifiers to mean "different block" (`.product-card--banner`). At that point it is a `banner` block of its own.
- Don't hardcode colour, spacing or radius literals inside a component. Promote to a token first.

## Canonical examples

- [devdocs - Hummingbird CSS conventions](https://devdocs.prestashop-project.org/9/themes/hummingbird/css-conventions/) - BEM rules, custom-properties layer and override order Hummingbird enforces.
- [devdocs - Theme structure](https://devdocs.prestashop-project.org/9/themes/concepts/theme-structure/) - where `assets/css/theme.css` lands inside the theme tree.
- [BEM - Quick start](https://en.bem.info/) - canonical BEM methodology reference (block / element / modifier semantics).
- [BEM - Naming convention](https://getbem.com/introduction/) - alternate explainer with the `block__element--modifier` syntax used throughout PS themes.
- [`PrestaShop/hummingbird` - `src/scss/`](https://github.com/PrestaShop/hummingbird/tree/develop/src/scss) - real-world component partials following the conventions described here.

## Related skills

- `theme-data-ps-bindings` - the JS-binding contract that pairs with every BEM block.
- `theme-asset-pipeline` - where the SCSS partial gets imported and compiled.
- `theme-storybook` - how to render the block against fixture data in isolation.
- `theme-tdd-component` - test-first workflow for the block's TS class.
- `theme-rtl-support` - logical CSS properties keep BEM partials direction-agnostic.
