---
name: module-add-integration-tests
description: Scaffold integration and functional tests that boot the PrestaShop 9 kernel, test CQRS handlers against a real database, verify controllers return correct responses, and validate hook dispatching with the full service container. Use when the user needs confidence that module code works within the running shop, beyond isolated unit tests.
---

## Requirements

Ask the user:
* Module technical name and root path (e.g. `modules/mymodule`).
* PHP namespace (e.g. `MyVendor\Mymodule`).
* PrestaShop root path (e.g. `/var/www/html` or the Docker-mounted path).
* Whether the test database is a dedicated one (recommended) or the dev database. Tests will truncate tables they own.
* Test framework preference: **PHPUnit with KernelTestCase** (default) or **Behat** for BDD-style scenarios. Both are supported.
* Which layers to cover: CQRS handlers, controllers, hook listeners, Doctrine repositories, form handlers.

## Steps

1. Add integration test dependencies to the module's `composer.json`:

    ```json
    {
      "require-dev": {
        "phpunit/phpunit": "^11.5",
        "symfony/test-pack": "^1.0"
      },
      "autoload-dev": {
        "psr-4": {
          "MyVendor\\Mymodule\\Tests\\": "tests/"
        }
      }
    }
    ```

    ```bash
    composer update --lock
    composer install
    ```

