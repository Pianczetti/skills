---
name: module-add-feature-flag
description: Register a feature flag from a PrestaShop 9 module and gate runtime behaviour behind it. Use when the user wants to ship a risky or partially-finished feature that the merchant can toggle from the BO without redeploying the module.
---

## Requirements

Ask the user:
* Flag identifier in `snake_case` (e.g. `mymod_new_pricing_engine`). Prefix it with the module short name to avoid collisions with core flags (`admin_api_multistore`, `discount`, `one_page_checkout`...).
* Default state: enabled or disabled. Risky features default to disabled.
* Stability: `beta` (most module flags) or `stable` (kept around for emergency rollback).
* Scope: single-shop or multistore. Feature flags in PS 9 are GLOBAL - the same on/off state applies across every shop. If you need per-shop branching, gate the flag AND combine with `ShopConstraint`.
* The label and description (translation domain `Modules.<modulename>.Admin`) shown in BO under Advanced Parameters > Feature Flags.

## Steps

1. Declare the flag at install-time. The `prestashop:feature-flag` console command only supports `enable | disable | list` - it does NOT have a `declare` subcommand. To register a new flag from a module, persist a `PrestaShopBundle\Entity\FeatureFlag` row in `install()` (or in an upgrade script) via the `prestashop.core.admin.feature_flag.repository` service:

    ```php
    use Doctrine\ORM\EntityManagerInterface;
    use PrestaShop\PrestaShop\Core\FeatureFlag\FeatureFlagSettings;
    use PrestaShopBundle\Entity\FeatureFlag;
    use PrestaShopBundle\Entity\Repository\FeatureFlagRepository;

    public function install(): bool
    {
        if (!parent::install()) { return false; }

        /** @var EntityManagerInterface $em */
        $em = $this->get('doctrine.orm.entity_manager');
        /** @var FeatureFlagRepository $repo */
        $repo = $this->get('prestashop.core.admin.feature_flag.repository');

        if (null === $repo->getByName('mymod_new_pricing_engine')) {
            $flag = new FeatureFlag('mymod_new_pricing_engine');
            $flag->setType(FeatureFlagSettings::TYPE_DEFAULT);   // 'env,dotenv,db'
            $flag->setStability(FeatureFlagSettings::STABILITY_BETA);
            $flag->setLabelWording('New pricing engine');
            $flag->setLabelDomain('Modules.Mymodule.Admin');
            $flag->setDescriptionWording('Route price computation through the v2 pricing engine. Disable to fall back to the legacy path.');
            $flag->setDescriptionDomain('Modules.Mymodule.Admin');
            $flag->setState(false);
            $em->persist($flag);
            $em->flush();
        }
        return true;
    }
    ```

   In `uninstall()`, delete the row so the BO list does not keep a dangling entry:

    ```php
    public function uninstall(): bool
    {
        $em = $this->get('doctrine.orm.entity_manager');
        $repo = $this->get('prestashop.core.admin.feature_flag.repository');
        if (null !== $flag = $repo->getByName('mymod_new_pricing_engine')) {
            $em->remove($flag);
            $em->flush();
        }
        return parent::uninstall();
    }
    ```

2. Gate runtime behaviour by injecting `PrestaShop\PrestaShop\Core\FeatureFlag\FeatureFlagManager` (the public service id is `prestashop.core.admin.feature_flag.feature_flag_manager`). Use `isEnabled()` / `isDisabled()` at every branch point:

    ```php
    use PrestaShop\PrestaShop\Core\FeatureFlag\FeatureFlagManager;

    final class PriceCalculator
    {
        public function __construct(
            private readonly FeatureFlagManager $featureFlagManager,
            private readonly LegacyPricer $legacy,
            private readonly NewPricer $v2,
        ) {}

        public function compute(Cart $cart): Money
        {
            if ($this->featureFlagManager->isEnabled('mymod_new_pricing_engine')) {
                return $this->v2->compute($cart);
            }
            return $this->legacy->compute($cart);
        }
    }
    ```

   `FeatureFlagManager` caches per-request, so calling `isEnabled()` in a hot loop is fine.

