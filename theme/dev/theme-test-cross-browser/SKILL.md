---
name: theme-test-cross-browser
description: Set up a cross-browser testing matrix (Chrome, Firefox, Safari, mobile viewports) using Playwright to verify layout integrity, JavaScript functionality, and responsive breakpoints on a PrestaShop 9 theme. Use when the user wants automated assurance that the front office works across all target browsers.
---

## Requirements

Ask the user:
* Theme technical name and root path (e.g. `themes/mytheme`).
* Running PrestaShop dev instance URL with demo data.
* Browser matrix. Default: Chromium, Firefox, WebKit (Safari). Add mobile devices as needed.
* Viewport breakpoints to test. Default: 375px (mobile), 768px (tablet), 1024px (small desktop), 1440px (large desktop).
* Critical user journeys to cover. Minimum: browse category, view product, add to cart, complete checkout, login/register.
* Whether to run on every PR or nightly. Default: PR for smoke, nightly for full matrix.

## Steps

1. Install Playwright with all browser engines:

    ```bash
    npm install --save-dev @playwright/test
    npx playwright install --with-deps
    ```

2. Configure the browser matrix in `playwright.config.ts`:

    ```ts
    // playwright.config.ts
    import { defineConfig, devices } from '@playwright/test';

    export default defineConfig({
      testDir: './tests/cross-browser',
      retries: 1,
      projects: [
        // Desktop browsers
        { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
        { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
        { name: 'webkit', use: { ...devices['Desktop Safari'] } },
        // Mobile viewports
        { name: 'mobile-chrome', use: { ...devices['Pixel 5'] } },
        { name: 'mobile-safari', use: { ...devices['iPhone 13'] } },
        // Tablet
        { name: 'tablet', use: { ...devices['iPad (gen 7)'] } },
      ],
      use: {
        baseURL: process.env.PS_BASE_URL || 'http://localhost:8001',
        trace: 'on-first-retry',
      },
    });
    ```

3. Write layout verification tests that assert critical elements are visible and correctly positioned across all browsers:

    ```ts
    // tests/cross-browser/layout.spec.ts
    import { test, expect } from '@playwright/test';

    test('header renders correctly', async ({ page }) => {
      await page.goto('/');
      const header = page.locator('[data-ps-component="header"]');
      await expect(header).toBeVisible();
      const logo = page.locator('[data-ps-element="logo"]');
      await expect(logo).toBeVisible();
    });

    test('footer renders correctly', async ({ page }) => {
      await page.goto('/');
      const footer = page.locator('footer');
      await expect(footer).toBeVisible();
    });

    test('product grid displays items', async ({ page }) => {
      await page.goto('/3-clothes');
      const products = page.locator('[data-ps-component="product-miniature"]');
      await expect(products.first()).toBeVisible();
      expect(await products.count()).toBeGreaterThan(0);
    });
    ```

4. Write responsive breakpoint tests that verify layout adapts at each breakpoint:

    ```ts
    // tests/cross-browser/responsive.spec.ts
    import { test, expect } from '@playwright/test';

    test('mobile menu toggles on small viewports', async ({ page, isMobile }) => {
      test.skip(!isMobile, 'mobile-only test');
      await page.goto('/');
      const menuToggle = page.locator('[data-ps-action="toggle-menu"]');
      await expect(menuToggle).toBeVisible();
      await menuToggle.click();
      const nav = page.locator('[data-ps-component="main-nav"]');
      await expect(nav).toBeVisible();
    });

    test('desktop shows horizontal navigation', async ({ page, isMobile }) => {
      test.skip(isMobile, 'desktop-only test');
      await page.goto('/');
      const nav = page.locator('[data-ps-component="main-nav"]');
      await expect(nav).toBeVisible();
    });
    ```

5. Write critical user journey tests (add to cart, checkout) that run across the full matrix:

    ```ts
    // tests/cross-browser/checkout.spec.ts
    import { test, expect } from '@playwright/test';

    test('add to cart and complete checkout', async ({ page }) => {
      await page.goto('/en/men/1-1-hummingbird-printed-t-shirt.html');
      await page.click('[data-ps-action="add-to-cart"]');
      await page.waitForSelector('[data-ps-component="cart-modal"]');
      await page.click('[data-ps-action="proceed-to-checkout"]');

      // Cart page
      await expect(page.locator('[data-ps-component="cart-items"]')).toBeVisible();
      await page.click('[data-ps-action="checkout"]');

      // Checkout steps
      await expect(page).toHaveURL(/order/);
    });
    ```

