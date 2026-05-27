---
name: module-add-api-tests
description: Scaffold API tests for PrestaShop 9 Admin API resources (CQRS-backed, OAuth2-authenticated) that verify JSON responses, CRUD operations, validation errors, and authorization. Use when the module exposes Admin API endpoints and the user wants automated contract testing.
---

## Requirements

Ask the user:
* Module technical name and root path (e.g. `modules/mymodule`).
* Admin API resource name(s) exposed by the module (e.g. `MyEntity`).
* OAuth2 client credentials for the test environment (client_id / client_secret with the required scopes).
* Running PrestaShop 9 dev instance URL (e.g. `http://localhost:8001`).
* Expected CRUD operations: list, get, create, partial update, delete.
* Expected JSON schema for responses (or derive from the `#[CQRSGet]` / `#[CQRSCreate]` declarations).

## Steps

1. Add API test dependencies to `composer.json`:

    ```json
    {
      "require-dev": {
        "phpunit/phpunit": "^11.5",
        "symfony/http-client": "^6.4|^7.0",
        "helmich/phpunit-json-assert": "^3.0"
      }
    }
    ```

    ```bash
    composer update --lock
    composer install
    ```

2. Create a dedicated PHPUnit config for API tests:

    ```xml
    <?xml version="1.0" encoding="UTF-8"?>
    <phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
             bootstrap="vendor/autoload.php"
             colors="true"
             cacheDirectory=".phpunit.cache">
      <testsuites>
        <testsuite name="api">
          <directory>tests/Api</directory>
        </testsuite>
      </testsuites>
      <php>
        <env name="PS_BASE_URL" value="http://localhost:8001" />
        <env name="PS_API_CLIENT_ID" value="test_client" />
        <env name="PS_API_CLIENT_SECRET" value="test_secret" />
      </php>
    </phpunit>
    ```

3. Write a base API test case that handles OAuth2 token acquisition:

    ```php
    <?php
    // tests/Api/ApiTestCase.php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Tests\Api;

    use PHPUnit\Framework\TestCase;
    use Symfony\Component\HttpClient\HttpClient;
    use Symfony\Contracts\HttpClient\HttpClientInterface;

    abstract class ApiTestCase extends TestCase
    {
        protected static ?string $accessToken = null;
        protected static ?HttpClientInterface $client = null;

        protected static function getClient(): HttpClientInterface
        {
            if (null === self::$client) {
                self::$client = HttpClient::create([
                    'base_uri' => getenv('PS_BASE_URL') ?: 'http://localhost:8001',
                ]);
            }

            return self::$client;
        }

        protected static function getAccessToken(): string
        {
            if (null === self::$accessToken) {
                $response = self::getClient()->request('POST', '/admin-api/access-token', [
                    'body' => [
                        'grant_type' => 'client_credentials',
                        'client_id' => getenv('PS_API_CLIENT_ID') ?: 'test_client',
                        'client_secret' => getenv('PS_API_CLIENT_SECRET') ?: 'test_secret',
                    ],
                ]);
                $data = json_decode($response->getContent(), true);
                self::$accessToken = $data['access_token'];
            }

            return self::$accessToken;
        }

        protected static function apiRequest(string $method, string $path, array $options = []): array
        {
            $options['headers'] = array_merge(
                $options['headers'] ?? [],
                ['Authorization' => 'Bearer ' . self::getAccessToken()]
            );

            $response = self::getClient()->request($method, '/admin-api' . $path, $options);

            return [
                'status' => $response->getStatusCode(),
                'body' => json_decode($response->getContent(false), true),
                'headers' => $response->getHeaders(false),
            ];
        }
    }
    ```

4. Write CRUD operation tests:

    ```php
    <?php
    // tests/Api/MyEntityApiTest.php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Tests\Api;

    final class MyEntityApiTest extends ApiTestCase
    {
        private static ?int $createdId = null;

        public function testCreateEntity(): void
        {
            $result = self::apiRequest('POST', '/myentities', [
                'json' => [
                    'name' => 'Test Entity',
                    'active' => true,
                ],
            ]);

            self::assertSame(201, $result['status']);
            self::assertArrayHasKey('id', $result['body']);
            self::assertSame('Test Entity', $result['body']['name']);
            self::$createdId = $result['body']['id'];
        }

        /**
         * @depends testCreateEntity
         */
        public function testGetEntity(): void
        {
            $result = self::apiRequest('GET', '/myentities/' . self::$createdId);

            self::assertSame(200, $result['status']);
            self::assertSame(self::$createdId, $result['body']['id']);
            self::assertSame('Test Entity', $result['body']['name']);
        }

        /**
         * @depends testCreateEntity
         */
        public function testListEntities(): void
        {
            $result = self::apiRequest('GET', '/myentities');

            self::assertSame(200, $result['status']);
            self::assertIsArray($result['body']);
            self::assertNotEmpty($result['body']);
        }

        /**
         * @depends testCreateEntity
         */
        public function testPartialUpdateEntity(): void
        {
            $result = self::apiRequest('PATCH', '/myentities/' . self::$createdId, [
                'json' => ['name' => 'Updated Entity'],
                'headers' => ['Content-Type' => 'application/merge-patch+json'],
            ]);

            self::assertSame(200, $result['status']);
            self::assertSame('Updated Entity', $result['body']['name']);
        }

        /**
         * @depends testPartialUpdateEntity
         */
        public function testDeleteEntity(): void
        {
            $result = self::apiRequest('DELETE', '/myentities/' . self::$createdId);

            self::assertSame(204, $result['status']);
        }

        /**
         * @depends testDeleteEntity
         */
        public function testGetDeletedEntityReturns404(): void
        {
            $result = self::apiRequest('GET', '/myentities/' . self::$createdId);

            self::assertSame(404, $result['status']);
        }
    }
    ```