3. Toggle the flag at runtime via the BO or the CLI. The merchant goes to **Advanced Parameters > Feature Flags**, finds the row by its label (translated through `Modules.Mymodule.Admin`), and flips the switch. From the CLI:

    ```bash
    bin/console prestashop:feature-flag list
    bin/console prestashop:feature-flag enable mymod_new_pricing_engine
    bin/console prestashop:feature-flag disable mymod_new_pricing_engine
    ```

4. Override the flag from the environment. PS resolves the state by walking the configured layers in order; the default order is `env,dotenv,db` (constant `FeatureFlagSettings::TYPE_DEFAULT`). The env-var name is `PS_FF_<UPPER_FLAG_NAME>`:

    ```dotenv
    # .env.local - force-enable in staging without flipping the BO toggle
    PS_FF_MYMOD_NEW_PRICING_ENGINE=1
    ```

   An env layer wins over a dotenv layer wins over the database value - the BO list shows the active source in brackets next to the type column.

5. Plan the removal. A flag is technical debt by design. Document in the module CHANGELOG when the flag will be dropped (typically two minor versions after the feature ships stable). Removal is a single upgrade script that drops the row plus deletes the `if ($flag->isEnabled(...))` branches in code.

## Do

- Ship every risky or partially-finished feature behind a flag. The merchant is one click away from rolling back.
- Pick a name that includes the module short code (`mymod_*`). Core flags use bare snake_case (`discount`, `one_page_checkout`); the module prefix prevents collisions.
- Default to disabled for new features; default to enabled only when you're rolling back a previously-default behaviour.
- Translate label and description through `Modules.<modulename>.Admin` so they appear correctly in the BO list.
- Remove the flag (and its branches) once the feature has been default-enabled for two minor versions without incident.

## Don't

- Don't try `bin/console prestashop:feature-flag declare ...` - that subcommand does NOT exist. The console command supports only `enable`, `disable`, `list`.
- Don't read the flag from inside a tight loop without caching. `FeatureFlagManager` already caches per request, but if you bypass it (e.g. raw SQL on `ps_feature_flag`) you will incur a query per call.
- Don't ship a module that depends on a flag being enabled for normal operation. A flag is for risky NEW behaviour layered on top of a working baseline.
- Don't mutate flags from the front office. Toggling is a BO / CLI / env-var concern.

## Canonical examples

- [devdocs - prestashop:feature-flag](https://devdocs.prestashop-project.org/9/development/components/console/prestashop-feature-flag/) - the canonical CLI subcommands (`enable`, `disable`, `list`) and the layer priority (`dotenv`, `env`, `db`).
- [devdocs - Console commands index](https://devdocs.prestashop-project.org/9/development/components/console/) - sibling `prestashop:*` commands you may chain in scripts.
- [`PrestaShop/PrestaShop` - FeatureFlagManager](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/FeatureFlag/FeatureFlagManager.php) - the service to inject; defines `isEnabled`, `isDisabled`, `enable`, `disable`.
- [`PrestaShop/PrestaShop` - FeatureFlagSettings](https://github.com/PrestaShop/PrestaShop/blob/develop/src/Core/FeatureFlag/FeatureFlagSettings.php) - the `STABILITY_*`, `TYPE_*` and `PREFIX` constants used when registering a flag.
- [`PrestaShop/PrestaShop` - FeatureFlag entity](https://github.com/PrestaShop/PrestaShop/blob/develop/src/PrestaShopBundle/Entity/FeatureFlag.php) and [`FeatureFlagRepository`](https://github.com/PrestaShop/PrestaShop/blob/develop/src/PrestaShopBundle/Entity/Repository/FeatureFlagRepository.php) - persistence layer used by `install()`.
- [`PrestaShop/PrestaShop` - FeatureFlagCommand](https://github.com/PrestaShop/PrestaShop/blob/develop/src/PrestaShopBundle/Command/FeatureFlagCommand.php) - read it once to confirm the `enable | disable | list` surface.
