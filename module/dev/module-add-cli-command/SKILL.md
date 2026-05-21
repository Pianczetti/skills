---
name: module-add-cli-command
description: Ship a Symfony console command from a PrestaShop 9 module so the merchant can run a maintenance task, bulk import or scheduled job via bin/console <command>. Use when the user wants a CLI entry point that delegates to existing CQRS handlers and services.
---

## Requirements

Ask the user:
* Command name in `<vendor>:<module>:<verb>` shape (e.g. `mymod:vouchers:expire`, `mymod:catalog:reindex`). The verb is imperative.
* Arguments and options. Arguments are positional and required by default; options are flagged (`--dry-run`, `--shop=2`).
* Which existing CQRS handler / service the command should call. The command itself must stay thin.
* Whether the command runs in a multistore context. If yes, take a `--shop` / `--all-shops` / `--shop-group` option and translate it to a `ShopConstraint`.

## Steps

1. Create the command class under `src/Command/<Verb><Subject>Command.php` using the PSR-4 namespace declared in the module's `composer.json`. Use the `#[AsCommand]` attribute - PrestaShop's container has `autoconfigure: true` enabled by default, which auto-tags any class implementing `Symfony\Component\Console\Command\Command` with `console.command`. No manual `tags:` entry is required:

    ```php
    <?php
    declare(strict_types=1);

    namespace MyVendor\Mymodule\Command;

    use MyVendor\Mymodule\Domain\Voucher\Command\ExpireVouchersCommand;
    use PrestaShop\PrestaShop\Core\CommandBus\CommandBusInterface;
    use PrestaShop\PrestaShop\Core\Domain\Shop\ValueObject\ShopConstraint;
    use Symfony\Component\Console\Attribute\AsCommand;
    use Symfony\Component\Console\Command\Command;
    use Symfony\Component\Console\Input\InputInterface;
    use Symfony\Component\Console\Input\InputOption;
    use Symfony\Component\Console\Output\OutputInterface;
    use Symfony\Component\Console\Style\SymfonyStyle;
    use Symfony\Component\DependencyInjection\Attribute\Autowire;

    #[AsCommand(
        name: 'mymod:vouchers:expire',
        description: 'Mark expired vouchers as inactive.',
    )]
    final class ExpireVouchersCommand extends Command
    {
        public function __construct(
            #[Autowire(service: 'prestashop.core.command_bus')]
            private readonly CommandBusInterface $commandBus,
        ) {
            parent::__construct();
        }

        protected function configure(): void
        {
            $this
                ->addOption('shop', null, InputOption::VALUE_REQUIRED, 'Restrict to a single shop id.')
                ->addOption('dry-run', null, InputOption::VALUE_NONE, 'Print actions without persisting.');
        }

        protected function execute(InputInterface $input, OutputInterface $output): int
        {
            $io = new SymfonyStyle($input, $output);
            $shopConstraint = $input->getOption('shop')
                ? ShopConstraint::shop((int) $input->getOption('shop'))
                : ShopConstraint::allShops();

            $expired = $this->commandBus->handle(new ExpireVouchersCommand(
                shopConstraint: $shopConstraint,
                dryRun: (bool) $input->getOption('dry-run'),
            ));

            $io->success(sprintf('%d vouchers expired.', $expired));
            return Command::SUCCESS;
        }
    }
    ```

2. Verify `config/services.yml` has the standard skeleton from `module-create`. The `_defaults: { autoconfigure: true }` line is what causes Symfony to discover and tag the command. If you opted out of `autoconfigure`, add the `console.command` tag manually:

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

3. Run the command from the PrestaShop project root (NOT from the module folder). PS exposes `bin/console` at the install root - it boots the kernel with all enabled modules and their service definitions:

    ```bash
    cd /path/to/prestashop
    bin/console mymod:vouchers:expire --shop=2 --dry-run
    bin/console list mymod                 # list every command in the mymod namespace
    bin/console help mymod:vouchers:expire # full usage
    ```

4. For long-running batch commands, chunk work and flush per batch. Doctrine memory grows linearly otherwise:

    ```php
    foreach (array_chunk($ids, 500) as $batch) {
        foreach ($batch as $id) { /* ... */ }
        $this->em->flush();
        $this->em->clear();
    }
    ```

5. Schedule the command via cron (the merchant's responsibility). Document the recommended schedule in the module README:

    ```cron
    0 3 * * * php /var/www/prestashop/bin/console mymod:vouchers:expire --no-interaction
    ```

6. For one-off install-time or upgrade-time scripts that must run automatically, use the `upgrade/upgrade-<version>.php` mechanism described in `module-add-migration` instead of a CLI command. CLI commands are for repeatable operations the merchant invokes manually or via cron.

## Do

- Keep CLI logic THIN. The command's job is: parse input, build a Command/Query, dispatch to the bus, format the output.
- Reuse existing CQRS handlers. The same handler powers your admin controller, your API resource AND the CLI - one source of truth.
- Use `SymfonyStyle` (`$io->success`, `$io->table`, `$io->progressIterate`) for human-readable output.
- Return `Command::SUCCESS` / `Command::FAILURE` / `Command::INVALID` so cron and CI pipelines can detect failures.
- Take a `--shop` / `--all-shops` option for any multistore-aware command and convert it to a `ShopConstraint` before dispatch.

## Don't

- Don't call `Db::getInstance()->execute(...)` directly from the command. Persistence belongs in repositories or handlers.
- Don't read `Context::getContext()->shop` from inside a CLI command. The CLI has no HTTP request, so the static context is whatever `bin/console` boots it to. Pass the shop in explicitly.
- Don't echo to stdout via `print_r` / `var_dump`. Use `$output->writeln(...)` or `SymfonyStyle` so `--quiet` and `-v` flags work.
- Don't catch `\Throwable` and swallow it. Let exceptions bubble; the kernel returns a non-zero exit code automatically.
- Don't manually tag the service if `autoconfigure: true` is set - that double-registers the command and makes `list` show it twice.

## Canonical examples

- [devdocs - Modules / Commands](https://devdocs.prestashop-project.org/9/modules/concepts/commands/) - the contract for shipping CLI commands from a module.
- [devdocs - PrestaShop console commands](https://devdocs.prestashop-project.org/9/development/components/console/) - the index of every `prestashop:*` command shipped by the core (use them as patterns).
- [`PrestaShop/example-modules` - democonsolecommand](https://github.com/PrestaShop/example-modules/tree/master/democonsolecommand) - minimal working module that ships a console command via `#[AsCommand]` and `autoconfigure`.
- [`PrestaShop/PrestaShop` - Command directory](https://github.com/PrestaShop/PrestaShop/tree/develop/src/PrestaShopBundle/Command) - core commands written in the same style (`prestashop:feature-flag`, `prestashop:list:commands-and-queries`, `prestashop:translation:export-module`).
