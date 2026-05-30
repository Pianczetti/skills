---
name: module-add-behat-tests
description: Scaffold Behat (BDD) behavioural tests for a PrestaShop 9 module that exercise its CQRS command and query handlers through Gherkin scenarios against a real database, with no browser. Use when the user wants executable, business-readable specifications for module behaviour (CRUD, bulk actions, constraint/error cases) that complement isolated PHPUnit unit tests.
---

## Requirements

Ask the user:
* Module technical name, root path (e.g. `modules/mymodule`) and PHP namespace (e.g. `MyVendor\Mymodule`).
* PrestaShop root path with dev dependencies installed - Behat runs against a real, installed shop, not a bare MySQL.
* Which aggregate(s) to cover and their existing CQRS surface (commands, queries, typed exceptions). Behat drives the same bus the controllers use - see `module-add-cqrs-command` / `module-add-cqrs-query`.
* Whether the aggregate has bulk actions (see `module-add-cqrs-bulk-command`) so the scenarios cover the partial-failure path.
* Whether the test database is dedicated (recommended). Scenarios restore tables between runs, so they must not point at a real store.

## Steps

1. Add Behat to the module's `composer.json` dev dependencies and a PSR-4 mapping for the test contexts:

    ```json
    {
      "require-dev": {
        "behat/behat": "^3.14"
      },
      "autoload-dev": {
        "psr-4": {
          "MyVendor\\Mymodule\\Tests\\": "tests/"
        }
      }
    }
    ```

    ```bash
    composer update --lock && composer install
    ```

2. Lay out the Behat tree under the module. Keep feature files (the spec) and contexts (the PHP step glue) separate, mirroring core's `tests/Integration/Behaviour` layout:

    ```
    modules/mymodule/tests/Behaviour/
    ├── behat.yml
    ├── Features/
    │   └── Scenario/Voucher/voucher_management.feature
    └── Features/
        └── Context/Domain/Voucher/VoucherFeatureContext.php
    ```

