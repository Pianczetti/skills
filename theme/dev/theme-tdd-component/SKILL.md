---
name: theme-tdd-component
description: Drive a PrestaShop 9 theme component from a failing test - a Jest or Vitest unit test against a JSDOM fixture for behaviour, a Storybook story for visual review, and a Playwright E2E spec on a real PS instance for the full integration. Use whenever a new BEM component carries non-trivial JavaScript behaviour (cart updates, accordion, quick view, mobile menu).
---

## Requirements

Ask the user:
* Component name (kebab-case BEM block, e.g. `quick-view`, `mobile-menu`, `quantity-stepper`).
* The behaviour to spec, expressed as observable DOM mutations or events: "clicking `[data-ps-action='open']` removes `hidden` from `[data-ps-element='dialog']`", "submitting the form dispatches `prestashop:quick-view:open` on the document".
* Test runner. Hummingbird ships **Jest** with `jest-environment-jsdom`. Fresh themes can pick **Vitest** for the same model with faster feedback. Pick one and stay consistent across the theme.
* Whether the E2E layer runs against a Docker-backed PS dev stack (preferred) or against a remote staging environment.

## Steps

1. Write the **unit test first** under `src/js/components/<block>.test.ts` (colocated next to the source). Spec the smallest observable contract: DOM in -> DOM out:

    ```ts
    // src/js/components/quick-view.test.ts
    import { describe, expect, it, beforeEach } from '@jest/globals';   // or 'vitest'
    import { QuickView } from './quick-view';

    function fixture(): HTMLElement {
      document.body.innerHTML = `
        <div data-ps-component="quick-view" data-ps-product-id="42">
          <button data-ps-action="open" data-ps-element="trigger">Open</button>
          <div data-ps-element="dialog" hidden>
            <button data-ps-action="close" data-ps-element="close">Close</button>
          </div>
        </div>`;
      return document.body.querySelector<HTMLElement>('[data-ps-component="quick-view"]')!;
    }

    describe('quick-view', () => {
      let root: HTMLElement;
      beforeEach(() => { root = fixture(); new QuickView(root); });

      it('opens when the trigger is clicked', () => {
        root.querySelector<HTMLElement>('[data-ps-action="open"]')!.click();
        expect(root.querySelector<HTMLElement>('[data-ps-element="dialog"]')!.hidden).toBe(false);
      });

      it('closes when the close button is clicked', () => {
        root.querySelector<HTMLElement>('[data-ps-action="open"]')!.click();
        root.querySelector<HTMLElement>('[data-ps-action="close"]')!.click();
        expect(root.querySelector<HTMLElement>('[data-ps-element="dialog"]')!.hidden).toBe(true);
      });
    });
    ```

    Run it (`npm test`) and confirm the test **fails** because the implementation does not exist yet.

2. Implement the minimum TS class to make the unit test pass. Follow `theme-data-ps-bindings` (root-scoped delegated listeners, `data-ps-action` dispatch, no class / id selectors):

    ```ts
    // src/js/components/quick-view.ts
    export class QuickView {
      constructor(private readonly root: HTMLElement) {
        const dialog = root.querySelector<HTMLElement>('[data-ps-element="dialog"]')!;
        root.addEventListener('click', (e) => {
          const action = (e.target as HTMLElement).closest<HTMLElement>('[data-ps-action]')?.dataset.psAction;
          if (action === 'open')  dialog.hidden = false;
          if (action === 'close') dialog.hidden = true;
        });
      }
    }
    ```

    Re-run `npm test`; the suite should turn green.

3. Add the SCSS partial **after** the behaviour is correct. Style is declarative; keep behaviour out of CSS. See `theme-add-bem-component` for the BEM rules:

    ```scss
    // src/scss/components/_quick-view.scss
    .quick-view {
      &__dialog { display: none; }
      &__dialog:not([hidden]) { display: block; }
    }
    ```

4. Write a Storybook story for visual review (see `theme-storybook`). Cover each modifier the unit tests assert:

    ```ts
    // stories/theme/quick-view.stories.ts
    import type { Meta, StoryObj } from '@storybook/html';
    const render = () => `
      <div class="quick-view" data-ps-component="quick-view" data-ps-product-id="42">
        <button class="btn btn-link" data-ps-action="open">Open</button>
        <div class="quick-view__dialog" data-ps-element="dialog" hidden>
          <button class="btn-close" data-ps-action="close" aria-label="Close"></button>
          <p>Quick view contents</p>
        </div>
      </div>`;
    const meta: Meta = { title: 'Catalog/Quick view', render };
    export default meta;
    export const Default: StoryObj = {};
    ```

