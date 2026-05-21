---
name: theme-accessibility-audit
description: Run an automated and manual WCAG 2.1 AA audit on every main PrestaShop 9 page family using axe-core (via Playwright or Puppeteer), Lighthouse and pa11y, plus a documented manual checklist for things tools can't catch (focus order, semantic structure, copy quality). Use before publishing a theme to the marketplace and as a CI gate on every PR.
---

## Requirements

Ask the user:
* Target conformance level. Default to **WCAG 2.1 AA** (the PrestaShop guideline). Bump to AAA only if the merchant has an explicit accessibility statement requiring it.
* Tooling stack:
  * **`@axe-core/playwright`** for headless-browser scans on a real PS install (preferred - matches the real DOM after JS hydration).
  * **Lighthouse** (Chrome DevTools or `lhci`) for performance + a11y combined regressions on a few key URLs.
  * **pa11y** for an additional crawler with HTML CodeSniffer and Axe rule sets.
* Which page families ship in scope. Minimum: home, category listing, product, cart, checkout (each step), customer login / registration, my account, contact, 404. Authenticated pages need a fixture login.
* Whether the audit runs on every PR (recommended) or only nightly. Both lanes should fail the build on `wcag2a` / `wcag2aa` violations.

## Steps

1. Install the tooling alongside the theme's existing dev pipeline (see `theme-asset-pipeline`):

    ```bash
    npm install --save-dev @playwright/test @axe-core/playwright pa11y lighthouse
    npx playwright install --with-deps
    ```

2. Write a Playwright spec that loads each page family on a running PS dev instance and runs axe-core. Fail on any violation tagged `wcag2a` or `wcag2aa`:

    ```ts
    // tests/a11y.spec.ts
    import { test, expect } from '@playwright/test';
    import AxeBuilder from '@axe-core/playwright';

    const PAGES: Array<[string, string]> = [
      ['home',     '/'],
      ['category', '/3-clothes'],
      ['product',  '/en/men/1-1-hummingbird-printed-t-shirt.html'],
      ['cart',     '/cart?action=show'],
      ['checkout', '/order'],
      ['login',    '/login'],
      ['account',  '/my-account'],
      ['contact',  '/contact-us'],
      ['404',      '/this-page-does-not-exist'],
    ];

    for (const [name, path] of PAGES) {
      test(`a11y: ${name}`, async ({ page }) => {
        await page.goto(path, { waitUntil: 'networkidle' });
        const results = await new AxeBuilder({ page })
          .withTags(['wcag2a', 'wcag2aa', 'wcag21a', 'wcag21aa'])
          .analyze();
        expect(results.violations, JSON.stringify(results.violations, null, 2)).toEqual([]);
      });
    }
    ```

    Add fixtures (logged-in customer, populated cart) via Playwright's storage state so authenticated pages get covered.

3. Add a Lighthouse run that scores the same URLs and tracks the accessibility budget over time:

    ```bash
    npx lighthouse https://localhost:8001/ \
      --only-categories=accessibility \
      --chrome-flags="--headless --ignore-certificate-errors" \
      --output=json --output-path=./reports/lighthouse-home.json
    ```

    Treat the score as a regression budget (e.g. fail if the score drops below 95 or below the previous baseline).

4. Run pa11y as a third opinion. It catches a few rules axe-core does not (and vice versa), and its HTML CodeSniffer rule set maps directly to WCAG techniques:

    ```bash
    npx pa11y --standard WCAG2AA https://localhost:8001/
    ```

