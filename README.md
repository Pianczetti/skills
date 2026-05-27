# PrestaShop Skills

Welcome to the **PrestaShop Skills** repository! This repository serves as a
centralized collection of AI "skills" designed to help you interact with,
manage, and develop for PrestaShop. These skills cover various domains including
PrestaShop modules, core components, and themes.

## ⚠️ Security & Disclaimer

**Use with caution, especially in production environments.**
These skills provide instructions to AI agents that may execute CLI commands (e.g., `bin/console`), modify files, or trigger critical processes like backups and restorations.
- **Always review** the commands and actions proposed by your AI assistant before allowing it to execute them on a live or production store.
- We recommend testing skills on a staging or local environment first.

## ✅ Prerequisites

To use these skills, you will need:
- **Node.js / npm**: Required to run the `npx skills` command.
- **An AI Assistant / Agent**: A compatible AI development tool (such as Cursor, GitHub Copilot CLI, or other agentic frameworks) capable of reading and executing instructions from the installed skills.

## 🚀 Installation & Usage

You can easily install the skills you need by using the `npx skills` CLI. The
command will list the available skills in the targeted directory and install
them directly into the compatible AI assistants present on your system.

**General Command syntax:**

```bash
npx skills install PrestaShop/skills/[application]/[user|dev]
```

**Installation Example:** To install operational user skills for the
`autoupgrade` module:

```bash
npx skills install PrestaShop/skills/autoupgrade/user
```

