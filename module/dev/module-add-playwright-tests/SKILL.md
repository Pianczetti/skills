---
name: module-add-playwright-tests
description: Scaffold Playwright end-to-end UI tests for a PrestaShop 9 module's back-office pages (grids, forms, bulk actions) and front-office controllers, using the Page Object Model pattern with Mocha + Chai. Use when the user wants browser-level acceptance tests that simulate merchant or customer interaction with the module's pages.
---

## Requirements

Ask the user:
* Module technical name and root path (e.g. `modules/mymodule`).
* Which pages to test: BO listing (grid CRUD, filter, sort, bulk), BO form (create/edit), FO pages, or all.
* Whether the module uses a feature flag (routes carry `_legacy_feature_flag`). If so, tests must enable the flag in `before()`.
* Whether page objects already exist in the `@prestashop-core/ui-testing` library or need to be created alongside the campaigns.
* Running PrestaShop 9 instance URL (BO + FO) and admin credentials for the test environment.

## Steps

### 1. Directory layout

Place tests under `tests/UI/` in the module root, mirroring the core convention:

```
modules/mymodule/tests/UI/
├── campaigns/
│   └── functional/
│       └── BO/
│           └── 01_voucherCRUD.ts
│           └── 02_voucherFilterSort.ts
│           └── 03_voucherBulkActions.ts
├── pages/
│   └── BO/
│       └── boMymoduleVouchersPage.ts
│       └── boMymoduleVouchersCreatePage.ts
├── data/
│   └── FakerVoucher.ts
├── .mocharc.json
└── tsconfig.json
```

For modules that contribute page objects upstream to `@prestashop-core/ui-testing`, create them there instead of locally. For private/internal modules, the local `pages/` directory is appropriate.

### 2. Page objects (POM pattern)

Page objects encapsulate selectors and interactions. They **never assert** - they return values for the campaign to assert.

**Listing page object (`boMymoduleVouchersPage.ts`)**:

```typescript
import type {Page} from '@prestashop-core/ui-testing';

export default class BOMymoduleVouchersPage {
  // Selectors
  private readonly gridTable: string = '#mymodule_voucher_grid_table';
  private readonly filterNameInput: string = '#mymodule_voucher_code';
  private readonly filterSearchButton: string = '#mymodule_voucher_grid_search_button';
  private readonly filterResetButton: string = '#mymodule_voucher_grid_reset_button';
  private readonly bulkSelectAll: string = '#mymodule_voucher_grid_bulk_action_select_all';
  private readonly bulkActionsButton: string = '#mymodule_voucher_grid_bulk_actions_btn';
  private readonly bulkDeleteButton: string = '#mymodule_voucher_grid_bulk_action_delete_selection';
  private readonly confirmDeleteButton: string = '#mymodule_voucher_grid_confirm_modal button.btn-confirm-submit';

  // Methods - return values, never assert
  async getNumberOfElementsInGrid(page: Page): Promise<number> {
    return (await page.$$(`${this.gridTable} tbody tr`)).length;
  }

  async filterByCode(page: Page, value: string): Promise<void> {
    await page.fill(this.filterNameInput, value);
    await page.click(this.filterSearchButton);
    await page.waitForLoadState('networkidle');
  }

  async resetFilter(page: Page): Promise<void> {
    await page.click(this.filterResetButton);
    await page.waitForLoadState('networkidle');
  }

  async getTextColumn(page: Page, row: number, column: string): Promise<string> {
    return page.textContent(
      `${this.gridTable} tbody tr:nth-child(${row}) td.column-${column}`
    ) ?? '';
  }

  async goToAddNewPage(page: Page): Promise<void> {
    await page.click('[data-role="page-header-desc-mymodule_voucher-link"]');
    await page.waitForLoadState('networkidle');
  }

  async goToEditPage(page: Page, row: number): Promise<void> {
    await page.click(`${this.gridTable} tbody tr:nth-child(${row}) td.column-actions a.edit`);
    await page.waitForLoadState('networkidle');
  }

  async bulkDelete(page: Page): Promise<string> {
    await page.click(this.bulkSelectAll);
    await page.click(this.bulkActionsButton);
    await page.click(this.bulkDeleteButton);
    await page.click(this.confirmDeleteButton);
    return page.textContent('.alert-success') ?? '';
  }
}

export const boMymoduleVouchersPage = new BOMymoduleVouchersPage();
```

**Create/Edit page object (`boMymoduleVouchersCreatePage.ts`)**:

```typescript
import type {Page} from '@prestashop-core/ui-testing';
import type FakerVoucher from '../../data/FakerVoucher';

export default class BOMymoduleVouchersCreatePage {
  public readonly pageTitleCreate: string = 'Add new voucher';
  public readonly pageTitleEdit: string = 'Edit:';
  public readonly successfulCreationMessage: string = 'Successful creation';
  public readonly successfulUpdateMessage: string = 'Successful update';

  private readonly codeInput: string = '#mymodule_voucher_code';
  private readonly discountInput: string = '#mymodule_voucher_discount_percent';
  private readonly activeToggle: string = '#mymodule_voucher_active_1';
  private readonly saveButton: string = '#save-button';

  async createEditVoucher(page: Page, data: FakerVoucher): Promise<string> {
    await page.fill(this.codeInput, data.code);
    await page.fill(this.discountInput, data.discountPercent.toString());
    if (data.active) {
      await page.click(this.activeToggle);
    }
    await page.click(this.saveButton);
    return page.textContent('.alert-success') ?? '';
  }

  async getPageTitle(page: Page): Promise<string> {
    return page.textContent('h1.title') ?? '';
  }
}

export const boMymoduleVouchersCreatePage = new BOMymoduleVouchersCreatePage();
```

### 3. Faker data class

```typescript
// tests/UI/data/FakerVoucher.ts
import {faker} from '@faker-js/faker';

export default class FakerVoucher {
  public readonly code: string;
  public readonly discountPercent: number;
  public readonly active: boolean;

  constructor(data: Partial<FakerVoucher> = {}) {
    this.code = data.code ?? faker.string.alphanumeric(8).toUpperCase();
    this.discountPercent = data.discountPercent ?? faker.number.int({min: 1, max: 50});
    this.active = data.active ?? true;
  }
}
```

A bare `new FakerVoucher()` must produce a valid entity that passes all module validation.

### 4. Campaign file (CRUD example)

```typescript
// tests/UI/campaigns/functional/BO/01_voucherCRUD.ts
import testContext from '@utils/testContext';
import {expect} from 'chai';
import {
  boDashboardPage, boLoginPage,
  type BrowserContext, type Page, utilsPlaywright,
} from '@prestashop-core/ui-testing';
import {boMymoduleVouchersPage} from '../../../pages/BO/boMymoduleVouchersPage';
import {boMymoduleVouchersCreatePage} from '../../../pages/BO/boMymoduleVouchersCreatePage';
import FakerVoucher from '../../../data/FakerVoucher';

const baseContext: string = 'functional_BO_modules_mymodule_01_voucherCRUD';

describe('BO - Modules - Mymodule : CRUD Voucher', function () {
  let browserContext: BrowserContext;
  let page: Page;
  const createData: FakerVoucher = new FakerVoucher();
  const editData: FakerVoucher = new FakerVoucher({active: false});

  before(async function () {
    browserContext = await utilsPlaywright.createBrowserContext(this.browser);
    page = await utilsPlaywright.newTab(browserContext);
  });

  after(async () => {
    await utilsPlaywright.closeBrowserContext(browserContext);
  });

  it('should login in BO', async function () {
    await testContext.addContextItem(this, 'testIdentifier', 'loginBO', baseContext);
    await boLoginPage.goTo(page, global.BO.URL);
    await boLoginPage.successLogin(page, global.BO.EMAIL, global.BO.PASSWD);
    const pageTitle = await boDashboardPage.getPageTitle(page);
    expect(pageTitle).to.contains(boDashboardPage.pageTitle);
  });

  it('should go to module voucher page', async function () {
    await testContext.addContextItem(this, 'testIdentifier', 'goToVoucherPage', baseContext);
    await boDashboardPage.goToSubMenu(page, 'Modules', 'Mymodule');
    // Navigate via sidebar or direct URL
  });

  it('should go to add new voucher page', async function () {
    await testContext.addContextItem(this, 'testIdentifier', 'goToAddPage', baseContext);
    await boMymoduleVouchersPage.goToAddNewPage(page);
    const title = await boMymoduleVouchersCreatePage.getPageTitle(page);
    expect(title).to.contains(boMymoduleVouchersCreatePage.pageTitleCreate);
  });

  it('should create a voucher', async function () {
    await testContext.addContextItem(this, 'testIdentifier', 'createVoucher', baseContext);
    const result = await boMymoduleVouchersCreatePage.createEditVoucher(page, createData);
    expect(result).to.contains(boMymoduleVouchersCreatePage.successfulCreationMessage);
  });

  it('should verify the voucher exists in the grid', async function () {
    await testContext.addContextItem(this, 'testIdentifier', 'verifyCreate', baseContext);
    await boMymoduleVouchersPage.filterByCode(page, createData.code);
    const code = await boMymoduleVouchersPage.getTextColumn(page, 1, 'code');
    expect(code).to.equal(createData.code);
    await boMymoduleVouchersPage.resetFilter(page);
  });

  it('should go to edit the voucher', async function () {
    await testContext.addContextItem(this, 'testIdentifier', 'goToEditPage', baseContext);
    await boMymoduleVouchersPage.filterByCode(page, createData.code);
    await boMymoduleVouchersPage.goToEditPage(page, 1);
    const title = await boMymoduleVouchersCreatePage.getPageTitle(page);
    expect(title).to.contains(boMymoduleVouchersCreatePage.pageTitleEdit);
  });

  it('should edit the voucher', async function () {
    await testContext.addContextItem(this, 'testIdentifier', 'editVoucher', baseContext);
    const result = await boMymoduleVouchersCreatePage.createEditVoucher(page, editData);
    expect(result).to.contains(boMymoduleVouchersCreatePage.successfulUpdateMessage);
  });
});
```