5. Manually audit what tools cannot infer. Document the checklist in `tests/a11y/CHECKLIST.md` and walk it before each release:

    * **Document structure**: a single `<h1>` per page, headings nested in order without skipping levels, landmarks `<header>`, `<nav>`, `<main>`, `<aside>`, `<footer>` present and unique.
    * **Forms**: every `<input>` / `<select>` / `<textarea>` has a programmatically-associated `<label>`. Validation errors are announced via `aria-live` regions, not just colour.
    * **Images**: meaningful images carry an `alt`; decorative ones use `alt=""` or CSS background images.
    * **Focus**: focus is visible on every interactive element (no `outline: none` without a replacement); focus order matches reading order; modal / off-canvas widgets trap focus and restore it on close.
    * **Colour**: contrast meets AA (4.5:1 for body text, 3:1 for large text and UI components). Verify against the theme's design tokens, not against rendered pixels alone.
    * **Keyboard**: every interaction (menu, dropdown, accordion, carousel, dialog) is reachable and operable with `Tab`, `Shift+Tab`, `Enter`, `Space` and `Esc`.
    * **Reduced motion**: animations respect `prefers-reduced-motion`.
    * **Language**: `<html lang="...">` reflects the active language; sub-trees in another language carry their own `lang` attribute.

6. Wire the suite into CI so `wcag2a` / `wcag2aa` violations fail the build:

    ```yaml
    # .github/workflows/a11y.yml (excerpt)
    - run: npm ci
    - run: npm run build
    - run: docker compose up -d            # boot the PS dev stack
    - run: npx playwright test tests/a11y.spec.ts
    - run: npx pa11y-ci
    ```

7. Verify:
    * Playwright + axe-core report zero `wcag2a` / `wcag2aa` violations on every page in `PAGES`.
    * Lighthouse accessibility category is at or above the budget on each URL.
    * pa11y completes with no errors at WCAG2AA.
    * The manual checklist is signed off in the PR description.

## Do

- Ship the a11y spec as a CI job that fails the build on regressions. A passing local run is not enough.
- Cover the **authenticated** pages too (my account, address book, order history). Most regressions hide behind login.
- Re-run the audit with the front office in an RTL language (see `theme-rtl-support`); RTL surfaces direction-specific focus and reading-order regressions.
- Keep a `tests/a11y/CHECKLIST.md` so manual review is reproducible.

## Don't

- Don't rely on a single tool. axe-core, Lighthouse and pa11y catch overlapping but different rules; the combined set still misses things only humans see (focus order, copy quality, plain language).
- Don't suppress axe rules with `disableRules(...)` to make CI green. Either fix the violation or document a justified exemption (linked to a WCAG technique) in the spec file.
- Don't audit only the home page. Cart, checkout and authenticated pages are where most accessibility regressions live.
- Don't re-introduce `outline: none` without a `:focus-visible` replacement. Keyboard users lose the cursor.

## Canonical examples

- [devdocs - Hummingbird accessibility](https://devdocs.prestashop-project.org/9/themes/hummingbird/accessibility/) - the accessibility primitives Hummingbird ships (skip links, landmarks, focus management, keyboard handlers).
- [devdocs - Theme accessibility guidelines](https://devdocs.prestashop-project.org/9/themes/guidelines/accessibility/) - the PrestaShop project's WCAG 2.1 AA baseline for themes.
- [WCAG 2.1 specification](https://www.w3.org/TR/WCAG21/) - normative reference for the success criteria flagged by the rules.
- [Deque axe-core](https://github.com/dequelabs/axe-core) - rule engine used under the hood by every tool above.
- [`@axe-core/playwright`](https://github.com/dequelabs/axe-core-npm/tree/develop/packages/playwright) - Playwright integration documentation.
- [pa11y](https://pa11y.org/) - alternative crawler with HTML CodeSniffer and Axe rule sets.
- [Lighthouse accessibility audit](https://developer.chrome.com/docs/lighthouse/overview) - browser-built audit with a tracked score over time.
- [`PrestaShop/hummingbird` - `src/js/accessibility/`](https://github.com/PrestaShop/hummingbird/tree/develop/src/js/accessibility) - reference TS helpers (focus trap, skip-link wiring) shipped by the upstream theme.

## Related skills

- `theme-rtl-support` - run the same audit with the front office in an RTL language to catch direction-specific regressions.
- `theme-data-ps-bindings` - axe rules check the real semantics; the data-attribute contract keeps refactors from breaking them.
- `theme-tdd-component` - colocate axe assertions inside Playwright component E2E tests for fast feedback.
- `theme-validate` - precedes the a11y audit; theme structure must validate before pages even render.
