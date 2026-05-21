---
name: module-add-payment-module
description: Scaffold a PrestaShop 9 payment module that exposes payment options at checkout and validates orders through the canonical paymentOptions / validateOrder / displayPaymentReturn flow. Use when the user wants to add a new gateway (direct, redirect, iframe, off-site or webhook-confirmed).
---

## Requirements

Ask the user:
* Payment provider name and its short technical slug (lower_snake_case, used as `$this->name`, e.g. `mygateway`).
* Supported flow(s): direct (synchronous capture), redirect (off-site form), iframe, or webhook-confirmed (asynchronous).
* Supported currencies (ISO codes) and supported countries (ISO 3166-1 alpha-2). The module must filter `paymentOptions` by these.
* Whether the gateway provides an instant payment notification / webhook, and the signature scheme (HMAC SHA-256, RSA, JWT...).
* Configuration keys: API key, secret, mode (`live`/`sandbox`).

## Steps

1. Create a standard PS 9 module via `module-create`. The main class extends `PaymentModule`, not `Module`, so the helpers `validateOrder()`, `currentOrder` and `currentOrderReference` are available:

    ```php
    class Mygateway extends PaymentModule
    {
        public function __construct()
        {
            $this->name = 'mygateway';
            $this->tab = 'payments_gateways';
            $this->version = '1.0.0';
            $this->author = 'My Company';
            $this->bootstrap = true;
            $this->controllers = ['validation', 'webhook'];
            $this->ps_versions_compliancy = ['min' => '9.0.0', 'max' => _PS_VERSION_];
            parent::__construct();
            $this->displayName = $this->trans('My Gateway', [], 'Modules.Mygateway.Admin');
            $this->limited_currencies = ['EUR', 'USD'];
        }
    }
    ```

2. In `install()`, register the two hooks every payment module needs and persist any `Configuration` keys:

    ```php
    public function install(): bool
    {
        return parent::install()
            && $this->registerHook(['paymentOptions', 'displayPaymentReturn'])
            && Configuration::updateValue('MYGATEWAY_API_KEY', '')
            && Configuration::updateValue('MYGATEWAY_MODE', 'sandbox');
    }
    ```

3. Implement `hookPaymentOptions(array $params): array`. Return an array of `PrestaShop\PrestaShop\Core\Payment\PaymentOption` objects. Filter by currency and country before building the option list. Each option sets a CTA, a logo, and either a form action URL (redirect/iframe) or an `additionalInformation` template (direct):

    ```php
    use PrestaShop\PrestaShop\Core\Payment\PaymentOption;

    public function hookPaymentOptions(array $params): array
    {
        if (!$this->active || !$this->checkCurrency($params['cart'])) {
            return [];
        }

        $option = (new PaymentOption())
            ->setModuleName($this->name)
            ->setCallToActionText($this->trans('Pay with My Gateway', [], 'Modules.Mygateway.Shop'))
            ->setAction($this->context->link->getModuleLink($this->name, 'validation', [], true))
            ->setLogo(Media::getMediaPath(_PS_MODULE_DIR_ . $this->name . '/views/img/logo.png'))
            ->setAdditionalInformation($this->fetch('module:' . $this->name . '/views/templates/front/payment_infos.tpl'));

        return [$option];
    }

    protected function checkCurrency(Cart $cart): bool
    {
        $currency = new Currency((int) $cart->id_currency);
        foreach ($this->getCurrency((int) $cart->id_currency) as $module_currency) {
            if ((int) $currency->id === (int) $module_currency['id_currency']) {
                return true;
            }
        }
        return false;
    }
    ```

4. Create the validation front controller at `controllers/front/validation.php`. This is what `setAction()` points to. Reload the cart server-side, recompute the total, then call `$this->module->validateOrder(...)` with `Configuration::get('PS_OS_PAYMENT')` as the order state for synchronous captures (use `PS_OS_BANKWIRE` / your own state for pending flows):

    ```php
    class MygatewayValidationModuleFrontController extends ModuleFrontController
    {
        public function postProcess(): void
        {
            $cart = $this->context->cart;
            if (!$cart->id_customer || !$cart->id_address_delivery || !$cart->id_address_invoice || !$this->module->active) {
                Tools::redirect('index.php?controller=order&step=1');
            }

            $customer = new Customer((int) $cart->id_customer);
            if (!Validate::isLoadedObject($customer)) {
                Tools::redirect('index.php?controller=order&step=1');
            }

            // Recompute the total server-side; never trust a hidden field from the cart page.
            $total = (float) $cart->getOrderTotal(true, Cart::BOTH);
            $currencyId = (int) Context::getContext()->currency->id;

            $this->module->validateOrder(
                (int) $cart->id,
                (int) Configuration::get('PS_OS_PAYMENT'),
                $total,
                $this->module->displayName,
                null,
                ['transaction_id' => $transactionId ?? ''],
                $currencyId,
                false,
                $customer->secure_key
            );

            Tools::redirect('index.php?controller=order-confirmation&id_cart=' . (int) $cart->id
                . '&id_module=' . (int) $this->module->id
                . '&id_order=' . (int) $this->module->currentOrder
                . '&key=' . $customer->secure_key);
        }
    }
    ```

