---
name: module-add-carrier-module
description: Scaffold a PrestaShop 9 carrier module that registers a Carrier ObjectModel on install, computes shipping cost per cart, and preserves order history on uninstall by soft-deleting the carrier. Use when the user needs to expose a new shipping option (DHL, UPS, click-and-collect, etc.).
---

## Requirements

Ask the user:
* Carrier display name and a short technical slug (used as `$this->name`).
* Range mode: by weight (`need_range = 1`, `range_behavior = 0`) or by price.
* Range rows: each row is `(min, max, cost)` per zone (e.g. weight 0-1 kg in zone "Europe" costs 5.90).
* Zones served (existing PS zones, e.g. `Europe`, `North America`) and the tax rule group id.
* `max_package_weight`, max width/height/depth.
* Whether the cost is computed server-side (static ranges in DB) or fetched live from a carrier API in `getOrderShippingCostExternal`.

## Steps

1. Create a standard module via `module-create`. The main class extends `CarrierModule`, not `Module`, so PrestaShop auto-detects the cost callbacks:

    ```php
    class Mycarrier extends CarrierModule
    {
        public $id_carrier; // populated by core when the carrier is selected on the cart

        public function __construct()
        {
            $this->name = 'mycarrier';
            $this->tab = 'shipping_logistics';
            $this->version = '1.0.0';
            $this->bootstrap = true;
            $this->ps_versions_compliancy = ['min' => '9.0.0', 'max' => _PS_VERSION_];
            parent::__construct();
            $this->displayName = $this->trans('My Carrier', [], 'Modules.Mycarrier.Admin');
        }
    }
    ```

2. In `install()`, create a `Carrier` ObjectModel and persist it. The two flags that bind the carrier to your module are `is_module = true` and `external_module_name = $this->name`. Save the resulting id under a `Configuration` key so `uninstall()` can find it:

    ```php
    public function install(): bool
    {
        if (!parent::install() || !$this->registerHook(['displayCarrierExtraContent'])) {
            return false;
        }

        $carrier = new Carrier();
        $carrier->name = 'My Carrier';
        $carrier->is_module = true;
        $carrier->shipping_external = true;
        $carrier->external_module_name = $this->name;
        $carrier->need_range = true;
        $carrier->range_behavior = 0; // 0 = apply highest defined range, 1 = disable when out of range
        $carrier->shipping_method = Carrier::SHIPPING_METHOD_WEIGHT; // or SHIPPING_METHOD_PRICE
        $carrier->max_width = 100;
        $carrier->max_height = 100;
        $carrier->max_depth = 100;
        $carrier->max_weight = 30;

        foreach (Language::getLanguages() as $lang) {
            $carrier->delay[$lang['id_lang']] = $this->trans('2-3 working days', [], 'Modules.Mycarrier.Shop');
        }

        if (!$carrier->add()) {
            return false;
        }

        // Logo (75x75) under views/img/carrier.jpg
        @copy(_PS_MODULE_DIR_ . $this->name . '/views/img/carrier.jpg', _PS_SHIP_IMG_DIR_ . $carrier->id . '.jpg');

        // Zones
        foreach (Zone::getZones(true) as $zone) {
            Db::getInstance()->insert('carrier_zone', ['id_carrier' => (int) $carrier->id, 'id_zone' => (int) $zone['id_zone']]);
        }

        // Weight ranges
        $range = new RangeWeight();
        $range->id_carrier = (int) $carrier->id;
        $range->delimiter1 = 0;
        $range->delimiter2 = 30;
        $range->add();

        // Delivery price = (carrier x zone x range)
        foreach (Zone::getZones(true) as $zone) {
            Db::getInstance()->insert('delivery', [
                'id_carrier'        => (int) $carrier->id,
                'id_range_price'    => null,
                'id_range_weight'   => (int) $range->id,
                'id_zone'           => (int) $zone['id_zone'],
                'price'             => 5.90,
            ]);
        }

        // Shop association
        $carrier->setGroups([(int) Configuration::get('PS_UNIDENTIFIED_GROUP'), (int) Configuration::get('PS_GUEST_GROUP'), (int) Configuration::get('PS_CUSTOMER_GROUP')]);

        return Configuration::updateValue('MYCARRIER_CARRIER_ID', (int) $carrier->id);
    }
    ```

