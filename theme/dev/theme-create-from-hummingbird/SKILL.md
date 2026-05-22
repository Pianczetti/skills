---
name: theme-create-from-hummingbird
description: Bootstrap a new PrestaShop 9 theme by cloning Hummingbird (the official starter), renaming it and rebuilding the assets, with an upstream-sync workflow so security and accessibility fixes can flow back in. Use this whenever the user asks for "a new theme" without a hard reason to start from scratch.
---

## Requirements

Ask the user:
* Target theme name (lower_snake_case, MUST match the directory under `themes/` and the `name:` key in `theme.yml`).
* Display name, author and starting version.
* Target PrestaShop version. Hummingbird is pinned to specific PrestaShop releases - check the compatibility matrix in the Hummingbird README before picking a branch.
* Whether to keep the bundled toolchain (Webpack + SCSS + TypeScript + Storybook). Recommended yes; if the user wants to drop it, redirect to `theme-create-from-scratch`.
* Whether to retain a link to upstream Hummingbird so security patches can be pulled in (recommended).

## Steps

1. From your PrestaShop install's `themes/` directory, clone Hummingbird and detach (or rebase) the git history:

    ```bash
    cd themes/
    git clone https://github.com/PrestaShop/hummingbird.git mytheme
    cd mytheme
    rm -rf .git && git init
    git add -A && git commit -m "Initial commit from Hummingbird"
    ```

    `rm -rf .git` discards the upstream history and gives you a clean repo. If you instead want to track upstream long-term, replace that line with `git remote rename origin upstream && git remote add origin <your-repo-url>` so `upstream/develop` stays available for `git merge upstream/develop` or cherry-picks.

2. Rename the theme. The folder name, the `name:` field in `config/theme.yml` and the `name` field in `package.json` must agree:

    ```yaml
    # config/theme.yml
    name: mytheme              # MUST match the directory
    display_name: My Theme
    version: 1.0.0
    author:
      name: "Your Name"
    meta:
      compatibility:
        from: 9.1.0
        to: ~9.1.0
        framework: bootstrap-v5.3.3
    ```

    ```json
    // package.json
    {
      "name": "mytheme",
      "version": "1.0.0"
    }
    ```

    Grep the source tree for any branding strings you want to swap (`hummingbird`, `Hummingbird`, `data-ps-theme="hummingbird"`):

    ```bash
    grep -rIn 'hummingbird' src/ templates/ config/ package.json README.md
    ```

3. Install dependencies and build the assets. The toolchain lives at the theme root, not in `_dev/`. Hummingbird targets Node 20.x (a `.nvmrc` is shipped):

    ```bash
    nvm use            # or: node --version  # must be 20.x
    npm ci             # exact lockfile install
    npm run build      # compiles src/ into assets/
    ```

    For day-to-day work use `npm run watch` to recompile on save, and `npm run storybook` to develop components in isolation. `assets/` is the build output - never edit it by hand.

4. Activate the theme. Two paths:

    * Back Office: Design > Theme & Logo > Add new theme, upload the zipped folder, then Use this theme. The folder is already inside `themes/` so you can also pick it directly.
    * CLI: `bin/console prestashop:theme:enable mytheme` from the PrestaShop root.

5. Set up the upstream sync workflow if you want Hummingbird's bug fixes and accessibility patches:

    ```bash
    git remote add upstream https://github.com/PrestaShop/hummingbird.git
    git fetch upstream
    git merge upstream/develop                  # or cherry-pick specific commits
    npm ci && npm run build                     # always rebuild after a sync
    ```

    Keep your customisations in dedicated `src/scss/_<feature>.scss` partials, dedicated TS modules and dedicated `templates/` overrides so merges from upstream stay clean.

## Do

- Track upstream Hummingbird (as `upstream` or `origin`) so security and accessibility fixes flow into your fork. The Hummingbird team ships them on the same branch that matches your PrestaShop version.
- Prefer overriding Bootstrap variables and SCSS partials before forking a component. The "Customization approach" section of the from-Hummingbird devdocs page lists the right escalation order: variables -> overrides -> custom components.
- Use the `data-ps-*` attributes Hummingbird ships for JS hooks. They are the documented contract for behaviour and survive markup tweaks.
- Run `npm run lint` and `npm run format` before committing - Hummingbird ships ESLint, Stylelint and Prettier configs you inherit for free.

## Don't

- Don't strip Hummingbird's accessibility primitives (skip links, ARIA landmarks, focus management, keyboard handlers in `src/js/`). Removing them silently breaks WCAG compliance and the `npm run lint` rules will not catch it.
- Don't edit files under `assets/`. They are the compiled output of `npm run build`. Edit `src/` and rebuild.
- Don't delete `.nvmrc`, `webpack.config.js`, `tsconfig.json` or `package-lock.json`. They pin the build environment.
- Don't bump `meta.compatibility.from` past your target PrestaShop version. Hummingbird's markup tracks core changes per PrestaShop release.

## Canonical examples

- [devdocs - Starting from Hummingbird](https://devdocs.prestashop-project.org/9/themes/create-a-theme/from-hummingbird/) - the source for the clone-and-rename workflow used here.
- [devdocs - Quick start](https://devdocs.prestashop-project.org/9/themes/getting-started/quick-start/) - the 5-minute version (clone, edit `theme.yml`, `npm ci && npm run build`, activate).
- [devdocs - Hummingbird development workflow](https://devdocs.prestashop-project.org/9/themes/hummingbird/development-workflow/) - script commands (`build`, `watch`, `lint`, `format`, `storybook`) and Smarty/CCC cache toggles for dev mode.
- [devdocs - Hummingbird requirements](https://devdocs.prestashop-project.org/9/themes/getting-started/requirements/) - Node 20.x, npm and the supported PrestaShop matrix.
- [`PrestaShop/hummingbird`](https://github.com/PrestaShop/hummingbird) - upstream repo (track this as `upstream`).
- [`PrestaShop/hummingbird` - README](https://github.com/PrestaShop/hummingbird/blob/develop/README.md) - Docker compose, Storybook and the version compatibility matrix.

## Related skills

- `theme-create-from-scratch` - go bare metal when Hummingbird's toolchain is overkill.
- `theme-create-child-theme` - extend Hummingbird without forking it (better when you only need small tweaks).
- `theme-override-template` - swap a single template inside the forked theme.
- `theme-add-layout` - register a new layout in `theme.yml`.
