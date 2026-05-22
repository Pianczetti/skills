---
name: migrate-helperform-to-symfony-form
description: Replace a HelperForm-based configuration page (getContent + renderForm) with Symfony FormType + FormHandler + FormDataProvider + Twig template. Use when a module uses the legacy HelperForm pattern for its settings and must migrate to the PS 9 modern form architecture.
---

## Requirements

- Identify the legacy `getContent()` method and its `$fields_form` array defining form fields.
- Map each field entry (`type => 'text'`, `type => 'switch'`, `type => 'select'`) to a Symfony Form Type.
- List configuration keys read/written by `getConfigFieldsValues()` and `postProcess()`.
- Confirm the target route for the new settings page.

## Steps

1. **Extract field definitions** from the legacy `$fields_form` array. Typical legacy pattern:

    ```php
    // LEGACY - to be removed
    protected function renderForm(): string
    {
        $fields_form = [[
            'form' => [
                'legend' => ['title' => $this->l('Settings')],
                'input' => [
                    ['type' => 'text', 'label' => $this->l('API key'), 'name' => 'MYMODULE_API_KEY'],
                    ['type' => 'switch', 'label' => $this->l('Active'), 'name' => 'MYMODULE_ACTIVE', 'values' => [...]],
                ],
                'submit' => ['title' => $this->l('Save')],
            ],
        ]];
        $helper = new \HelperForm();
        // ...
        return $helper->generateForm($fields_form);
    }
    ```

2. **Create the Symfony FormType** under `src/Form/<Name>Configuration.php`:

    ```php
    namespace MyVendor\Mymodule\Form;

    use PrestaShopBundle\Form\Admin\Type\SwitchType;
    use PrestaShopBundle\Form\Admin\Type\TranslatorAwareType;
    use Symfony\Component\Form\Extension\Core\Type\TextType;
    use Symfony\Component\Form\FormBuilderInterface;

    class GeneralConfiguration extends TranslatorAwareType
    {
        public function buildForm(FormBuilderInterface $builder, array $options): void
        {
            $builder
                ->add('api_key', TextType::class, [
                    'label' => $this->trans('API key', 'Modules.Mymodule.Admin'),
                ])
                ->add('active', SwitchType::class, [
                    'label' => $this->trans('Active', 'Modules.Mymodule.Admin'),
                ]);
        }
    }
    ```

3. **Create the FormDataProvider** implementing `FormDataProviderInterface`:

    ```php
    <?php
    namespace MyVendor\Mymodule\Form;

    use PrestaShop\PrestaShop\Core\Form\FormDataProviderInterface;

    class GeneralConfigurationDataProvider implements FormDataProviderInterface
    {
        public function getData(): array
        {
            return [
                'api_key' => \Configuration::get('MYMODULE_API_KEY'),
                'active' => (bool) \Configuration::get('MYMODULE_ACTIVE'),
            ];
        }

        public function setData(array $data): array
        {
            \Configuration::updateValue('MYMODULE_API_KEY', $data['api_key']);
            \Configuration::updateValue('MYMODULE_ACTIVE', (int) $data['active']);
            return [];
        }
    }
    ```

4. **Wire the FormHandler** in `config/services.yml`:

    ```yaml
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

5. **Replace `getContent()`** with a modern controller action:

    ```php
    #[IsGranted('update', subject: '_legacy_controller')]
    public function settingsAction(Request $request): Response
    {
        $handler = $this->container->get('mymodule.form.general_configuration_handler');
        $form = $handler->getForm();
        $form->handleRequest($request);

        if ($form->isSubmitted() && $form->isValid()) {
            $errors = $handler->save($form->getData());
            if (empty($errors)) {
                $this->addFlash('success', $this->trans('Settings saved', 'Modules.Mymodule.Admin'));
            }
            return $this->redirectToRoute('mymodule_settings');
        }

        return $this->render('@Modules/mymodule/views/templates/admin/settings.html.twig', [
            'form' => $form->createView(),
        ]);
    }
    ```

6. **Delete the legacy code**: remove `getContent()`, `renderForm()`, `getConfigFieldsValues()`, and `postProcess()` from the main module class.

## Do

- Map legacy `type => 'switch'` to `PrestaShopBundle\Form\Admin\Type\SwitchType`, not a plain `CheckboxType`.
- Validate inside `setData()` and return errors as an array; never throw.
- Use `TranslatorAwareType` as the base so `$this->trans()` is available inside the form builder.

## Don't

- Don't keep `getContent()` alongside the new form; it causes double rendering if the module is configured via the legacy module manager page.
- Don't write configuration directly from the controller; always go through `FormHandler` + `FormDataProvider`.
- Don't use `$this->l('...')` in the new FormType; use `$this->trans('...', 'Modules.Mymodule.Admin')`.

## Canonical examples

- [devdocs - Migration guide: settings forms](https://devdocs.prestashop-project.org/9/development/architecture/migration-guide/forms/settings-forms/)
- [devdocs - Modern configuration page](https://devdocs.prestashop-project.org/9/modules/creation/adding-configuration-page-modern/)

## Related skills

- `module-add-config-page-modern` - full recipe for the modern form pattern.
- `migrate-admin-controller` - migrating the controller that hosts the form.
- `migrate-legacy-translations` - replacing `$this->l()` with Symfony Translator.
