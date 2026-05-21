---
name: theme-data-ps-bindings
description: Wire JavaScript behaviour to a PrestaShop 9 theme via `data-ps-component` / `data-ps-element` / `data-ps-action` attributes (the documented Hummingbird contract) instead of CSS class or `#id` selectors, so refactoring the styling never breaks the behaviour. Use whenever a BEM block needs interactivity.
---

## Requirements

Ask the user:
* Which BEM block (kebab-case) needs behaviour. The same name is used for both the CSS class and the `data-ps-component` value.
* What interactions are needed: click / submit / change / keyboard, with the verbs you want to expose (`open`, `close`, `toggle`, `add-to-cart`, `quick-view`). Each verb becomes a `data-ps-action` value.
* What inner targets the script must read or mutate (`title`, `cta`, `quantity-input`, `error`). Each of those becomes a `data-ps-element` value.
* Whether the component needs to expose state to other components or modules (for example, a global event on `prestashop` - see `theme-add-theme-hook` for the JS event bus).

## Steps

1. Annotate the markup with three documented attribute families. The CSS classes stay for *styling* only:

    | Attribute            | Purpose                                                             |
    |----------------------|---------------------------------------------------------------------|
    | `data-ps-component`  | The block root. One per BEM block, on the outermost element.       |
    | `data-ps-element`    | A named inner part the script reads or mutates.                    |
    | `data-ps-action`     | A verb the script dispatches when the element is interacted with.  |

    ```smarty
    <div class="quick-view" data-ps-component="quick-view" data-ps-product-id="{$product.id_product}">
      <button class="quick-view__trigger btn btn-link"
              data-ps-action="open"
              data-ps-element="trigger"
              type="button">
        {l s='Quick view' d='Shop.Mytheme'}
      </button>

      <div class="quick-view__dialog" role="dialog" hidden data-ps-element="dialog">
        <button class="quick-view__close" data-ps-action="close" data-ps-element="close" aria-label="{l s='Close' d='Shop.Mytheme'}"></button>
        <div data-ps-element="content"></div>
      </div>
    </div>
    ```

    Use `kebab-case` for every attribute value. TypeScript reads them as camelCase via `dataset` (`data-ps-action="open"` -> `el.dataset.psAction === 'open'`).

2. Ship one TypeScript class per component, scoped to the root element. Query the document only by `data-ps-component`; never by a CSS class:

    ```ts
    // src/js/components/quick-view.ts
    type Action = 'open' | 'close';

    export class QuickView {
      private readonly dialog: HTMLElement;

      constructor(private readonly root: HTMLElement) {
        const dialog = this.root.querySelector<HTMLElement>('[data-ps-element="dialog"]');
        if (!dialog) throw new Error('quick-view: missing [data-ps-element="dialog"]');
        this.dialog = dialog;

        this.root.addEventListener('click', this.onClick);
      }

      private onClick = (event: Event): void => {
        const target = (event.target as HTMLElement).closest<HTMLElement>('[data-ps-action]');
        if (!target || !this.root.contains(target)) return;
        const action = target.dataset.psAction as Action | undefined;
        if (action === 'open') this.open();
        if (action === 'close') this.close();
      };

      private open(): void { this.dialog.hidden = false; }
      private close(): void { this.dialog.hidden = true; }
    }
    ```

3. Bootstrap each component once on `DOMContentLoaded`. One `querySelectorAll` per component is enough; never re-scan the DOM from inside the component:

    ```ts
    // src/js/theme.ts
    import { QuickView } from './components/quick-view';
    import { ProductCard } from './components/product-card';

    document.addEventListener('DOMContentLoaded', () => {
      document.querySelectorAll<HTMLElement>('[data-ps-component="quick-view"]')
        .forEach((el) => new QuickView(el));
      document.querySelectorAll<HTMLElement>('[data-ps-component="product-card"]')
        .forEach((el) => new ProductCard(el));
    });
    ```