5. Add a Playwright E2E spec under `tests/e2e/<block>.spec.ts` that exercises the component on a real PS instance. The unit test proves the logic; the E2E test proves the integration (asset pipeline output, theme.yml registration, Smarty markup):

    ```ts
    // tests/e2e/quick-view.spec.ts
    import { test, expect } from '@playwright/test';

    test('quick view opens from the product card', async ({ page }) => {
      await page.goto('/3-clothes', { waitUntil: 'networkidle' });
      await page.locator('[data-ps-component="product-card"] [data-ps-action="quick-view"]').first().click();
      await expect(page.locator('[data-ps-component="quick-view"] [data-ps-element="dialog"]')).toBeVisible();
    });
    ```

    Boot the PS dev stack (`docker compose up -d`) before running `npm run e2e`.

6. Wire the scripts in `package.json` so each layer runs by name:

    ```jsonc
    {
      "scripts": {
        "test":     "jest",
        "test:watch": "jest --watch",
        "e2e":      "playwright test"
      }
    }
    ```

7. Verify:
    * `npm test` is green and covers every behaviour the spec listed.
    * `npm run storybook` shows the component in every modifier.
    * `npm run e2e` is green against the PS dev stack.
    * `npm run lint` and `npm run stylelint` pass.

## Do

- Write the unit test **first** and watch it fail. Test-after often skips edge cases the test-first dialogue surfaces.
- Keep behaviour in TypeScript, styling in SCSS. The TS class is fully testable in JSDOM; the SCSS is declarative and reviewed visually.
- Colocate `<block>.test.ts` next to `<block>.ts` (Hummingbird's convention with `responsive-toggler.test.ts`). Tests and source travel together.
- Use the same `data-ps-*` selectors in the unit test, the Storybook story and the Playwright spec. One DOM contract, three layers of verification.

## Don't

- Don't reach into the global `document` from a component test. Build a JSDOM fixture, attach it, then exercise the component scoped to its root.
- Don't put behaviour in CSS (`:hover` toggling state via `:has()` shenanigans). Behaviour has to be testable; CSS is not.
- Don't skip the Playwright layer. The unit test cannot prove `theme.yml` references the right asset bundle or that Smarty rendered the right markup.
- Don't fixture the test against CSS classes. Anchor on `data-ps-component` / `data-ps-element` / `data-ps-action` so the test survives a BEM rename.

## Canonical examples

- [devdocs - Hummingbird JavaScript conventions](https://devdocs.prestashop-project.org/9/themes/hummingbird/javascript-conventions/) - the `data-ps-*` contract the tests anchor on.
- [devdocs - Hummingbird development workflow](https://devdocs.prestashop-project.org/9/themes/hummingbird/development-workflow/) - canonical script names (`test`, `lint`, `build`).
- [`PrestaShop/hummingbird` - `src/js/responsive-toggler.test.ts`](https://github.com/PrestaShop/hummingbird/blob/develop/src/js/responsive-toggler.test.ts) - real Jest + JSDOM unit test colocated next to the component source.
- [`PrestaShop/hummingbird` - `src/js/responsive-toggler.ts`](https://github.com/PrestaShop/hummingbird/blob/develop/src/js/responsive-toggler.ts) - the component implementation that test exercises.
- [Vitest documentation](https://vitest.dev/) - alternative to Jest for fresh themes; same JSDOM model, faster feedback.
- [Playwright documentation](https://playwright.dev/) - the E2E layer for component integration on a real PS instance.

## Related skills

- `theme-add-bem-component` - BEM markup the test fixture renders.
- `theme-data-ps-bindings` - the JS contract the unit test asserts against.
- `theme-asset-pipeline` - bundles the TS so Playwright can hit `assets/js/theme.js` on the dev stack.
- `theme-storybook` - visual layer paired with the unit and E2E layers.
- `theme-accessibility-audit` - axe assertions can drop into the same Playwright spec for per-component a11y coverage.
