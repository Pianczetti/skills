---
name: migrate-helperform-to-symfony-form
description: Replace a legacy getContent() + HelperForm configuration page with a Symfony FormType, FormHandler, FormDataProvider and Twig template. Use when a module still renders its settings through the HelperForm array and needs to move to the modern Form stack.
---

## Requirements

- Module name and vendor namespace.
- The existing `$fields_form` array structure (field names, types, validation rules).
- Configuration keys read/written by the form (`Configuration::get()` / `Configuration::updateValue()`).
- Whether multistore-scoped values are needed (`ShopConstraint`).

## Steps

1. Create the FormType under `src/Form/Type/<Name>ConfigurationType.php` using Symfony form types. Map each HelperForm field to its Symfony equivalent (`TextType`, `SwitchType`, `ChoiceType`, `TranslateType`, etc.):

    ```php
    namespace MyVendor\Mymodule\Form\Type;

    use PrestaShopBundle\Form\Admin\Type\TranslatorAwareType;
    use Symfony\Component\Form\Extension\Core\Type\TextType;
    use Symfony\Component\Form\FormBuilderInterface;

    class ConfigurationType extends TranslatorAwareType
    {
        public function buildForm(FormBuilderInterface $builder, array $options): void
        {
            $builder
                ->add('api_key', TextType::class, [
                    'label' => $this->trans('API Key', [], 'Modules.Mymodule.Admin'),
                    'required' => true,
                ])
            ;
        }
    }
    ```

2. Create the FormDataProvider under `src/Form/DataProvider/<Name>ConfigurationDataProvider.php` implementing `FormDataProviderInterface`. Read current values from `Configuration::get()` and write them in `setData()`:

    ```php
    namespace MyVendor\Mymodule\Form\DataProvider;

    use PrestaShop\PrestaShop\Core\Form\FormDataProviderInterface;
    use PrestaShop\PrestaShop\Core\Configuration\DataConfigurationInterface;

    class ConfigurationDataProvider implements FormDataProviderInterface
    {
        public function getData(): array
        {
            return ['api_key' => \Configuration::get('MYMODULE_API_KEY')];
        }

        public function setData(array $data): array
        {
            \Configuration::updateValue('MYMODULE_API_KEY', $data['api_key']);
            return [];
        }
    }
    ```

3. Register services in `config/services.yml`:

    ```yaml
    services:
      MyVendor\Mymodule\Form\Type\ConfigurationType:
        parent: 'form.type.translatable.aware'
        public: true
        tags: ['form.type']

      MyVendor\Mymodule\Form\DataProvider\ConfigurationDataProvider:
        autowire: true

      mymodule.form.handler.configuration:
        class: 'PrestaShop\PrestaShop\Core\Form\Handler'
        arguments:
          - '@form.factory'
          - '@prestashop.core.hook.dispatcher'
          - '@MyVendor\Mymodule\Form\DataProvider\ConfigurationDataProvider'
          - 'MyVendor\Mymodule\Form\Type\ConfigurationType'
          - 'Configuration'
    ```

4. Update the controller action to use the FormHandler:

    ```php
    public function configAction(Request $request): Response
    {
        $formHandler = $this->get('mymodule.form.handler.configuration');
        $form = $formHandler->getForm();
        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            $formHandler->save($form->getData());
            $this->addFlash('success', $this->trans('Settings saved.', [], 'Modules.Mymodule.Admin'));
            return $this->redirectToRoute('mymodule_config');
        }

        return $this->render('@Modules/mymodule/views/templates/admin/config.html.twig', [
            'form' => $form->createView(),
        ]);
    }
    ```

5. Create the Twig template rendering the form with the core form theme:

    ```twig
    {% extends '@PrestaShop/Admin/layout.html.twig' %}
    {% block content %}
      {{ form_start(form) }}
      {{ form_widget(form) }}
      <button type="submit" class="btn btn-primary">{{ 'Save'|trans({}, 'Admin.Actions') }}</button>
      {{ form_end(form) }}
    {% endblock %}
    ```

6. Remove the old `getContent()` method body and the `$fields_form` array from the module class. If the module class still needs a `getContent()` for the BO "Configure" link, redirect to the new route:

    ```php
    public function getContent(): void
    {
        Tools::redirectAdmin($this->context->link->getAdminLink('AdminMymoduleConfig'));
    }
    ```

7. Verify by loading the config page in the BO: submit with valid and invalid data, confirm Configuration values persist.

## Do

- Use `TranslatorAwareType` as base so `$this->trans()` is available inside `buildForm()`.
- Map HelperForm `'type' => 'switch'` to `PrestaShopBundle\Form\Admin\Type\SwitchType`.
- Return an empty array from `setData()` on success; return an array of error strings on failure.

## Don't

- Don't keep `HelperForm` alongside the new Symfony Form; remove it in the same PR.
- Don't call `Configuration::updateValue()` directly from the controller; always go through the DataProvider.
- Don't use `$this->l()` in the FormType; use `$this->trans()` with `Modules.Mymodule.Admin` domain.

## Related skills

- `module-add-config-page-modern` - greenfield Symfony Form config page.
- `migrate-admin-controller` - the controller migration this form plugs into.
- `migrate-legacy-translations` - replacing `$this->l()` in labels.