5. Write validation error tests:

    ```php
    <?php
    // tests/Api/MyEntityValidationTest.php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Tests\Api;

    final class MyEntityValidationTest extends ApiTestCase
    {
        public function testCreateWithMissingRequiredFieldReturns422(): void
        {
            $result = self::apiRequest('POST', '/myentities', [
                'json' => ['active' => true], // missing 'name'
            ]);

            self::assertContains($result['status'], [400, 422]);
            self::assertArrayHasKey('errors', $result['body']);
        }

        public function testCreateWithInvalidDataReturnsError(): void
        {
            $result = self::apiRequest('POST', '/myentities', [
                'json' => ['name' => '', 'active' => true],
            ]);

            self::assertContains($result['status'], [400, 422]);
        }
    }
    ```

6. Write authorization tests:

    ```php
    <?php
    // tests/Api/MyEntityAuthorizationTest.php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Tests\Api;

    use Symfony\Component\HttpClient\HttpClient;

    final class MyEntityAuthorizationTest extends ApiTestCase
    {
        public function testRequestWithoutTokenReturns401(): void
        {
            $client = HttpClient::create([
                'base_uri' => getenv('PS_BASE_URL') ?: 'http://localhost:8001',
            ]);
            $response = $client->request('GET', '/admin-api/myentities');

            self::assertSame(401, $response->getStatusCode());
        }

        public function testRequestWithInvalidTokenReturns401(): void
        {
            $client = HttpClient::create([
                'base_uri' => getenv('PS_BASE_URL') ?: 'http://localhost:8001',
            ]);
            $response = $client->request('GET', '/admin-api/myentities', [
                'headers' => ['Authorization' => 'Bearer invalid_token_here'],
            ]);

            self::assertSame(401, $response->getStatusCode());
        }
    }
    ```

7. Run the API test suite:

    ```bash
    vendor/bin/phpunit -c phpunit.api.xml.dist --testsuite=api
    ```

8. Wire into CI:

    ```yaml
    # .github/workflows/api-tests.yml
    name: API Tests
    on: [pull_request]
    jobs:
      api:
        runs-on: ubuntu-latest
        steps:
          - uses: actions/checkout@v4
          - run: docker compose up -d  # boots PS with Admin API enabled
          - run: composer install
            working-directory: modules/mymodule
          - run: vendor/bin/phpunit -c phpunit.api.xml.dist
            working-directory: modules/mymodule
            env:
              PS_BASE_URL: http://localhost:8001
              PS_API_CLIENT_ID: ${{ secrets.TEST_API_CLIENT_ID }}
              PS_API_CLIENT_SECRET: ${{ secrets.TEST_API_CLIENT_SECRET }}
    ```

9. Verify:
    * `vendor/bin/phpunit -c phpunit.api.xml.dist` passes against a running PS 9 instance with the module installed.
    * CRUD operations create, read, update, and delete entities correctly.
    * Validation errors return proper HTTP status codes and error payloads.
    * Unauthenticated requests are rejected with 401.
    * The response JSON matches the expected schema.

## Do

- Test the full OAuth2 flow (token acquisition + authenticated request) in at least one test.
- Verify response Content-Type headers (`application/ld+json` or `application/json` depending on config).
- Test pagination parameters (`limit`, `offset`) on list endpoints.
- Clean up created entities after the test run (use `@depends` ordering with a final delete, or a `tearDownAfterClass`).

## Don't

- Don't hardcode access tokens. Always acquire them dynamically via the OAuth2 client credentials flow.
- Don't test against a production instance. Always use a dedicated test environment.
- Don't skip authorization tests. They verify the OAuth2 scopes declared in the `#[ApiResource]` attributes.
- Don't assert on auto-increment IDs. Assert on known field values instead.
- Don't leave test entities in the database. The suite must be idempotent.

## Canonical examples

- [devdocs - Admin API](https://devdocs.prestashop-project.org/9/admin-api/) - the Admin API architecture and OAuth2 flow.
- [devdocs - Admin API resources](https://devdocs.prestashop-project.org/9/admin-api/resources/) - how CQRS resources are exposed.
- [`PrestaShop/PrestaShop` - tests/Integration/ApiPlatform/](https://github.com/PrestaShop/PrestaShop/tree/develop/tests/Integration/ApiPlatform) - core API integration tests.
- [Symfony HttpClient](https://symfony.com/doc/current/http_client.html) - the HTTP client used in the test base class.

## Related skills

- `module-add-admin-api-resource` - the API resource declarations tested by this skill.
- `module-add-cqrs-command` - the commands dispatched by create/update/delete endpoints.
- `module-add-cqrs-query` - the queries dispatched by get/list endpoints.
- `module-add-unit-tests` - unit tests for the handlers themselves (no HTTP layer).
- `module-add-integration-tests` - integration tests that boot the kernel without going through HTTP.
