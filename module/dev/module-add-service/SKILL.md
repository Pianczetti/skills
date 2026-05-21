---
name: module-add-service
description: Register a Symfony service in a PrestaShop 9 module via config/services.yml using autowiring. Use when the user needs to inject a class as a dependency in controllers, hook listeners, or other services.
---

## Requirements

Ask the user:
* The service class FQCN under the module's PSR-4 namespace (e.g. `MyVendor\Mymodule\Service\OrderExporter`).
* Constructor dependencies. Note whether they are core PrestaShop services (e.g. `Doctrine\ORM\EntityManagerInterface`, `Symfony\Contracts\Translation\TranslatorInterface`, `PrestaShopBundle\Service\Hook\HookDispatcherInterface`) or sibling module services.
* Whether the service must be public. Default is private; mark public ONLY if the service is fetched from the container by name (e.g. by a legacy controller via `$this->module->get('...')` or a hook listener resolved by tag).
* Any tags the service must carry (`prestashop.form.identifiable_object.builder`, `kernel.event_listener`, `console.command`, ...).

## Steps

1. Place the class under `src/<SubFolder>/<Name>.php` matching its namespace. PSR-4 expects directory names to match namespace segments exactly.

    ```php
    <?php
    namespace MyVendor\Mymodule\Service;

    use Doctrine\ORM\EntityManagerInterface;
    use Symfony\Contracts\Translation\TranslatorInterface;

    class OrderExporter
    {
        public function __construct(
            private readonly EntityManagerInterface $em,
            private readonly TranslatorInterface $translator,
        ) {}

        public function export(int $orderId): string { /* ... */ }
    }
    ```

2. With the default `resource:` glob from `module-create`, the class is auto-registered, autowired and private:

    ```yaml
    services:
      _defaults:
        autowire: true
        autoconfigure: true
        public: false

      MyVendor\Mymodule\:
        resource: '../src/*'
        exclude: '../src/{Entity,DTO}'
    ```

   To override one service (make it public, add a tag, pin a constructor argument), add an explicit entry below the glob - it wins:

    ```yaml
      MyVendor\Mymodule\Service\OrderExporter:
        public: true
        arguments:
          $exportDir: '%kernel.project_dir%/var/exports/mymodule'
        tags:
          - { name: 'kernel.event_listener', event: 'kernel.terminate', method: 'onTerminate' }
    ```

3. Inject the service:

    * In a controller action, type-hint the argument: `public function indexAction(OrderExporter $exporter)`.
    * In another service, type-hint the constructor argument; autowiring resolves it.
    * From the legacy module class only when unavoidable: `$this->module->get(OrderExporter::class)` (requires `public: true`).

4. After editing `services.yml`, clear the cache: `php bin/console cache:clear`.

5. Inspect the wiring:

    ```bash
    php bin/console debug:container MyVendor\\Mymodule\\Service\\OrderExporter
    php bin/console debug:autowiring TranslatorInterface
    ```

## Do

- Use constructor injection with `readonly` properties (PHP 8.1+). Type-hint interfaces, not concrete classes, when possible.
- Keep services private unless something outside the container fetches them by ID. Autowiring removes the need for public services in most cases.
- Use the `resource:` glob plus `exclude:` for `Entity/` and `DTO/` rather than registering each service by hand.
- Tag services for cross-cutting concerns (`kernel.event_listener`, `console.command`, `prestashop.form.identifiable_object.builder`).

## Don't

- Don't fetch services from a static container (`SymfonyContainer::getInstance()`, `\Context::getContext()->get(...)`) outside legacy bridges. Inject them.
- Don't make every service `public: true` "just in case" - it leaks implementation details and breaks the autowiring contract.
- Don't register services that are abstract, traits, interfaces, or `Entity` / `DTO` classes; exclude them from the glob.
- Don't repeat dependencies in `arguments:` when autowiring already resolves them; only set `arguments:` for scalar parameters or when you need a non-default service.

## Canonical examples

- [devdocs - Module services](https://devdocs.prestashop-project.org/9/modules/concepts/services/).
- Working sample with services + form data providers: [`example-modules/demoformdataproviders`](https://github.com/PrestaShop/example-modules/tree/master/demoformdataproviders).
- Working sample with Doctrine repositories injected as services: [`example-modules/demodoctrine`](https://github.com/PrestaShop/example-modules/tree/master/demodoctrine).
