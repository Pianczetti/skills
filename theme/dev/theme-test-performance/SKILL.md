---
name: theme-test-performance
description: Run Lighthouse and Web Vitals audits on PrestaShop 9 front-office pages to enforce LCP, FID/INP, CLS budgets, audit asset sizes and render-blocking resources, and gate CI on performance regressions. Use when the user wants measurable performance baselines and automated regression detection.
---

## Requirements

Ask the user:
* Theme technical name and root path (e.g. `themes/mytheme`).
* Running PrestaShop dev instance URL with demo data loaded.
* Performance budgets. Defaults (aligned with Core Web Vitals "good" thresholds):
  * LCP < 2.5s
  * INP < 200ms (replaces FID)
  * CLS < 0.1
  * Total page weight < 1.5 MB
  * First Contentful Paint < 1.8s
  * Lighthouse Performance score >= 80
* Pages to audit. Minimum: home, category, product, cart, checkout.
* Whether to track budgets over time (Lighthouse CI server) or just fail/pass per PR.
* Device profile. Default: mobile (Moto G Power on 4G throttling) - the Lighthouse default.

## Steps

1. Install Lighthouse CI alongside the theme's dev pipeline:

    ```bash
    npm install --save-dev @lhci/cli lighthouse
    ```

2. Create a Lighthouse CI config at the theme root:

    ```js
    // lighthouserc.js
    module.exports = {
      ci: {
        collect: {
          url: [
            'http://localhost:8001/',
            'http://localhost:8001/3-clothes',
            'http://localhost:8001/en/men/1-1-hummingbird-printed-t-shirt.html',
            'http://localhost:8001/cart?action=show',
            'http://localhost:8001/order',
          ],
          numberOfRuns: 3,
          settings: {
            preset: 'desktop', // or remove for mobile (default)
            chromeFlags: '--headless --no-sandbox --ignore-certificate-errors',
          },
        },
        assert: {
          assertions: {
            'categories:performance': ['error', { minScore: 0.8 }],
            'first-contentful-paint': ['error', { maxNumericValue: 1800 }],
            'largest-contentful-paint': ['error', { maxNumericValue: 2500 }],
            'cumulative-layout-shift': ['error', { maxNumericValue: 0.1 }],
            'interactive': ['warn', { maxNumericValue: 3800 }],
            'total-byte-weight': ['error', { maxNumericValue: 1500000 }],
            'render-blocking-resources': ['warn', { minScore: 0.9 }],
            'unused-css-rules': ['warn', { minScore: 0.85 }],
            'unused-javascript': ['warn', { minScore: 0.85 }],
          },
        },
        upload: {
          target: 'temporary-public-storage', // or 'lhci' for self-hosted server
        },
      },
    };
    ```

3. Add an npm script to run the audit:

    ```json
    {
      "scripts": {
        "perf": "lhci autorun",
        "perf:collect": "lhci collect",
        "perf:assert": "lhci assert"
      }
    }
    ```

4. Audit asset bundle sizes. Add a budget file that Lighthouse reads to flag individual resources:

    ```json
    // budgets.json
    [
      {
        "path": "/*",
        "resourceSizes": [
          { "resourceType": "script", "budget": 200 },
          { "resourceType": "stylesheet", "budget": 100 },
          { "resourceType": "image", "budget": 500 },
          { "resourceType": "font", "budget": 100 },
          { "resourceType": "total", "budget": 1500 }
        ],
        "resourceCounts": [
          { "resourceType": "script", "budget": 15 },
          { "resourceType": "stylesheet", "budget": 5 }
        ]
      }
    ]
    ```

    Reference it in `lighthouserc.js` under `collect.settings.budgets`.

5. Check for render-blocking resources manually. Verify the theme's `config/theme.yml` asset declarations use proper attributes:

    * CSS critical path is inlined or `media="print"` swapped.
    * JS bundles carry `defer` or `async` (declared via `attributes: { defer: true }` in `theme.yml` assets).
    * Web fonts use `font-display: swap` and are preloaded with `<link rel="preload" as="font">`.

6. Wire into CI:

    ```yaml
    # .github/workflows/perf.yml
    name: Performance Budget
    on: [pull_request]
    jobs:
      lighthouse:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - run: npm ci
          - run: npm run build
          - run: docker compose up -d
          - run: npx lhci autorun
    ```

7. Verify:
    * `npm run perf` passes locally with all assertions green.
    * LCP, CLS and Performance score meet the defined budgets on every audited URL.
    * `total-byte-weight` is under budget for each page.
    * CI fails the build when a PR introduces a regression (larger bundle, new render-blocking resource, layout shift).

## Do

- Run at least 3 runs per URL (`numberOfRuns: 3`) and let LHCI median the results. Single runs are noisy.
- Audit on mobile throttling (Lighthouse default) even if your audience is desktop-heavy. Mobile exposes issues faster.
- Track scores over time with the LHCI server or Lighthouse Treemap so regressions are visible in charts, not just pass/fail.
- Check `unused-css-rules` and `unused-javascript` to keep the asset pipeline lean.

## Don't

- Don't run Lighthouse against a cold Docker container that is still compiling caches. Warm the instance first (`curl -s $URL > /dev/null`).
- Don't set budgets so tight that normal content changes (new product images, longer copy) cause failures. Budget against resource types and scores, not absolute byte counts of individual files.
- Don't ignore CLS on product and category pages. Image placeholders without explicit `width`/`height` are the number-one offender.
- Don't test only the home page. Checkout and product pages often carry the heaviest JS and most layout shifts.

## Canonical examples

- [Lighthouse CI documentation](https://github.com/GoogleChrome/lighthouse-ci) - full reference for `lhci autorun`, assertions, and server setup.
- [web.dev - Core Web Vitals](https://web.dev/articles/vitals) - the thresholds behind LCP, INP, CLS.
- [Lighthouse performance budgets](https://developer.chrome.com/docs/lighthouse/performance/performance-budgets) - the `budgets.json` format.
- [`PrestaShop/hummingbird`](https://github.com/PrestaShop/hummingbird) - reference theme optimized for performance (deferred JS, critical CSS, font-display swap).

## Related skills

- `theme-asset-pipeline` - the build that produces the bundles Lighthouse measures.
- `theme-accessibility-audit` - run alongside performance; Lighthouse covers both in one pass.
- `theme-test-visual-regression` - visual tests complement perf tests; both run on the same CI stack.
- `theme-validate` - structural validation before performance testing.
