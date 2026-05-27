---
name: theme-test-visual-regression
description: Set up Playwright visual regression testing for a PrestaShop 9 theme with baseline screenshots of key pages, pixel-diff comparison, and CI integration via Chromatic or self-hosted snapshots. Use when the user wants to catch unintended visual changes before they reach production.
---

## Requirements

Ask the user:
* Theme technical name and root path (e.g. `themes/mytheme`).
* Running PrestaShop dev instance URL (e.g. `http://localhost:8001`). Screenshots require a booted front office with demo data.
* Visual regression backend:
  * **Self-hosted** (Playwright `toHaveScreenshot()` with baseline PNGs committed to the repo).
  * **Chromatic** (cloud service, auto-baseline from the `main` branch, PR review UI).
* Page families in scope. Minimum: home, category, product, cart, checkout, login, my account, 404.
* Viewports to cover. Default: desktop (1440x900), tablet (768x1024), mobile (375x812).
* Acceptable pixel-diff threshold. Default: `maxDiffPixelRatio: 0.01` (1%).

## Steps

1. Install Playwright alongside the theme's dev dependencies:

    ```bash
    npm install --save-dev @playwright/test
    npx playwright install --with-deps chromium
    ```

    If using Chromatic, also add:

    ```bash
    npm install --save-dev chromatic
    ```

2. Create a Playwright config tuned for visual regression at the theme root:

    ```ts
    // playwright.config.ts
    import { defineConfig, devices } from '@playwright/test';

    export default defineConfig({
      testDir: './tests/visual',
      snapshotDir: './tests/visual/__snapshots__',
      updateSnapshots: 'missing',
      expect: {
        toHaveScreenshot: { maxDiffPixelRatio: 0.01 },
      },
      projects: [
        { name: 'desktop', use: { ...devices['Desktop Chrome'], viewport: { width: 1440, height: 900 } } },
        { name: 'mobile', use: { ...devices['iPhone 13'], viewport: { width: 375, height: 812 } } },
      ],
      use: {
        baseURL: process.env.PS_BASE_URL || 'http://localhost:8001',
        screenshot: 'only-on-failure',
      },
    });
    ```

3. Write a visual regression spec that screenshots each page family per viewport:

    ```ts
    // tests/visual/pages.spec.ts
    import { test, expect } from '@playwright/test';

    const PAGES: Array<[string, string]> = [
      ['home',     '/'],
      ['category', '/3-clothes'],
      ['product',  '/en/men/1-1-hummingbird-printed-t-shirt.html'],
      ['cart',     '/cart?action=show'],
      ['checkout', '/order'],
      ['login',    '/login'],
      ['account',  '/my-account'],
      ['404',      '/this-page-does-not-exist'],
    ];

    for (const [name, path] of PAGES) {
      test(`visual: ${name}`, async ({ page }) => {
        await page.goto(path, { waitUntil: 'networkidle' });
        // Hide dynamic content (timestamps, CSRF tokens, ads)
        await page.evaluate(() => {
          document.querySelectorAll('[data-ps-dynamic]').forEach(el => el.remove());
        });
        await expect(page).toHaveScreenshot(`${name}.png`, { fullPage: true });
      });
    }
    ```

4. Generate initial baselines by running Playwright with `--update-snapshots`:

    ```bash
    npx playwright test --update-snapshots
    ```

    Commit the resulting `tests/visual/__snapshots__/` directory. These PNGs serve as the golden reference.

5. For authenticated pages, set up a storage state fixture:

    ```ts
    // tests/visual/auth.setup.ts
    import { test as setup } from '@playwright/test';

    setup('authenticate', async ({ page }) => {
      await page.goto('/login');
      await page.fill('[name="email"]', 'pub@prestashop.com');
      await page.fill('[name="password"]', '123456789');
      await page.click('#submit-login');
      await page.context().storageState({ path: 'tests/visual/.auth/customer.json' });
    });
    ```

6. Wire into CI so visual diffs block merge:

    ```yaml
    # .github/workflows/visual.yml
    name: Visual Regression
    on: [pull_request]
    jobs:
      visual:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - run: npm ci
          - run: npx playwright install --with-deps chromium
          - run: docker compose up -d  # boot PS dev stack
          - run: npx playwright test tests/visual/
          - uses: actions/upload-artifact@v4
            if: failure()
            with:
              name: visual-diff
              path: test-results/
    ```

    For Chromatic, replace the Playwright step with:

    ```bash
    npx chromatic --project-token=$CHROMATIC_TOKEN
    ```

7. Verify:
    * `npx playwright test tests/visual/` passes locally against the running PS instance with zero diff failures.
    * CI uploads diff artifacts on failure so reviewers can inspect pixel changes.
    * Baselines update cleanly when intentional design changes are made (`--update-snapshots`).

## Do

- Hide non-deterministic content (timestamps, random product order, session tokens) before screenshotting by marking volatile elements with `data-ps-dynamic` and stripping them in the spec.
- Keep viewport list minimal (desktop + mobile). Adding tablet triples CI time with diminishing returns.
- Store baselines in the repo (`__snapshots__/`) so any contributor can regenerate locally.
- Use `fullPage: true` so below-the-fold regressions are caught.

## Don't

- Don't screenshot pages that depend on third-party widgets (live chat, analytics banners) without mocking them out. They cause flaky diffs.
- Don't set `maxDiffPixelRatio` above 5%. Large thresholds hide real regressions.
- Don't commit baselines generated on a different OS or font stack than CI. Rendering differences cause permanent flakiness.
- Don't run visual tests without demo data loaded. Empty pages produce meaningless baselines.

## Canonical examples

- [Playwright visual comparisons](https://playwright.dev/docs/test-snapshots) - the `toHaveScreenshot()` API and snapshot management.
- [Chromatic](https://www.chromatic.com/docs/) - cloud visual review service with automatic baselines.
- [`PrestaShop/hummingbird`](https://github.com/PrestaShop/hummingbird) - reference theme whose pages form the default screenshot list.

## Related skills

- `theme-accessibility-audit` - run axe-core alongside visual snapshots in the same Playwright instance.
- `theme-tdd-component` - component-level Playwright E2E tests complement full-page visual regression.
- `theme-storybook` - Storybook + Chromatic provides component-level visual regression as an alternative to full-page screenshots.
- `theme-asset-pipeline` - the build must complete before screenshots can be taken.