3. In the module class, implement BOTH cost callbacks. PS calls `getOrderShippingCost($cart, $shipping_cost)` for the in-DB cost and `getOrderShippingCostExternal($cart)` when `shipping_external = true` (return the API-computed value, or the same DB value, or `false` to disable the carrier for that cart):

    ```php
    public function getOrderShippingCost($cart, $shipping_cost)
    {
        if (!$this->active) { return false; }
        if (!$this->isCartEligible($cart)) { return false; } // returning false hides the carrier

        return (float) $shipping_cost; // already includes the configured ranges
    }

    public function getOrderShippingCostExternal($cart)
    {
        return $this->getOrderShippingCost($cart, 0);
    }
    ```

4. In `uninstall()`, mark the carrier `deleted = 1`. NEVER hard-delete the row: existing orders reference `id_carrier` and the BO order list joins on `ps_carrier`. Hard-deleting breaks the order history and any invoice still using that carrier:

    ```php
    public function uninstall(): bool
    {
        $carrierId = (int) Configuration::get('MYCARRIER_CARRIER_ID');
        if ($carrierId) {
            $carrier = new Carrier($carrierId);
            $carrier->deleted = 1;
            $carrier->update();
        }
        Configuration::deleteByName('MYCARRIER_CARRIER_ID');
        return parent::uninstall();
    }
    ```

5. To attach extra content to the carrier on the front (a date picker, a pickup-point selector, a phone-number prompt), register the `displayCarrierExtraContent` hook. The returned HTML is injected under the carrier line on the checkout page; the value the customer picks is sent back to your module via `actionValidateStepComplete` or saved against the cart:

    ```php
    public function hookDisplayCarrierExtraContent(array $params): string
    {
        if ((int) $params['carrier']['id'] !== (int) Configuration::get('MYCARRIER_CARRIER_ID')) {
            return '';
        }
        return $this->fetch('module:' . $this->name . '/views/templates/hook/extra.tpl');
    }
    ```

6. Test: install, set up zones in BO, place a test order. Then uninstall, reinstall, and confirm a NEW `id_carrier` is created (the soft-deleted one is preserved alongside it). Update the migration script (see `module-add-migration`) when the version-bumped module needs to add new ranges or zones.

## Do

- Set `is_module = true` AND `external_module_name = $this->name` AND `shipping_external = true` so PrestaShop calls your `getOrderShippingCostExternal()`.
- Always populate `carrier_zone`, `range_weight` (or `range_price`) and `delivery` together. A carrier without a delivery row in the cart's zone is silently filtered out at checkout.
- Soft-delete on uninstall (`deleted = 1`). Reinstalling creates a new carrier id; that is correct behaviour.
- Return `false` from the cost callback to hide the carrier for that specific cart (over weight, ineligible country, etc.). Returning `0.0` would offer free shipping.

## Don't

- Don't `DELETE FROM ps_carrier WHERE id_carrier = ...` on uninstall. It nukes order history, invoices and shipping legends. Set `deleted = 1` instead.
- Don't update an existing carrier in place across module versions. Create a NEW carrier (and soft-delete the old one) so historical orders keep their original cost computation.
- Don't compute the cost in JavaScript. The amount returned by `getOrderShippingCost()` is the amount the merchant invoices.
- Don't extend `Module` for new carrier modules. Extend `CarrierModule` so the base class wires the cost callbacks correctly.

## Canonical examples

- [devdocs - Carrier modules](https://devdocs.prestashop-project.org/9/modules/carrier/) (full lifecycle including range tables, zones and the `is_module`/`shipping_external` flags).
- [`PrestaShop/ps_carriercomparison`](https://github.com/PrestaShop/ps_carriercomparison) - real carrier comparison module shipped by the core team.
- [`PrestaShop/PrestaShop` - Carrier class](https://github.com/PrestaShop/PrestaShop/blob/develop/classes/Carrier.php) - the ObjectModel definition with `is_module`, `shipping_external`, `external_module_name`, `need_range`, `range_behavior`.
- [`PrestaShop/example-modules`](https://github.com/PrestaShop/example-modules) - canonical modules organisation; clone the structure of `democonsolecommand`/`demodoctrine` and replace the body with the carrier flow above.