5. For asynchronous gateways, expose a webhook controller (`controllers/front/webhook.php` declared in `$this->controllers`). Verify the signature FIRST, then call `validateOrder()`. Reject the request with `header('HTTP/1.1 400 Bad Request')` on signature failure:

    ```php
    public function postProcess(): void
    {
        $payload = (string) file_get_contents('php://input');
        $signature = (string) Tools::getValue('signature', $_SERVER['HTTP_X_MG_SIGNATURE'] ?? '');

        if (!hash_equals(hash_hmac('sha256', $payload, (string) Configuration::get('MYGATEWAY_SECRET')), $signature)) {
            header('HTTP/1.1 400 Bad Request');
            exit('invalid signature');
        }

        $event = json_decode($payload, true);
        $cart = new Cart((int) $event['cart_id']);
        if (!Validate::isLoadedObject($cart) || (float) $event['amount'] !== (float) $cart->getOrderTotal(true, Cart::BOTH)) {
            header('HTTP/1.1 422 Unprocessable Entity');
            exit('amount mismatch');
        }

        $customer = new Customer((int) $cart->id_customer);
        $this->module->validateOrder(
            (int) $cart->id,
            (int) Configuration::get('PS_OS_PAYMENT'),
            (float) $cart->getOrderTotal(true, Cart::BOTH),
            $this->module->displayName,
            null,
            ['transaction_id' => (string) $event['id']],
            (int) $cart->id_currency,
            false,
            $customer->secure_key
        );
    }
    ```

6. Implement `hookDisplayPaymentReturn(array $params): string` and ship `views/templates/hook/payment_return.tpl`. The hook is called on the order confirmation page. Render a short summary, never re-trigger payment logic:

    ```php
    public function hookDisplayPaymentReturn(array $params): string
    {
        if (!$this->active) { return ''; }
        $this->context->smarty->assign([
            'shop_name' => $this->context->shop->name,
            'reference' => $params['order']->reference,
        ]);
        return $this->fetch('module:' . $this->name . '/views/templates/hook/payment_return.tpl');
    }
    ```

## Do

- Always recompute the cart total server-side before calling `validateOrder()`. The amount you pass is the amount the merchant gets paid.
- Always verify webhook signatures with `hash_equals()` (constant-time) before any state change.
- Use `Configuration::get('PS_OS_PAYMENT')` for captured payments and a custom pending state (`Configuration::get('MYGATEWAY_OS_PENDING')`) for asynchronous flows that have not yet confirmed.
- Filter `paymentOptions` by currency AND country AND cart total range; an unsupported option must not be returned at all.

## Don't

- Don't trust the cart total from the client (hidden form field, JS variable, query string). Recompute from `$cart->getOrderTotal()` before validating.
- Don't skip signature verification on webhooks. An unsigned webhook is a fraud vector, not a feature.
- Don't call `Cart::delete()` or manipulate the cart from the validation controller. `validateOrder()` already creates the order and clears the cart.
- Don't extend `Module` for new payment modules. Extend `PaymentModule` so the base class handles order persistence and stock.

## Canonical examples

- [devdocs - Payment modules](https://devdocs.prestashop-project.org/9/modules/payment/) (the contract for `paymentOptions`, `displayPaymentReturn` and `validateOrder`).
- [`PrestaShop/ps_checkpayment`](https://github.com/PrestaShop/ps_checkpayment) - reference redirect-style payment module shipped with the core.
- [`PrestaShop/ps_wirepayment`](https://github.com/PrestaShop/ps_wirepayment) - reference offline (pending) payment module; uses a custom order state instead of `PS_OS_PAYMENT`.
- [`PrestaShop/PrestaShop` - PaymentOption](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/Payment/PaymentOption.php) - full setter list (`setForm`, `setInputs`, `setBinary`, `setAdditionalInformation`).
- [`PrestaShop/PrestaShop` - PaymentModule](https://github.com/PrestaShop/PrestaShop/blob/develop/classes/PaymentModule.php) - the `validateOrder()` signature you must call.