### 5. Test runner configuration

```json
// tests/UI/.mocharc.json
{
  "require": ["ts-node/register"],
  "extension": ["ts"],
  "spec": "campaigns/**/*.ts",
  "timeout": 60000,
  "slow": 10000
}
```

```json
// tests/UI/tsconfig.json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "strict": true,
    "esModuleInterop": true,
    "resolveJsonModule": true,
    "baseUrl": ".",
    "paths": {
      "@utils/*": ["../../tests/UI/utils/*"],
      "@prestashop-core/ui-testing": ["../../tests/UI/node_modules/@prestashop-core/ui-testing"]
    }
  }
}
```

### 6. Package dependencies

```json
// tests/UI/package.json
{
  "name": "mymodule-ui-tests",
  "private": true,
  "devDependencies": {
    "@playwright/test": "^1.40",
    "@prestashop-core/ui-testing": "latest",
    "@faker-js/faker": "^8.0",
    "chai": "^4.3",
    "mocha": "^10.0",
    "ts-node": "^10.9",
    "typescript": "^5.0"
  },
  "scripts": {
    "test": "mocha",
    "test:headed": "HEADLESS=false mocha"
  }
}
```

### 7. Running tests

```bash
# Install dependencies
npm install --prefix modules/mymodule/tests/UI

# Run headless (CI)
npm test --prefix modules/mymodule/tests/UI

# Run headed (development)
npm run test:headed --prefix modules/mymodule/tests/UI
```

### 8. Feature-flag gating (conditional)

If the module's page is behind a feature flag, enable it in `before()`:

```typescript
import {enableFeatureFlag, disableFeatureFlag} from '@commonTests/BO/advancedParameters/featureFlag';

// Before all tests
enableFeatureFlag('mymodule_new_page', baseContext);

// After all tests (cleanup)
disableFeatureFlag('mymodule_new_page', baseContext);
```

## Do

- **Every `it()` step** must call `testContext.addContextItem(this, 'testIdentifier', 'uniqueId', baseContext)` first - this tracks results and enables retry/filtering.
- Use `function()` (not arrow functions) in `describe()` blocks - Mocha needs `this` context.
- Assert the success flash after every create/edit/delete - never assume success.
- Use page object methods for all interactions; never write raw selectors in campaigns.
- Clean up created entities in `after()` to leave the system in its pre-test state.
- For toggle/quick-edit actions (AJAX), assert the status change directly on the grid row without page reload.
- For position/drag-and-drop, reload after the drag and verify the new order persisted to the database.
- Keep `testIdentifier` values globally unique across all campaigns by prefixing with `baseContext`.

## Don't

- Don't assert inside page objects. They return values (strings, booleans, numbers) for campaigns to assert.
- Don't use `page.waitForTimeout()` for synchronization - use `page.waitForLoadState('networkidle')` or specific selectors.
- Don't hardcode admin credentials in campaigns - read from `global.BO.EMAIL` / `global.BO.PASSWD`.
- Don't duplicate setup/teardown logic - check `tests/UI/commonTests/` for existing reusable describe blocks (e.g. `createProductTest`, `deleteProductTest`).
- Don't use random Faker data in assertions. Generate test data once, then assert against those known values.
- Don't skip `after()` cleanup. Tests that leave entities behind break subsequent runs.

## Canonical examples

- [PrestaShop UI Testing Library](https://github.com/PrestaShop/ui-testing-library) - page objects, Faker data, utilities.
- Core campaigns: `tests/UI/campaigns/functional/BO/11_international/03_taxes/01_taxes/02_CRUDTaxesInBO.ts` (simple CRUD), `01_filterTaxes.ts` (filter/sort).
- Common tests: `tests/UI/commonTests/BO/` (reusable setup/teardown blocks).
- [Playwright docs](https://playwright.dev/docs/intro) for the underlying browser automation API.

## Related skills

- `module-add-behat-tests` - behavioural tests that drive CQRS handlers without a browser (complementary: Behat tests the domain, Playwright tests the UI).
- `module-add-grid` - the Grid the listing page object interacts with.
- `module-add-admin-controller-modern` - the controller that serves the pages under test.
- `module-add-front-controller` - FO pages to test from the customer's perspective.
- `theme-test-visual-regression` - visual screenshot comparison for theme components (complementary focus).