_(For more details on the `npx skills` commands and mechanics, please refer to
the [vercel-labs/skills repository](https://github.com/vercel-labs/skills))._

## 📚 Available Skills Inventory

As this repository grows, this list will be updated. Here is a summary of the
skills currently available:

### Domain: `autoupgrade`

| Category | Skill Folder              | Skill Name                    | Description                                                                                                                                                              |
| -------- | ------------------------- | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `user`   | `prestashop-restore`      | **prestashop-store-rollback** | Restore a PrestaShop store to a previous backup. Use when you need to rollback, for example, if an update fails or if you need to restore a previous state of the store. |
| `user`   | `prestashop-update`       | **prestashop-store-update**   | Update a PrestaShop store by using the Module Update Assistant. Evaluates compatibility and proceeds with the upgrade.                                                   |
| `user`   | `prestashop-update-check` | **prestashop-store-check**    | Check if a PrestaShop store is ready to be updated. Assesses compatibility and available versions without starting the actual update.                                    |

### Domain: `module`

| Category | Skill Folder | Skill Name | Description |
| -------- | ------------ | ---------- | ----------- |
| `dev` | `module-add-admin-api-resource` | **module-add-admin-api-resource** | Expose a PrestaShop 9 module aggregate through the modern Admin API by declaring an API Platform resource that wires into the CQRS bus via #[CQRSGet], #[CQRSCreate], #[CQRSPartialUpdate] and #[CQRSDelete]. Use when the user wants a server-to-server REST endpoint authenticated with OAuth2 client credentials. |
| `dev` | `module-add-admin-controller-modern` | **module-add-admin-controller-modern** | Add a modern Symfony Back Office controller (PrestaShopAdminController) to a PrestaShop 9 module. Use when the user wants a back-office page, action, or AJAX endpoint served by their module. |
| `dev` | `module-add-carrier-module` | **module-add-carrier-module** | Scaffold a PrestaShop 9 carrier module that registers a Carrier ObjectModel on install, computes shipping cost per cart, and preserves order history on uninstall by soft-deleting the carrier. Use when the user needs to expose a new shipping option (DHL, UPS, click-and-collect, etc.). |
| `dev` | `module-add-cli-command` | **module-add-cli-command** | Ship a Symfony console command from a PrestaShop 9 module so the merchant can run a maintenance task, bulk import or scheduled job via bin/console <command>. Use when the user wants a CLI entry point that delegates to existing CQRS handlers and services. |
| `dev` | `module-add-config-page-legacy` | **module-add-config-page-legacy** | Build a legacy getContent() + HelperForm configuration page for an existing PrestaShop module. Use ONLY when the module must support PrestaShop < 8 or you are maintaining a legacy module that already uses this pattern. |
| `dev` | `module-add-config-page-modern` | **module-add-config-page-modern** | Build a modern Symfony Form + FormHandler + FormDataProvider configuration page for a PrestaShop 9 module. Use when the user wants a Back Office settings screen rendered from a modern admin controller. |
| `dev` | `module-add-cqrs-command` | **module-add-cqrs-command** | Scaffold a CQRS Command and CommandHandler in a PrestaShop 9 module to mutate state via the Symfony Messenger command bus instead of ObjectModel. Use when the user wants to create, update, delete or otherwise change an aggregate from a controller, console command, or hook listener. |
| `dev` | `module-add-cqrs-query` | **module-add-cqrs-query** | Scaffold a CQRS Query and QueryHandler in a PrestaShop 9 module to read data through the Symfony Messenger query bus instead of pulling Doctrine entities or ObjectModels directly into a controller. Use when the user wants to fetch data for a Back Office page, an API endpoint, or a presenter. |
| `dev` | `module-add-doctrine-entity` | **module-add-doctrine-entity** | Add a Doctrine ORM entity (with optional translatable fields) and a service repository to a PrestaShop 9 module, plus a SQL install script. Use when the user wants persistent storage for the module without going through legacy ObjectModel. |
| `dev` | `module-add-feature-flag` | **module-add-feature-flag** | Register a feature flag from a PrestaShop 9 module and gate runtime behaviour behind it. Use when the user wants to ship a risky or partially-finished feature that the merchant can toggle from the BO without redeploying the module. |
| `dev` | `module-add-front-controller` | **module-add-front-controller** | Add a ModuleFrontController to expose a front-office page or AJAX endpoint from a PrestaShop 9 module. Use when the user wants a customer-facing URL served by their module. |
| `dev` | `module-add-grid` | **module-add-grid** | Add a Back Office listing as a modern PrestaShop Grid (GridDefinitionFactory + Doctrine QueryBuilder + Twig macro), wired through service tags and rendered with the core grid template. Use when the user wants a sortable, filterable, paginated table in the Back Office instead of a HelperList. |
| `dev` | `module-add-mail-template` | **module-add-mail-template** | Add custom transactional or marketing mail templates to a PrestaShop 9 module and render them through MailTemplateRendererInterface, with translatable subject and body. Use when the user wants the module to send emails or extend the core mail themes. |
| `dev` | `module-add-migration` | **module-add-migration** | Ship idempotent schema and data migrations with a PrestaShop 9 module via versioned upgrade/upgrade-<version>.php scripts. Use when the user bumps the module version and needs to evolve the database (new column, new table, default-value change) without re-installing the module. |
| `dev` | `module-add-override` | **module-add-override** | Add a class or controller override under override/ in a PrestaShop 9 module as a LAST-RESORT extension mechanism. Use only when no hook, service decorator, CQRS command or event listener can express the change. |
| `dev` | `module-add-payment-module` | **module-add-payment-module** | Scaffold a PrestaShop 9 payment module that exposes payment options at checkout and validates orders through the canonical paymentOptions / validateOrder / displayPaymentReturn flow. Use when the user wants to add a new gateway (direct, redirect, iframe, off-site or webhook-confirmed). |
| `dev` | `module-add-service` | **module-add-service** | Register a Symfony service in a PrestaShop 9 module via config/services.yml using autowiring. Use when the user needs to inject a class as a dependency in controllers, hook listeners, or other services. |
| `dev` | `module-add-symfony-route` | **module-add-symfony-route** | Declare a Symfony route in a PrestaShop 9 module's config/routes.yml. Use when the user wants to expose a back-office or AJAX endpoint served by a modern controller. |
| `dev` | `module-add-translations-new` | **module-add-translations-new** | Wire the new (XLIFF + Symfony Translator) translation system into a PrestaShop 9 module. Use when the user wants translatable strings using `Modules.<Modulename>.<Area>` domains. |
| `dev` | `module-add-unit-tests` | **module-add-unit-tests** | Scaffold PHPUnit inside a PrestaShop 9 module so handlers, value objects and pure-PHP services can be unit-tested in isolation, without booting the shop. Use when the user wants `vendor/bin/phpunit` to run green from the module root. |
| `dev` | `module-add-vue-config-page` | **module-add-vue-config-page** | Build a Vue 3 SPA configuration page for a PrestaShop 9 module, bundled with Vite, mounted from a modern Symfony admin controller, and backed by Admin API resources. Use when the configuration UI is too rich for Symfony Forms (drag-and-drop, live preview, multi-step wizards, dashboards). |
| `dev` | `module-add-webservice-resource-legacy` | **module-add-webservice-resource-legacy** | Expose an entity through PrestaShop 9's legacy /api/ webservice by hooking addWebserviceResources from the module class. Use ONLY when integrating with a third-party that has not migrated to the modern Admin API. |
| `dev` | `module-add-widget` | **module-add-widget** | Implement PrestaShop\PrestaShop\Core\Module\WidgetInterface in a PrestaShop 9 module so it can render reusable, cache-friendly content from any display hook. Use when the user wants the module to expose a block (sidebar, header, footer, product page extra content, etc.). |
| `dev` | `module-bump-compatibility` | **module-bump-compatibility** | Update an existing PrestaShop module to support a newer minor or major (e.g. 8.x to 9.0, 9.0 to 9.1) - bump `ps_versions_compliancy`, migrate legacy patterns to modern equivalents, audit overrides and dependency constraints, and round-trip the install on the target. Use when the user wants to declare support for a new PS version. |
| `dev` | `module-create` | **module-create** | Scaffold a new PrestaShop 9 module with the modern Symfony layout (PSR-4 src/, services.yml, routes.yml, translations). Use when the user wants to create a brand-new module from scratch. |
| `dev` | `module-make-multistore-aware` | **module-make-multistore-aware** | Make a PrestaShop 9 module behave correctly across multiple shops and shop groups. Use this checklist whenever the module reads or writes Configuration, queries the database, exposes a BO admin page, or ships CQRS commands and queries. |
| `dev` | `module-package-zip` | **module-package-zip** | Build a clean, Marketplace-ready zip for a PrestaShop 9 module - no dev dependencies, no test artefacts, no dotfiles, with the correct `index.php` placeholders. Use right before publishing on the Marketplace, attaching to a GitHub release, or deploying to a managed shop. |
| `dev` | `module-register-hooks` | **module-register-hooks** | Register hook listeners and dispatch custom hooks from a PrestaShop 9 module the modern way, using HookDispatcherInterface instead of the legacy Hook::exec(). Use when the user wants the module to react to core events or expose its own extension points. |
| `dev` | `module-validate` | **module-validate** | Run the full pre-publication validation pipeline against a PrestaShop 9 module - composer manifest, install/uninstall round-trip, naming and legacy-link linters, plus the official Marketplace validator. Use before opening a release PR or submitting to the Addons Marketplace. |

### Domain: `theme`

| Category | Skill Folder | Skill Name | Description |
| -------- | ------------ | ---------- | ----------- |
| `dev` | `theme-accessibility-audit` | **theme-accessibility-audit** | Run an automated and manual WCAG 2.1 AA audit on every main PrestaShop 9 page family using axe-core (via Playwright or Puppeteer), Lighthouse and pa11y, plus a documented manual checklist for things tools can't catch (focus order, semantic structure, copy quality). Use before publishing a theme to the marketplace and as a CI gate on every PR. |
| `dev` | `theme-add-bem-component` | **theme-add-bem-component** | Add a Block / Element / Modifier styled component to a PrestaShop 9 theme - one block per SCSS file, no descendant selectors, design tokens via CSS custom properties (`--ps-*` or `--bs-*`), and a typed TypeScript class wired through `data-ps-component`. Use whenever the user introduces a new visual building block (product card, badge, accordion, mega-menu). |
| `dev` | `theme-add-layout` | **theme-add-layout** | Add a new page layout to a PrestaShop 9 theme - the `.tpl` shell, the `meta.available_layouts` declaration and the per-page assignment in `theme_settings`. Use when the user wants a new column arrangement, a checkout-specific shell or a marketing landing layout. |
| `dev` | `theme-add-page-section` | **theme-add-page-section** | Customise a specific PrestaShop 9 front-office page (home, category, product, cart, checkout, login, my account, contact) by injecting markup at a documented hook position rather than overriding the template. Use whenever the user wants to add or replace a section on one page family. |
| `dev` | `theme-add-theme-hook` | **theme-add-theme-hook** | Register a new theme-defined display hook in `config/theme.yml` (under `global_settings.hooks.custom_hooks`), call it from the right Smarty template, and document the payload modules will receive. Use when no built-in hook fits and you need a new extension point modules can attach to. |
| `dev` | `theme-add-translation` | **theme-add-translation** | Translate a PrestaShop 9 theme - wrap user-facing strings in templates with the `{l}` Smarty function under the right domain (`Shop.<Themename>` for theme-owned wordings, `Shop.Theme.*` to reuse core translations), then export the catalog from the Back Office. Use when the theme renders any English string and needs to support more than one language. |
| `dev` | `theme-asset-pipeline` | **theme-asset-pipeline** | Wire a PrestaShop 9 theme to a Node-based bundler (Hummingbird ships Webpack; new themes can pick Vite) with SCSS, TypeScript, ESLint, Stylelint, Prettier and a test runner, then declare the compiled outputs in `config/theme.yml` so PrestaShop loads them at the right priority. Use whenever a theme grows past plain `assets/css/theme.css` and needs a real build. |
| `dev` | `theme-bootstrap-compatibility` | **theme-bootstrap-compatibility** | Pin a PrestaShop 9 theme to a single supported Bootstrap version and decide how (or whether) to ship the Bootstrap Compatibility Layer that bridges Bootstrap 4 modules into a Bootstrap 5 theme. Use when the user starts a new PS 9 theme, upgrades a Classic-derived theme to Hummingbird, or hits broken modals / dropdowns coming from a third-party module. |
| `dev` | `theme-create-child-theme` | **theme-create-child-theme** | Create a child theme that inherits everything from a parent (usually `classic` or `hummingbird`) and only ships the overrides. Use this when the user wants small tweaks (a few templates, a stylesheet, a script) and wants to keep pulling parent updates. |
| `dev` | `theme-create-from-hummingbird` | **theme-create-from-hummingbird** | Bootstrap a new PrestaShop 9 theme by cloning Hummingbird (the official starter), renaming it and rebuilding the assets, with an upstream-sync workflow so security and accessibility fixes can flow back in. Use this whenever the user asks for "a new theme" without a hard reason to start from scratch. |
| `dev` | `theme-create-from-scratch` | **theme-create-from-scratch** | Scaffold a minimum viable PrestaShop 9 theme from an empty folder, with the strict directory layout, the required `config/theme.yml` keys and the activation flow. Use when the user wants full control over markup and assets and is OK starting blank instead of forking Hummingbird. |
| `dev` | `theme-data-ps-bindings` | **theme-data-ps-bindings** | Wire JavaScript behaviour to a PrestaShop 9 theme via `data-ps-component` / `data-ps-element` / `data-ps-action` attributes (the documented Hummingbird contract) instead of CSS class or `#id` selectors, so refactoring the styling never breaks the behaviour. Use whenever a BEM block needs interactivity. |
| `dev` | `theme-export` | **theme-export** | Package a PrestaShop 9 theme into a distributable zip via `bin/console prestashop:theme:export <name>`, then verify the archive contents (no `node_modules/`, no `.git`, no source SCSS / TS) and submit it to the PrestaShop Validator before publishing. Use when the theme has passed `theme-validate` and is ready for the marketplace or an internal release. |
| `dev` | `theme-override-template` | **theme-override-template** | Override a single template from your active theme - a front office Smarty `.tpl` or a back office Symfony Twig view shipped by a module. Use whenever the user wants to change rendered markup without forking the upstream theme or core. |
| `dev` | `theme-rtl-support` | **theme-rtl-support** | Add Right-to-Left language support to a PrestaShop 9 theme - rely on PrestaShop's automatic `*_rtl.css` discovery, ship a `rtl.css` override, fine-tune with `.rtlfix` files, and prefer logical CSS properties so layouts mirror without per-rule overrides. Use when the theme must serve Arabic, Hebrew, Persian, Urdu or any language whose `is_rtl` flag is on. |
| `dev` | `theme-storybook` | **theme-storybook** | Scaffold Storybook 8+ at the PrestaShop 9 theme root so BEM components render against fixture data outside the front office, with the same SCSS the production build uses, optional Chromatic visual review, and a static build that publishes to GitHub Pages or the marketplace listing. Use whenever the theme has more than a handful of BEM components and a designer or QA needs to review them in isolation. |
| `dev` | `theme-tdd-component` | **theme-tdd-component** | Drive a PrestaShop 9 theme component from a failing test - a Jest or Vitest unit test against a JSDOM fixture for behaviour, a Storybook story for visual review, and a Playwright E2E spec on a real PS instance for the full integration. Use whenever a new BEM component carries non-trivial JavaScript behaviour (cart updates, accordion, quick view, mobile menu). |
| `dev` | `theme-validate` | **theme-validate** | Validate a PrestaShop 9 theme's structure end-to-end before packaging - YAML lint on `config/theme.yml`, required keys, `preview.png` dimensions (500 x 746), every `assets.css` / `assets.js` path resolves to a real file, install / uninstall round-trip via the CLI, and the official online Validator. Use as the last gate before `theme-export`. |

## 🤖 Agent Definitions

Agents orchestrate multiple skills into complex, multi-step workflows. While a skill performs a single focused task, an agent sequences many skills to accomplish a high-level goal (e.g. building an entire module from requirements, or running a full QA pass on a theme).

| Agent | Description |
| ----- | ----------- |
| `module-creator` | Creates complete, production-ready PrestaShop 9 modules from scratch based on user requirements. |
| `module-legacy-migrator` | Migrates legacy PrestaShop modules to modern PS 9 architecture pattern by pattern. |
| `module-modifier` | Modifies existing modern PrestaShop 9 modules to add features, fix bugs, or extend functionality. |
| `theme-creator` | Creates new PrestaShop 9 themes from scratch or from Hummingbird with full asset pipeline and accessibility baseline. |
| `theme-legacy-migrator` | Migrates legacy/Classic-based PrestaShop themes to modern PS 9 Hummingbird architecture. |
| `theme-modifier` | Modifies existing PrestaShop 9 themes to add pages, change layouts, add components, and adjust styling. |
| `frontend-tester` | Tests PrestaShop 9 front-office themes with visual regression, accessibility, cross-browser, and performance audits. |
| `backend-tester` | Tests PrestaShop 9 module back-end with unit tests, integration tests, API tests, and security audits. |

Agent definitions live in the `agents/` directory. Each file describes the agent's role, the skills it uses, its standard workflow, and the rules it must follow.

## 🗂️ Repository Structure

The repository is organized by **application domains**. Currently, the supported
domains include:

- [`autoupgrade`](#domain-autoupgrade) (Module Update Assistant)
- [`module`](#domain-module) (PrestaShop 9 module development)
- [`theme`](#domain-theme) (PrestaShop 9 theme development)

_(More domains like core and specific modules will be added over time)._

Inside each application domain, skills are divided into two specific target
audiences:

- 🧑‍💻 **`user`**: Skills intended for store owners, merchants, and agencies.
  These allow you to perform operational actions on your PrestaShop store using
  AI (e.g., updating the store, checking compatibility, or restoring backups).
- 🛠️ **`dev`**: Skills intended for developers. These are tailored to help
  build, debug, and enhance the application domain itself.

**Directory tree example:**

```text
PrestaShop/skills/
├── agents/               # Agent definitions (orchestrate skills)
│   ├── module-creator.md
│   ├── module-legacy-migrator.md
│   ├── module-modifier.md
│   ├── theme-creator.md
│   ├── theme-legacy-migrator.md
│   ├── theme-modifier.md
│   ├── frontend-tester.md
│   └── backend-tester.md
├── autoupgrade/          # Domain
│   ├── user/             # User-facing operational skills
│   │   ├── prestashop-update/
│   │   ├── prestashop-restore/
│   │   └── ...
│   └── dev/              # Developer-facing skills
├── module/               # Domain
│   └── dev/              # Module development skills
│       ├── module-create/
│       ├── module-add-cqrs-command/
│       └── ...
├── theme/                # Domain
│   └── dev/              # Theme development skills
│       ├── theme-create-from-hummingbird/
│       ├── theme-add-bem-component/
│       └── ...
└── README.md             # This file
```

## 🤝 How to Contribute

Contributions to add new application domains, user skills, or developer skills are highly encouraged!

To create a new skill:
1. Create a new directory for your skill under the appropriate domain and category (e.g., `[domain]/[user|dev]/[skill-name]`).
2. Add a `SKILL.md` file in that directory.
3. Use the required frontmatter for the skill name and description:
   ```yaml
   ---
   name: your-skill-name
   description: A short description of what the skill does.
   ---
   ```
4. Write the detailed, step-by-step instructions for the AI to follow in the rest of the markdown file.
5. Open a Pull Request!

### 📖 Helpful Resources for Building Skills

If you are new to writing AI skills, here are some great resources and tutorials to help you understand how to write effective instructions for AI agents:
- [Anthropic Cookbook: Custom Skills Development](https://platform.claude.com/cookbook/skills-notebooks-03-skills-custom-development) - A great tutorial on how to structure and develop custom skills.
- [Vercel Labs Skills Repository](https://github.com/vercel-labs/skills) - The core engine that powers the `npx skills` command, containing examples and documentation on how the framework operates under the hood.
