---
name: module-add-webservice-resource-legacy
description: Expose an entity through PrestaShop 9's legacy /api/ webservice by hooking addWebserviceResources from the module class. Use ONLY when integrating with a third-party that has not migrated to the modern Admin API.
---

> **The Admin API is the modern surface.** Only use this skill when integrating with merchants who depend on the legacy `/api/` webservice or a third-party tool that has not migrated. For any new integration, start with [`module-add-admin-api-resource`](../module-add-admin-api-resource/SKILL.md) instead.

## Requirements

Ask the user:
* Resource name in `snake_case` plural, used as the URL segment (e.g. `articles`, `vouchers`). It becomes `https://<shop>/api/<resource>`.
* Entity class name and FQCN. The legacy webservice expects an `ObjectModel`-based class; entities written exclusively as Doctrine ORM cannot be exposed this way without a `WebserviceSpecificManagementInterface` adapter.
* Allowed CRUD methods (`GET`, `POST`, `PUT`, `DELETE`). Any method not allowed is listed under `forbidden_method`.
* Field list and which fields are translatable. Translatable fields require a `<table>_lang` companion and `multilang => true` in the entity definition.
* Whether the resource needs custom serialization or non-CRUD behavior. If yes, plan a `WebserviceSpecificManagementInterface` implementation.

## Steps

1. Define the entity as a legacy `ObjectModel` and declare BOTH the persistence definition (`$definition`) and the webservice exposure (`$webserviceParameters`). The webservice serialiser reads the second one to decide which fields to expose:

    ```php
    <?php
    // modules/wsarticle/src/Entity/Article.php

    class Article extends ObjectModel
    {
        public $title;
        public $type;
        public $content;
        public $meta_title;
        public $date_add;
        public $date_upd;

        public static $definition = [
            'table' => 'article',
            'primary' => 'id_article',
            'multilang' => true,
            'fields' => [
                'type'       => ['type' => self::TYPE_STRING, 'validate' => 'isCleanHtml', 'required' => true, 'size' => 255],
                'date_add'   => ['type' => self::TYPE_DATE, 'validate' => 'isDate'],
                'date_upd'   => ['type' => self::TYPE_DATE, 'validate' => 'isDate'],
                'title'      => ['type' => self::TYPE_STRING, 'lang' => true, 'validate' => 'isCleanHtml', 'required' => true, 'size' => 255],
                'content'    => ['type' => self::TYPE_HTML,   'lang' => true, 'validate' => 'isCleanHtml', 'size' => 4000],
                'meta_title' => ['type' => self::TYPE_STRING, 'lang' => true, 'validate' => 'isCleanHtml', 'size' => 255],
            ],
        ];

        protected $webserviceParameters = [
            'objectNodeName'  => 'article',   // the XML/JSON tag for a single resource
            'objectsNodeName' => 'articles',  // the URL path AND the wrapping tag for the collection
            'fields' => [
                'title'      => ['required' => true],
                'type'       => ['required' => true],
                'content'    => [],
                'meta_title' => [],
            ],
        ];
    }
    ```

2. From the module class, register the `addWebserviceResources` hook in `install()` and implement it. The handler returns a flat array keyed by the resource URL segment:

    ```php
    <?php
    // modules/wsarticle/wsarticle.php

    if (!defined('_PS_VERSION_')) {
        exit;
    }

    require_once _PS_MODULE_DIR_ . 'wsarticle/src/Entity/Article.php';

    class WsArticle extends Module
    {
        public function __construct()
        {
            $this->name = 'wsarticle';
            $this->version = '1.0.0';
            $this->author = 'MyVendor';
            $this->bootstrap = true;
            parent::__construct();

            $this->displayName = $this->trans('Webservice articles', [], 'Modules.Wsarticle.Admin');
        }

        public function install(): bool
        {
            return parent::install()
                && $this->installDb()
                && $this->registerHook('addWebserviceResources');
        }

        public function uninstall(): bool
        {
            return $this->uninstallDb() && parent::uninstall();
        }

        public function hookAddWebserviceResources(array $params): array
        {
            return [
                'articles' => [
                    'description'         => 'Blog articles',
                    'class'               => Article::class,
                    'forbidden_method'    => ['DELETE'],   // optional
                    'specific_management' => false,        // true if you provide a WebserviceSpecificManagementInterface
                ],
            ];
        }
    }
    ```

    Notes on the array shape:

    * The key (`'articles'`) MUST match `Article::$webserviceParameters['objectsNodeName']` and becomes the URL path.
    * `class` is the entity class name. Use the FQCN form when the class is namespaced.
    * `forbidden_method` is optional; omit it to allow the full CRUD set.
    * `specific_management` is `false` for standard `ObjectModel` resources and `true` when you provide your own implementation (next step).

