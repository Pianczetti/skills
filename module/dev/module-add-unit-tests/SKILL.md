---
name: module-add-unit-tests
description: Scaffold PHPUnit inside a PrestaShop 9 module so handlers, value objects and pure-PHP services can be unit-tested in isolation, without booting the shop. Use when the user wants `vendor/bin/phpunit` to run green from the module root.
---

## Requirements

Ask the user:
* Module technical name and root path (e.g. `modules/mymodule`).
* PHP namespace under test (typically `MyVendor\Mymodule\...`). Tests mirror this layout under `tests/Unit/`.
* Whether the test suite is **pure unit** (no PrestaShop runtime, no database) or **integration** (needs a booted core / a test database). Default to pure unit and keep core-coupled tests in a separate `tests/Integration/` folder.
* Whether the module already ships a `composer.json`. If not, see `module-create` first.
* Target PHPUnit version. PHP 8.1+ runs PHPUnit 10.x; PHP 8.2+ runs PHPUnit 11.x. Match what the host PrestaShop CI uses.

## Steps

1. Add PHPUnit to `require-dev` in the module's `composer.json` and reinstall the autoloader. Keep the dev dependency out of the production zip by running `composer install --no-dev` at packaging time (see `module-package-zip`):

    ```json
    {
      "name": "myvendor/mymodule",
      "require-dev": {
        "phpunit/phpunit": "^11.5"
      },
      "autoload": {
        "psr-4": { "MyVendor\\Mymodule\\": "src/" }
      },
      "autoload-dev": {
        "psr-4": { "MyVendor\\Mymodule\\Tests\\": "tests/" }
      }
    }
    ```

    ```bash
    cd modules/mymodule
    composer update --lock
    composer install
    ```

2. Drop a `phpunit.xml.dist` at the module root with one `unit` test suite pointed at `tests/Unit/`. The `bootstrap` attribute loads the module's own `vendor/autoload.php` for pure unit tests; switch it to `<prestashop_root>/config/bootstrap.php` only when a test genuinely needs core classes (legacy `ObjectModel`, `Configuration`, `Db`, etc.):

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
             bootstrap="vendor/autoload.php"
             colors="true"
             cacheDirectory=".phpunit.cache"
             failOnRisky="true"
             failOnWarning="true">
      <testsuites>
        <testsuite name="unit">
          <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="integration">
          <directory>tests/Integration</directory>
        </testsuite>
      </testsuites>
      <source>
        <include>
          <directory>src</directory>
        </include>
        <exclude>
          <directory>vendor</directory>
          <directory>_dev</directory>
          <directory>tests</directory>
        </exclude>
      </source>
    </phpunit>
    ```

3. Mirror the production namespace under `tests/Unit/<MirroredNamespace>/<Class>Test.php`. CQRS commands, value objects and pure domain services should not need any PrestaShop class to load:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Tests\Unit\Domain\Voucher\Command;

    use MyVendor\Mymodule\Domain\Voucher\Command\CreateVoucherCommand;
    use MyVendor\Mymodule\Domain\Voucher\Exception\VoucherConstraintException;
    use PHPUnit\Framework\TestCase;

    final class CreateVoucherCommandTest extends TestCase
    {
        public function testRejectsDiscountOutOfRange(): void
        {
            $this->expectException(VoucherConstraintException::class);
            new CreateVoucherCommand('SUMMER25', 0, new \DateTimeImmutable('+30 days'));
        }

        public function testHoldsImmutableState(): void
        {
            $cmd = new CreateVoucherCommand('SUMMER25', 25, new \DateTimeImmutable('2030-01-01'));
            self::assertSame('SUMMER25', $cmd->code);
            self::assertSame(25, $cmd->discountPercent);
        }
    }
    ```

