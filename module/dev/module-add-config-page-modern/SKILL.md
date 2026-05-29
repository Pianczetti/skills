---
name: module-add-config-page-modern
description: Build a modern Symfony Form + FormHandler + FormDataProvider configuration page for a PrestaShop 9 module. Use when the user wants a Back Office settings screen rendered from a modern admin controller.
---

## Requirements

Ask the user:
* The list of fields and their types (text, number, switch, choice, ...).
* The configuration keys, in `UPPER_SNAKE_CASE` and prefixed with the module name (e.g. `MYMODULE_API_KEY`, `MYMODULE_DEFAULT_LIMIT`). Predictable prefixes prevent collisions.
* Whether the form must be multistore-aware (per-shop / per-shop-group / global). If yes, the FormDataProvider must read/write through `Configuration::get/updateValue` with the correct `id_shop` / `id_shop_group` arguments.
* The route name and path for the configuration page (see `module-add-symfony-route`).

## Steps

1. Create the FormType under `src/Form/<Name>Configuration.php` extending `TranslatorAwareType`. The translator domain is `Modules.<Modulename>.Admin`:

    ```php
    <?php
    namespace MyVendor\Mymodule\Form;

    use PrestaShopBundle\Form\Admin\Type\TranslatorAwareType;
    use Symfony\Component\Form\Extension\Core\Type\IntegerType;
    use Symfony\Component\Form\Extension\Core\Type\TextType;
    use Symfony\Component\Form\FormBuilderInterface;

    class GeneralConfiguration extends TranslatorAwareType
    {
        public function buildForm(FormBuilderInterface $builder, array $options): void
        {
            $builder
                ->add('api_key', TextType::class, [
                    'label' => $this->trans('API key', 'Modules.Mymodule.Admin'),
                    'required' => true,
                ])
                ->add('default_limit', IntegerType::class, [
                    'label' => $this->trans('Default limit', 'Modules.Mymodule.Admin'),
                ]);
        }
    }
    ```

2. Create a `FormDataProvider` implementing `PrestaShop\PrestaShop\Core\Form\FormDataProviderInterface`. `getData()` reads from `Configuration` for the current shop context; `setData()` writes and returns an array of validation errors:

    ```php
    <?php
    namespace MyVendor\Mymodule\Form;

    use Configuration;
    use PrestaShop\PrestaShop\Core\Form\FormDataProviderInterface;
    use PrestaShop\PrestaShop\Adapter\Shop\Context as ShopContext;

    class GeneralConfigurationDataProvider implements FormDataProviderInterface
    {
        public function __construct(private readonly ShopContext $shopContext) {}

        public function getData(): array
        {
            $shopConstraint = $this->shopContext->getShopConstraint();
            return [
                'api_key' => (string) Configuration::get('MYMODULE_API_KEY', null, $shopConstraint->getShopGroupId()?->getValue(), $shopConstraint->getShopId()?->getValue()),
                'default_limit' => (int) Configuration::get('MYMODULE_DEFAULT_LIMIT', null, $shopConstraint->getShopGroupId()?->getValue(), $shopConstraint->getShopId()?->getValue()),
            ];
        }

        public function setData(array $data): array
        {
            $errors = [];
            if ($data['default_limit'] < 1) { $errors[] = ['key' => 'Limit must be positive', 'domain' => 'Modules.Mymodule.Admin', 'parameters' => []]; }
            if ($errors) { return $errors; }

            Configuration::updateValue('MYMODULE_API_KEY', (string) $data['api_key']);
            Configuration::updateValue('MYMODULE_DEFAULT_LIMIT', (int) $data['default_limit']);
            return [];
        }
    }
    ```

3. Wire a `FormHandler` in `config/services.yml` using the core's `prestashop.core.hook.form_handler_factory`:

    ```yaml
    services:
      MyVendor\Mymodule\Form\GeneralConfiguration: ~
      MyVendor\Mymodule\Form\GeneralConfigurationDataProvider: ~

      mymodule.form.general_configuration_handler:
        class: 'PrestaShop\PrestaShop\Core\Form\Handler'
        public: true
        arguments:
          $formFactory: '@form.factory'
          $hookDispatcher: '@prestashop.core.hook.dispatcher'
          $formDataProvider: '@MyVendor\Mymodule\Form\GeneralConfigurationDataProvider'
          $formBuilderType: 'MyVendor\Mymodule\Form\GeneralConfiguration'
          $hookName: 'mymoduleGeneralConfiguration'
    ```