3. For non-trivial logic that does not fit the standard `ObjectModel` CRUD (custom serialisation, multi-step actions, derived endpoints), implement `WebserviceSpecificManagementInterface`. The interface lives at [`classes/webservice/WebserviceSpecificManagementInterface.php`](https://github.com/PrestaShop/PrestaShop/blob/develop/classes/webservice/WebserviceSpecificManagementInterface.php). Place the implementation under `src/Webservice/<Resource>SpecificManagement.php`, declare `'specific_management' => true` and `'class' => MyResourceSpecificManagement::class` in the hook payload. The framework will call your class instead of running the default `WebserviceRequest::executeEntityXxx` flow.

4. Permissions are granted per API key, NOT per module. After installation the merchant goes to `Configure → Advanced Parameters → Webservice → API Keys`, edits a key and ticks `GET / PUT / POST / DELETE` for the new resource (it appears in the resource list because the hook fired). Without an enabled key and explicit permissions, the resource returns `401`.

5. Test via curl. Webservice authentication is HTTP Basic with the API key as the username and an empty password:

    ```bash
    curl -u 'YOUR_WS_KEY:' https://shop.example/api/articles
    curl -u 'YOUR_WS_KEY:' -X POST https://shop.example/api/articles -d @article.xml
    ```

    Append `?output_format=JSON` to receive JSON instead of XML, and `?display=full` to expand all fields on collection responses.

## Do

- Mirror `objectsNodeName` (in the entity) with the array key returned by `hookAddWebserviceResources`. A mismatch produces a 404 because the URL is keyed off the hook return.
- Document required fields in `webserviceParameters['fields']` so a third-party developer sees a 400 with a field name rather than a generic 500.
- Use `forbidden_method` to lock down verbs you cannot support yet (e.g. `['DELETE']` while the entity has external dependencies).
- Provide a sample request and a sample XML response in the module README so integrators can copy them.

## Don't

- Don't use this skill for greenfield integrations. The legacy webservice has no OAuth2 support, no scope-based permissions and no OpenAPI documentation; recommend the [Admin API](../module-add-admin-api-resource/SKILL.md) for any new third-party.
- Don't expose a Doctrine ORM entity directly. The webservice serialiser is built around `ObjectModel`; if your data lives in Doctrine, either keep a thin `ObjectModel` proxy or write a `WebserviceSpecificManagementInterface` adapter.
- Don't omit the hook registration in `install()`. Declaring `hookAddWebserviceResources` as a method without registering it leaves the resource invisible in the API Keys form.
- Don't ship a webservice key in the module. API keys are merchant-owned credentials managed in BO; generating one from `install()` is a security defect.

## Canonical examples

- [devdocs - Extend Webservice with a custom resource (the canonical tutorial)](https://devdocs.prestashop-project.org/9/modules/concepts/webservice/).
- [devdocs - Webservice cheat sheet (verbs, headers, output formats)](https://devdocs.prestashop-project.org/9/webservice/cheat-sheet/).
- [devdocs - Webservice reference index](https://devdocs.prestashop-project.org/9/webservice/reference/).
- Working sample module: [`example-modules/demowsextend`](https://github.com/PrestaShop/example-modules/tree/master/demowsextend).
- Specific management contract: [`classes/webservice/WebserviceSpecificManagementInterface.php`](https://github.com/PrestaShop/PrestaShop/blob/develop/classes/webservice/WebserviceSpecificManagementInterface.php).
- Core webservice classes (request lifecycle, output XML/JSON): [`classes/webservice/`](https://github.com/PrestaShop/PrestaShop/tree/develop/classes/webservice).