6. Test JavaScript functionality that is browser-sensitive (IntersectionObserver, CSS scroll-snap, Web Animations API):

    ```ts
    // tests/cross-browser/js-features.spec.ts
    import { test, expect } from '@playwright/test';

    test('lazy images load on scroll', async ({ page }) => {
      await page.goto('/3-clothes');
      const lazyImg = page.locator('img[loading="lazy"]').first();
      await lazyImg.scrollIntoViewIfNeeded();
      await expect(lazyImg).toHaveJSProperty('complete', true);
    });

    test('accordion expands on click', async ({ page }) => {
      await page.goto('/content/1-delivery');
      const trigger = page.locator('[data-ps-action="toggle-accordion"]').first();
      if (await trigger.isVisible()) {
        await trigger.click();
        const panel = page.locator('[data-ps-element="accordion-panel"]').first();
        await expect(panel).toBeVisible();
      }
    });
    ```

7. Wire into CI with a matrix strategy:

    ```yaml
    # .github/workflows/cross-browser.yml
    name: Cross-Browser Tests
    on: [pull_request]
    jobs:
      test:
        runs-on: ubuntu-latest
        strategy:
          matrix:
            project: [chromium, firefox, webkit, mobile-chrome, mobile-safari]
        steps:
          - uses: actions/checkout@v4
          - run: npm ci
          - run: npx playwright install --with-deps
          - run: docker compose up -d
          - run: npx playwright test --project=${{ matrix.project }}
          - uses: actions/upload-artifact@v4
            if: failure()
            with:
              name: traces-${{ matrix.project }}
              path: test-results/
    ```

8. Verify:
    * `npx playwright test` passes on all browser projects locally.
    * Layout elements are visible and correctly positioned in Chromium, Firefox, and WebKit.
    * Responsive tests pass at all defined breakpoints.
    * Critical user journeys (add to cart, checkout) complete without errors across the matrix.
    * CI runs the full matrix and uploads traces on failure.

## Do

- Use `data-ps-component` / `data-ps-element` / `data-ps-action` selectors rather than CSS classes. They survive BEM refactors.
- Enable Playwright traces (`trace: 'on-first-retry'`) so failures produce a browsable trace zip for debugging.
- Run WebKit tests even on Linux CI. Playwright's WebKit build covers most Safari-specific rendering quirks.
- Tag browser-specific skips with `test.skip()` and a clear reason, not blanket `fixme` annotations.

## Don't

- Don't test third-party scripts (analytics, chat widgets) in cross-browser tests. They are outside your control and cause flakiness.
- Don't rely on `waitForTimeout()` for synchronization. Use `waitForSelector()`, `expect().toBeVisible()`, or network idle.
- Don't run the full matrix on every PR if it takes longer than 10 minutes. Gate on Chromium + Firefox for PRs; run the full matrix nightly.
- Don't skip WebKit. Safari has unique flexbox, grid and scroll behavior that Chromium and Firefox do not reproduce.

## Canonical examples

- [Playwright cross-browser testing](https://playwright.dev/docs/browsers) - browser engine support and device emulation.
- [Playwright test configuration](https://playwright.dev/docs/test-configuration) - projects, retries, traces, and CI integration.
- [Playwright device descriptors](https://playwright.dev/docs/emulation#devices) - the full list of emulated devices.
- [`PrestaShop/hummingbird`](https://github.com/PrestaShop/hummingbird) - reference theme with `data-ps-*` bindings used as selectors.

## Related skills

- `theme-data-ps-bindings` - the `data-ps-*` contract ensures selectors work across all browsers without coupling to CSS.
- `theme-test-visual-regression` - visual regression catches rendering differences that functional tests miss.
- `theme-accessibility-audit` - axe-core tests can share the same Playwright instance.
- `theme-tdd-component` - component tests complement page-level cross-browser checks.
- `theme-bootstrap-compatibility` - Bootstrap version mismatches are the primary cause of cross-browser layout failures in PS themes.