4. Render and process the form from a modern admin controller (`PrestaShopAdminController`):

    ```php
    public function indexAction(Request $request): Response
    {
        $handler = $this->container->get('mymodule.form.general_configuration_handler');
        $form = $handler->getForm();
        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            $errors = $handler->save($form->getData());
            if ($errors) { foreach ($errors as $e) { $this->addFlash('error', $this->trans($e['key'], $e['domain'], $e['parameters'])); } }
            else { $this->addFlash('success', $this->trans('Settings saved', 'Modules.Mymodule.Admin')); }
            return $this->redirectToRoute('mymodule_settings_index');
        }

        return $this->render('@Modules/mymodule/views/templates/admin/settings/index.html.twig', [
            'form' => $form->createView(),
        ]);
    }
    ```

5. Twig template `views/templates/admin/settings/index.html.twig` extends the BO layout and renders the form:

    ```twig
    {% extends '@PrestaShop/Admin/layout.html.twig' %}
    {% block content %}
      {{ form_start(form) }}
        {{ form_widget(form) }}
        <button class="btn btn-primary" type="submit">{{ 'Save'|trans({}, 'Admin.Actions') }}</button>
      {{ form_end(form) }}
    {% endblock %}
    ```

6. For a multistore-aware form, also include the multistore selector (`{{ render(controller(...)) }}`) and read/write configuration with the right `id_shop_group` / `id_shop` arguments.

## Do

- Extend `TranslatorAwareType` so `$this->trans('...', 'Modules.Mymodule.Admin')` works in the form builder.
- Validate inside `setData()` and return errors as an array. Never throw from a data provider.
- Keep configuration keys prefixed and uppercase: `MYMODULE_*`. Document them in `composer.json` `extra` if other modules consume them.
- Register everything in `config/services.yml`. The `*_handler` service must be `public: true` because the controller fetches it by ID.
- Always pass the current `ShopConstraint` to `Configuration::get/updateValue` for multistore correctness.

## Don't

- Don't use `HelperForm` or override `getContent()` for new modules. That is the legacy path; see `module-add-config-page-legacy` only when supporting PS < 8.
- Don't write configuration directly from the controller. Go through the FormHandler and FormDataProvider so validation and hooks fire.
- Don't hardcode translation strings without the `Modules.Mymodule.Admin` domain.
- Don't skip the multistore context. `Configuration::updateValue('K', 'v')` writes to ALL shops; pass `$shop_group_id`/`$shop_id` explicitly.

## PS9 Module Compatibility Notes

### Form Handler argument name

PS9 `PrestaShop\Core\Form\Handler` expects `$formType`, NOT `$formBuilderType`:

```yaml
# WRONG
$formBuilderType: 'MyVendor\Mymodule\Form\ConfigurationType'
# CORRECT
$formType: 'MyVendor\Mymodule\Form\ConfigurationType'
```

### TranslatorAwareType requires $locales

If your FormType extends `TranslatorAwareType`, it needs `$locales` array (cannot be autowired). Either:
1. Use `parent: 'form.type.translatable.aware'` + tag `form.type`
2. Or extend `AbstractType` + inject `TranslatorInterface` manually

### Dynamic choices in FormTypes

Never use `'choices' => []`. Create a `ChoiceProvider` service with ALL arguments explicit:

```yaml
MyVendor\Mymodule\Form\ResourceChoiceProvider:
  arguments:
    $connection: '@doctrine.dbal.default_connection'
    $dbPrefix: '%database_prefix%'
    $languageId: "@=service('prestashop.adapter.legacy.context').getContext().language.id"
```

## Canonical examples

- [devdocs - Modern configuration page](https://devdocs.prestashop-project.org/9/modules/creation/adding-configuration-page-modern/).
- [devdocs - Forms (concept)](https://devdocs.prestashop-project.org/9/modules/concepts/forms/).
- [devdocs - Migration guide: settings forms](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/forms/settings-forms/).
- [devdocs - Multistore configuration forms](https://devdocs.prestashop-project.org/9/development/multistore/configuration-forms/).
- Working sample: [`example-modules/demoformdataproviders`](https://github.com/PrestaShop/example-modules/tree/master/demoformdataproviders) (FormDataProvider + Handler).
- Multistore-aware sample: [`example-modules/demomultistoreform`](https://github.com/PrestaShop/example-modules/tree/master/demomultistoreform).