4. Use **delegated** listeners attached to the component root and dispatch on `data-ps-action`. Delegation keeps the handler count constant when the component re-renders fragments (typical on cart / checkout AJAX updates):

    ```ts
    this.root.addEventListener('click', (e) => {
      const action = (e.target as HTMLElement).closest<HTMLElement>('[data-ps-action]')?.dataset.psAction;
      switch (action) {
        case 'add-to-cart':  return this.addToCart();
        case 'increment':    return this.increment();
        case 'decrement':    return this.decrement();
      }
    });
    ```

5. Pass extra context as `data-ps-*` attributes on the root (`data-ps-product-id`, `data-ps-cart-id`) instead of inline `<script>` blocks or window globals. Read them from `this.root.dataset`:

    ```ts
    const productId = Number(this.root.dataset.psProductId);
    if (!Number.isFinite(productId)) throw new Error('quick-view: missing product id');
    ```

6. Never bind to `.class` or `#id` selectors. Class names exist for the BEM stylesheet and *will* be renamed; `#id` values are global and conflict with module markup. The `data-ps-*` attributes are the documented behaviour contract that survives a CSS refactor.

7. Verify:
    * `npm run lint` passes with no `no-restricted-syntax` warnings against `.querySelector('.foo')` patterns.
    * Replace every CSS class in the SCSS with a different name (temporarily) and confirm the script still works - if it breaks, the bindings are not data-attribute-based.
    * Open the page with the Devtools "break on attribute modification" feature; only `data-ps-*` attributes should change between states.

## Do

- Pair every `data-ps-component` with a typed TS class. Treat the constructor's `root` element as the only DOM the component knows about.
- Use `closest('[data-ps-action]')` for delegation so clicks on inner descendants still resolve to the right verb.
- Keep `data-ps-action` verbs short and imperative (`open`, `submit`, `add-to-cart`, `toggle`).
- Promote shared behaviour to small helper functions (focus trap, aria-live announcer) imported from `src/js/helpers/`.

## Don't

- Don't query by CSS class or `#id` from JS. The next BEM rename will break the page silently.
- Don't ship inline `onclick="..."` handlers or `<script>` blocks in templates. Behaviour belongs in TS modules under `src/js/`.
- Don't depend on jQuery in new themes. The PS 9 storefront keeps it loaded only for legacy modules; new components ship vanilla TS.
- Don't reach across components by querying the document from inside a component. If two components need to talk, dispatch a custom event on the `prestashop` global event bus.

## Canonical examples

- [devdocs - Hummingbird JavaScript conventions](https://devdocs.prestashop-project.org/9/themes/hummingbird/javascript-conventions/) - the `data-ps-*` contract, no jQuery, TypeScript-first.
- [devdocs - Hummingbird CSS conventions](https://devdocs.prestashop-project.org/9/themes/hummingbird/css-conventions/) - explains why CSS classes are *not* a JS contract.
- [`PrestaShop/hummingbird` - `src/js/`](https://github.com/PrestaShop/hummingbird/tree/develop/src/js) - real components driven by `data-ps-*` attributes (`mobile-menu.ts`, `responsive-toggler.ts`, `quickview.ts`).
- [`PrestaShop/hummingbird` - `src/js/components/`](https://github.com/PrestaShop/hummingbird/tree/develop/src/js/components) - reusable TS components paired with BEM blocks.

## Related skills

- `theme-add-bem-component` - the styling counterpart of every JS-bound block.
- `theme-asset-pipeline` - bundles the TypeScript and emits `assets/js/theme.js`.
- `theme-tdd-component` - JSDOM tests assert behaviour against a fixture using these data attributes.
- `theme-add-theme-hook` - the global `prestashop` event bus for cross-component communication.
- `theme-accessibility-audit` - axe-core selectors rely on real semantics, not on whatever CSS classes a refactor renames.