3. Write the feature context. Extend core's `AbstractDomainFeatureContext` (available when the suite runs inside an installed shop with dev deps) so you inherit the command bus, query bus and shared storage. Drive writes through `getCommandBus()`, reads through `getQueryBus()`, and resolve string references to ids with `referenceToId()`:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Tests\Behaviour\Features\Context\Domain\Voucher;

    use Behat\Gherkin\Node\TableNode;
    use MyVendor\Mymodule\Domain\Voucher\Command\AddVoucherCommand;
    use MyVendor\Mymodule\Domain\Voucher\Command\DeleteVoucherCommand;
    use MyVendor\Mymodule\Domain\Voucher\Exception\VoucherConstraintException;
    use MyVendor\Mymodule\Domain\Voucher\Query\GetVoucherForEditing;
    use PrestaShop\PrestaShop\Tests\Integration\Behaviour\Features\Context\Domain\AbstractDomainFeatureContext;
    use PHPUnit\Framework\Assert;

    class VoucherFeatureContext extends AbstractDomainFeatureContext
    {
        /**
         * @When I add a voucher :reference with following properties:
         */
        public function addVoucher(string $reference, TableNode $node): void
        {
            $data = $node->getRowsHash();
            $voucherId = $this->getCommandBus()->handle(new AddVoucherCommand(
                $data['code'],
                (int) $data['discount_percent'],
                (bool) $data['active'],
            ));
            // Store the new id under the reference so later steps resolve it.
            $this->getSharedStorage()->set($reference, $voucherId->getValue());
        }

        /**
         * @Then voucher :reference should have the following properties:
         */
        public function assertVoucherProperties(string $reference, TableNode $node): void
        {
            // Stateless: load fresh from the DB, never trust state from a previous step.
            $result = $this->getQueryBus()->handle(new GetVoucherForEditing($this->referenceToId($reference)));
            $expected = $node->getRowsHash();

            Assert::assertSame($expected['code'], $result->getCode());
            Assert::assertSame((bool) $expected['active'], $result->isActive());
        }

        /**
         * @When I delete voucher :reference
         */
        public function deleteVoucher(string $reference): void
        {
            $this->getCommandBus()->handle(new DeleteVoucherCommand($this->referenceToId($reference)));
        }
    }
    ```

4. Implement error scenarios with the **capture-then-assert** pattern: the `@When` step catches the typed domain exception and stores it; the very next `@Then` step asserts it. Never `try/catch` and ignore - core's `@AfterStep` hook re-throws an unasserted captured exception, and `@BeforeScenario` clears leftovers:

    ```php
    /**
     * @When I add a voucher :reference with an empty code
     */
    public function addVoucherWithEmptyCode(string $reference): void
    {
        try {
            $this->getCommandBus()->handle(new AddVoucherCommand('', 10, true));
        } catch (VoucherConstraintException $e) {
            $this->setLastException($e);
        }
    }

    /**
     * @Then I should get an error that the voucher code is invalid
     */
    public function assertCodeError(): void
    {
        $this->assertLastErrorIs(VoucherConstraintException::class);
    }
    ```

5. Write the Gherkin feature file. Keep steps stateless (action and assertion are self-contained), use string references instead of hardcoded ids, and tag the feature so the database is restored before it runs:

    ```gherkin
    @restore-all-tables-before-feature
    Feature: Voucher management
      As an employee
      I want to manage vouchers
      So that I can offer discounts

      Scenario: Add a new voucher
        When I add a voucher "voucher1" with following properties:
          | code            | SUMMER2026 |
          | discount_percent| 15         |
          | active          | true       |
        Then voucher "voucher1" should have the following properties:
          | code   | SUMMER2026 |
          | active | true       |

      Scenario: Cannot add a voucher with an empty code
        When I add a voucher "bad_voucher" with an empty code
        Then I should get an error that the voucher code is invalid

      Scenario: Delete a voucher
        When I add a voucher "voucher1" with following properties:
          | code            | TODELETE |
          | discount_percent| 5        |
          | active          | true     |
        And I delete voucher "voucher1"
        Then voucher "voucher1" should not exist
    ```

    For an aggregate with bulk actions, add a scenario that asserts the collect-and-continue semantics (see `module-add-cqrs-bulk-command`):

    ```gherkin
      Scenario: Bulk delete vouchers
        Given the following vouchers exist:
          | reference | code   |
          | voucher1  | FIRST  |
          | voucher2  | SECOND |
          | voucher3  | THIRD  |
        When I bulk delete vouchers "voucher1,voucher2"
        Then voucher "voucher1" should not exist
        And voucher "voucher2" should not exist
        And voucher "voucher3" should exist
    ```

6. Register the context in the module's `behat.yml`. The path/namespace points the suite at your context class:

    ```yaml
    # tests/Behaviour/behat.yml
    default:
      suites:
        mymodule:
          paths:
            - '%paths.base%/Features/Scenario'
          contexts:
            - MyVendor\Mymodule\Tests\Behaviour\Features\Context\Domain\Voucher\VoucherFeatureContext
    ```

7. Verify step wiring without executing scenarios, then run the suite against the installed shop from the PrestaShop root:

    ```bash
    # From the module: confirm every Gherkin step matches a definition
    vendor/bin/behat -c tests/Behaviour/behat.yml --dry-run

    # From the PrestaShop root: run for real against the test database
    php vendor/bin/behat -c modules/mymodule/tests/Behaviour/behat.yml
    ```

8. (Optional) Wire into CI behind a fully installed PS instance (a bare MySQL is not enough - the shop tables and module must exist). Reuse the Docker-based job from `module-add-integration-tests`, swapping the command for the Behat run above.

## Do

- Keep every step **stateless**: the assertion step independently loads the entity from the database. Never store a query result in memory in one step for a later step to assert.
- Use string references (`"voucher1"`) and `referenceToId()` / `referencesToIds()` to resolve them - never hardcode integer ids in steps or scenarios.
- Drive behaviour through the command and query buses, exactly as the controllers do. This is what makes the scenario a real behavioural test of the module's domain layer.
- Pair every exception capture with an assertion in the next step (`setLastException()` then `assertLastErrorIs()`). Catch the **typed** domain exception, never `\Exception`.
- Tag state-mutating features with `@restore-all-tables-before-feature` so scenarios cannot leak data into each other or into a real store.
- Reuse existing core step definitions for fixtures (creating a currency, language, etc.) instead of re-implementing them.

## Don't

- Don't write stateful step pairs like `When I add a voucher` / `Then the last created voucher should...`. The assertion must name the reference and reload it.
- Don't `try/catch` a domain exception and swallow it. An unasserted captured exception is re-thrown by core's `@AfterStep` hook by design - always assert it.
- Don't point Behat at a production or shared dev database. Scenarios restore tables and will wipe data.
- Don't duplicate pure-logic coverage here. Value-object validation and branchy service logic belong in `module-add-unit-tests`; Behat covers behaviour through the bus.
- Don't assert UI or rendered HTML in Behat - that is the browser layer's job (use Playwright via the theme testing skills). Behat stops at the domain/CQRS boundary.

## Canonical examples

- [devdocs - Behaviour tests (Behat)](https://devdocs.prestashop-project.org/9/testing/behaviour-tests/) and [Modules / Testing](https://devdocs.prestashop-project.org/9/modules/testing/).
- Core feature contexts and scenarios: [`tests/Integration/Behaviour/Features`](https://github.com/PrestaShop/PrestaShop/tree/develop/tests/Integration/Behaviour/Features) under `PrestaShop/PrestaShop` - start from the `Tax` domain (simple) and `Manufacturer` (sub-resources).
- The shared base class: `AbstractDomainFeatureContext` (command bus, query bus, shared storage) and `CommonFeatureContext` (the `@restore-all-tables-*`, `@clear-cache-*` and exception-safety hooks).
- [Behat documentation](https://docs.behat.org/en/latest/) for Gherkin syntax and context/step-definition mechanics.

## Related skills

- `module-add-cqrs-command` and `module-add-cqrs-query` - the handlers these scenarios drive through the bus.
- `module-add-cqrs-bulk-command` - the bulk handler whose partial-failure behaviour the bulk scenario covers.
- `module-add-unit-tests` - isolated logic tests (no kernel, no DB) that complement these behavioural tests.
- `module-add-integration-tests` - kernel-booting PHPUnit tests and the reusable Docker CI job.
- `module-add-doctrine-entity` - the persistence layer the scenarios read from and write to.