2. Create a dedicated PHPUnit config for integration tests at `phpunit.integration.xml.dist`. The bootstrap must load PrestaShop's kernel:

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
             bootstrap="tests/Integration/bootstrap.php"
             colors="true"
             cacheDirectory=".phpunit.cache"
             failOnRisky="true"
             failOnWarning="true">
      <testsuites>
        <testsuite name="integration">
          <directory>tests/Integration</directory>
        </testsuite>
      </testsuites>
      <php>
        <env name="PS_ROOT_DIR" value="/var/www/html" />
      </php>
    </phpunit>
    ```

3. Write the integration test bootstrap that boots the PrestaShop kernel:

    ```php
    <?php
    // tests/Integration/bootstrap.php
    declare(strict_types=1);

    $psRootDir = getenv('PS_ROOT_DIR') ?: '/var/www/html';
    require_once $psRootDir . '/config/config.inc.php';

    // Boot Symfony kernel for service container access
    if (file_exists($psRootDir . '/app/AppKernel.php')) {
        require_once $psRootDir . '/app/AppKernel.php';
    }
    ```

4. Create a base test case class that provides access to the service container:

    ```php
    <?php
    // tests/Integration/IntegrationTestCase.php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Tests\Integration;

    use PHPUnit\Framework\TestCase;
    use PrestaShop\PrestaShop\Core\CommandBus\CommandBusInterface;
    use Symfony\Component\DependencyInjection\ContainerInterface;

    abstract class IntegrationTestCase extends TestCase
    {
        protected static ?ContainerInterface $container = null;

        protected static function getContainer(): ContainerInterface
        {
            if (null === self::$container) {
                global $kernel;
                if (null === $kernel) {
                    $kernel = new \AppKernel('test', true);
                    $kernel->boot();
                }
                self::$container = $kernel->getContainer();
            }

            return self::$container;
        }

        protected static function getCommandBus(): CommandBusInterface
        {
            return self::getContainer()->get('prestashop.core.command_bus');
        }
    }
    ```

5. Write CQRS handler integration tests that hit the real database:

    ```php
    <?php
    // tests/Integration/Domain/Handler/CreateEntityHandlerTest.php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Tests\Integration\Domain\Handler;

    use MyVendor\Mymodule\Domain\Command\CreateEntityCommand;
    use MyVendor\Mymodule\Domain\Query\GetEntityQuery;
    use MyVendor\Mymodule\Tests\Integration\IntegrationTestCase;

    final class CreateEntityHandlerTest extends IntegrationTestCase
    {
        public function testCreateAndRetrieveEntity(): void
        {
            $bus = self::getCommandBus();
            $id = $bus->handle(new CreateEntityCommand('Test Entity', true));

            self::assertNotNull($id);

            $result = $bus->handle(new GetEntityQuery($id));
            self::assertSame('Test Entity', $result->name);
        }

        protected function tearDown(): void
        {
            // Clean up test data
            $connection = self::getContainer()->get('doctrine.dbal.default_connection');
            $connection->executeStatement('DELETE FROM ps_mymodule_entity WHERE name = :name', ['name' => 'Test Entity']);
        }
    }
    ```

6. Write controller integration tests using Symfony's WebTestCase pattern:

    ```php
    <?php
    // tests/Integration/Controller/AdminControllerTest.php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Tests\Integration\Controller;

    use MyVendor\Mymodule\Tests\Integration\IntegrationTestCase;
    use Symfony\Component\HttpFoundation\Request;

    final class AdminControllerTest extends IntegrationTestCase
    {
        public function testIndexActionReturns200(): void
        {
            $controller = self::getContainer()->get('myvendor.mymodule.controller.admin');
            $request = Request::create('/admin-dev/mymodule/list');
            $request->attributes->set('_route', 'admin_mymodule_list');

            $response = $controller->indexAction($request);

            self::assertSame(200, $response->getStatusCode());
        }
    }
    ```

7. Write hook dispatching tests that verify the module responds to hooks with correct output:

    ```php
    <?php
    // tests/Integration/Hook/HookDispatcherTest.php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Tests\Integration\Hook;

    use MyVendor\Mymodule\Tests\Integration\IntegrationTestCase;
    use PrestaShop\PrestaShop\Core\Hook\HookDispatcherInterface;

    final class HookDispatcherTest extends IntegrationTestCase
    {
        public function testDisplayHookRendersOutput(): void
        {
            /** @var HookDispatcherInterface $dispatcher */
            $dispatcher = self::getContainer()->get('prestashop.core.hook.dispatcher');
            $result = $dispatcher->dispatchWithParameters('displayHome', ['module' => 'mymodule']);

            self::assertIsArray($result);
        }
    }
    ```

8. Run the integration suite from the module root (requires a running PS instance):

    ```bash
    vendor/bin/phpunit -c phpunit.integration.xml.dist --testsuite=integration
    ```

9. Wire into CI with a Docker-based PS instance:

    ```yaml
    # .github/workflows/integration.yml
    name: Integration Tests
    on: [pull_request]
    jobs:
      integration:
        runs-on: ubuntu-latest
        services:
          mysql:
            image: mysql:8.0
            env:
              MYSQL_ROOT_PASSWORD: prestashop
              MYSQL_DATABASE: prestashop_test
        steps:
          - uses: actions/checkout@v4
          - run: docker compose -f docker-compose.test.yml up -d
          - run: composer install
            working-directory: modules/mymodule
          - run: vendor/bin/phpunit -c phpunit.integration.xml.dist
            working-directory: modules/mymodule
    ```

10. Verify:
    * `vendor/bin/phpunit -c phpunit.integration.xml.dist` passes against a running PS instance.
    * CQRS handlers persist and retrieve data from the real database.
    * Controllers return expected HTTP status codes.
    * Hook dispatching produces expected output.
    * Test data is cleaned up after each test run.

## Do

- Use a dedicated test database or prefix test data so it can be reliably cleaned up.
- Boot the kernel once per test class (not per test method) for performance. Use `setUpBeforeClass()`.
- Test the full command bus dispatch path, including middleware (validation, multistore scoping).
- Keep integration tests in `tests/Integration/` separate from unit tests. They have different bootstrap requirements and run times.

## Don't

- Don't mock the database or command bus in integration tests. The point is to test the real wiring.
- Don't leave test data behind. Always clean up in `tearDown()` or use database transactions that roll back.
- Don't run integration tests in CI without a properly installed PS instance. A bare MySQL is not enough; the shop tables must exist.
- Don't test trivial getters/setters here. Those belong in unit tests.
- Don't depend on specific auto-increment IDs. Query by known attributes instead.

## Canonical examples

- [devdocs - Testing / Integration tests](https://devdocs.prestashop-project.org/9/testing/integration-tests/) - core integration test patterns.
- [devdocs - Modules / Testing](https://devdocs.prestashop-project.org/9/modules/testing/) - module test entry point.
- [`PrestaShop/PrestaShop` - tests/Integration/](https://github.com/PrestaShop/PrestaShop/tree/develop/tests/Integration) - reference integration tests from core.
- [Symfony Testing - KernelTestCase](https://symfony.com/doc/current/testing.html#integration-tests) - the base pattern for service container tests.

## Related skills

- `module-add-unit-tests` - pure unit tests that do not need the kernel.
- `module-add-cqrs-command` - the handlers tested by integration tests.
- `module-add-cqrs-query` - query handlers validated against a real database.
- `module-add-doctrine-entity` - the persistence layer under test.
- `module-validate` - structural validation before running the test suite.