4. For handlers, fake every collaborator. Inject doubles of `EntityManagerInterface`, repositories, `HookDispatcherInterface`, `CommandBusInterface`, the `Configuration` adapter, etc. The handler should never reach for static globals such as `Context::getContext()`, `Db::getInstance()` or `Configuration::get(...)` directly:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Tests\Unit\Domain\Voucher\CommandHandler;

    use Doctrine\ORM\EntityManagerInterface;
    use MyVendor\Mymodule\Domain\Voucher\Command\CreateVoucherCommand;
    use MyVendor\Mymodule\Domain\Voucher\CommandHandler\CreateVoucherHandler;
    use MyVendor\Mymodule\Domain\Voucher\ValueObject\VoucherId;
    use PHPUnit\Framework\TestCase;

    final class CreateVoucherHandlerTest extends TestCase
    {
        public function testPersistsAndReturnsId(): void
        {
            $em = $this->createMock(EntityManagerInterface::class);
            $em->expects(self::once())->method('persist');
            $em->expects(self::once())->method('flush');

            $handler = new CreateVoucherHandler($em);
            $id = $handler->handle(new CreateVoucherCommand('SUMMER25', 25, new \DateTimeImmutable('2030-01-01')));

            self::assertInstanceOf(VoucherId::class, $id);
        }
    }
    ```

5. Run the suite from the module root. CI does the same:

    ```bash
    cd modules/mymodule
    vendor/bin/phpunit --testsuite=unit
    vendor/bin/phpunit --testsuite=unit --coverage-text   # requires Xdebug or pcov
    ```

6. Integration tests that genuinely need PrestaShop classes go under `tests/Integration/`. Point the suite's bootstrap at the host `config/bootstrap.php` via a separate `phpunit.integration.xml.dist`, or set the path through `PS_ROOT_DIR` in the CI workflow. Refer to the canonical patterns in the module testing docs and the core's CI workflow before writing integration tests; pure unit tests should remain the bulk of the suite.

## Do

- Keep CQRS handlers, value objects, DTOs and form data providers fully unit-testable. They take collaborators through the constructor and never call static globals.
- Mirror the production namespace exactly under `tests/Unit/`. The PSR-4 `autoload-dev` block makes tests autoload without manual `require_once`.
- Mock collaborators with `createMock(...)` or hand-rolled fakes. Mocks make intent visible; real Doctrine / real `Db::getInstance()` belong in integration tests.
- Run `vendor/bin/phpunit` from the module root before every commit. Wire the same command into `.github/workflows/php.yml` so CI matches local.
- Keep `failOnRisky="true"` and `failOnWarning="true"` in `phpunit.xml.dist`. They turn silent deprecations into build failures.

## Don't

- Don't couple unit tests to `Context::getContext()`, `Db::getInstance()`, `Configuration::get(...)` or `Tools::*`. Inject a fake or a typed adapter; the handler should not know whether it runs inside a shop.
- Don't include `tests/` or `phpunit.xml.dist` in the published zip. Both are stripped at packaging time (`module-package-zip`).
- Don't run `doctrine:schema:update --force` or hit a real database from a unit test. That's an integration concern and belongs to a separate suite.
- Don't ship PHPUnit in `require` (it must stay in `require-dev`). The production autoloader has no business loading test code.
- Don't write tests against `ObjectModel` subclasses for new code. They're untestable in isolation by design; refactor the logic into a handler or a plain service first.

## Canonical examples

- [devdocs - Modules / Testing](https://devdocs.prestashop-project.org/9/modules/testing/) - the entry point for module test guidance.
- [devdocs - Modules / Basic checks](https://devdocs.prestashop-project.org/9/modules/testing/basic-checks/) and [Advanced checks](https://devdocs.prestashop-project.org/9/modules/testing/advanced-checks/) - validations every module suite should run.
- [devdocs - Modules / CI-CD](https://devdocs.prestashop-project.org/9/modules/testing/ci-cd/) - GitHub Actions templates for module repos.
- [devdocs - Testing / Unit tests](https://devdocs.prestashop-project.org/9/testing/unit-tests/) and [How to create your own unit tests](https://devdocs.prestashop-project.org/9/testing/unit-tests/how-to-create-your-own-unit-tests/) and [How to execute](https://devdocs.prestashop-project.org/9/testing/unit-tests/how-to-execute-tests/) - the patterns the core itself uses.
- [PHPUnit configuration reference](https://docs.phpunit.de/en/11.5/configuration.html) - every `phpunit.xml.dist` attribute documented.
- [`PrestaShop/PrestaShop` - composer.json](https://github.com/PrestaShop/PrestaShop/blob/develop/composer.json) and [`.github/workflows/php.yml`](https://github.com/PrestaShop/PrestaShop/blob/develop/.github/workflows/php.yml) - reference for `require-dev` PHPUnit version and CI invocation.

## Related skills

- `module-add-cqrs-command` and `module-add-cqrs-query` - the handlers you want to unit-test.
- `module-add-doctrine-entity` - integration tests cover the persistence layer.
- `module-validate` - the lint/install/uninstall checks that complement PHPUnit.
- `module-package-zip` - strips `tests/` and `phpunit.xml.dist` from the distributed zip.
