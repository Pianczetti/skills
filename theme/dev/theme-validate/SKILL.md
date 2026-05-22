---
name: theme-validate
description: Validate a PrestaShop 9 theme's structure end-to-end before packaging - YAML lint on `config/theme.yml`, required keys, `preview.png` dimensions (500 x 746), every `assets.css` / `assets.js` path resolves to a real file, install / uninstall round-trip via the CLI, and the official online Validator. Use as the last gate before `theme-export`.
---

## Requirements

Ask the user:
* Path to the theme on disk (typically `themes/<name>/` inside a PS install).
* Target PrestaShop version (must be inside the `meta.compatibility.from`/`to` range declared in `theme.yml`).
* Whether the validation runs locally only, or also against the [online Validator](https://validator.prestashop.com/).
* Whether the round-trip will be performed on a disposable PS instance (recommended). The activation step replaces the shop's image presets - never validate on a live store.

## Steps

1. Lint `config/theme.yml` as YAML. Any structural error fails the activation, so catch it before:

    ```bash
    yamllint -d "{extends: default, rules: {line-length: disable}}" config/theme.yml
    # or
    npx js-yaml config/theme.yml > /dev/null
    ```

2. Confirm the **required keys** are present and well-formed:

    | Key                                  | Notes                                                        |
    |--------------------------------------|--------------------------------------------------------------|
    | `name`                               | lower_snake_case, MUST match the directory under `themes/`.  |
    | `display_name`                       | Human-readable label shown in the BO theme picker.          |
    | `version`                            | Semver string.                                               |
    | `author.name`                        | Required; `email` and `url` are optional but recommended.    |
    | `meta.compatibility.from` / `.to`    | PS version range. Activation refuses themes outside it.      |
    | `meta.available_layouts`             | At least one entry.                                          |
    | `global_settings.image_types`        | Required; activation REPLACES the shop's image presets.     |
    | `global_settings.hooks.custom_hooks` | Required when the theme declares its own theme hooks.        |
    | `theme_settings.default_layout`      | Must reference a key under `meta.available_layouts`.         |
    | `assets.css` / `assets.js`           | Optional; every entry must resolve to an existing file.      |

    Note: there is no `assets.rtl_css` key. RTL stylesheets are picked up by filename suffix (`*_rtl.css`); see `theme-rtl-support`.

3. Confirm `preview.png` exists at the theme root and measures **500 x 746** (the BO theme picker thumbnail):

    ```bash
    file preview.png            # expect "PNG image data, 500 x 746"
    identify -format '%w x %h\n' preview.png   # ImageMagick alternative
    ```

    A wrong size or a missing `preview.png` makes the BO render a placeholder; the Validator flags it.

4. Resolve every asset path. Loop over `config/theme.yml`'s `assets.css` and `assets.js` entries and confirm each `path` exists relative to the theme root:

    ```bash
    yq '.assets.css[].path, .assets.js[].path' config/theme.yml \
      | while read -r p; do test -f "$p" || echo "MISSING: $p"; done
    ```

    Treat any "MISSING" line as a fatal error; PrestaShop fails the page render when a registered asset 404s.

5. Round-trip the theme through the CLI on a disposable PS install. This catches `image_types` collisions, missing layouts, broken hooks and YAML schema errors that the static checks miss:

    ```bash
    bin/console prestashop:theme:enable mytheme       # activate
    # ... browse a couple of pages, confirm they render ...
    bin/console prestashop:theme:enable classic       # restore the default
    ```

    Watch the console output. The activator surfaces every validation failure (missing template directory, unknown layout, malformed hook definition).

6. Run the official online Validator at [validator.prestashop.com](https://validator.prestashop.com/). Sign in, upload the zip produced by `theme-export`, and confirm the report is clean. The Validator runs the same structural checks as the activator plus marketplace policies (forbidden directories, file-size budgets, `index.php` presence in every directory).

7. Smoke-test the templates. With Developer Mode on (`_PS_MODE_DEV_ = true`, BO Performance: Force compilation Yes / Cache No), open every main page family (home, category, product, cart, checkout, login, my account, contact, 404). Confirm:
    * No Smarty error overlays.
    * View-source shows the theme's compiled `theme.css` and `theme.js`.
    * Module hooks render at the documented positions (see `theme-add-page-section`).

8. Verify:
    * `yamllint`, asset-path resolution and `preview.png` checks all pass.
    * `bin/console prestashop:theme:enable <name>` succeeds and `bin/console prestashop:theme:enable classic` restores cleanly.
    * The online Validator reports no errors.
    * Every page family renders without Smarty / PHP notices in `var/logs/`.

## Do

- Always validate on a disposable PS install. Activation **replaces** `global_settings.image_types`; running the round-trip on a live store would resize every product image.
- Verify `preview.png` is exactly 500 x 746 (BO theme picker thumbnail). Earlier docs mentioned 1110 x 800 - the current marketplace contract is 500 x 746.
- Re-run the asset-path check after every build. A renamed entry point in `webpack.config.js` is silently broken until a page tries to load the asset.
- Capture the Validator report in the PR description so reviewers see the same evidence.

## Don't

- Don't activate the theme on a production shop just to validate. Use a copy of the database.
- Don't add `assets.rtl_css` to `theme.yml`; PrestaShop ignores it. RTL siblings are filename-discovered (`theme_rtl.css`).
- Don't skip the round-trip back to `classic`. A broken uninstall leaves the shop in the new theme even when activation rolled back; the round-trip catches that.
- Don't manually patch the activator's error output (e.g. delete a missing template). Fix the source so the next build is clean.

## Canonical examples

- [devdocs - Theme validation](https://devdocs.prestashop-project.org/9/themes/distribution/validation/) - the structural rules the activator enforces.
- [devdocs - Theme structure](https://devdocs.prestashop-project.org/9/themes/concepts/theme-structure/) - full `theme.yml` reference (compatibility, layouts, configuration, modules, hooks, image_types, dependencies).
- [devdocs - `prestashop:theme:enable`](https://devdocs.prestashop-project.org/9/development/components/console/prestashop-theme-enable/) - CLI activation, error semantics and rollback.
- [PrestaShop Validator](https://validator.prestashop.com/) - the official online checker for marketplace submissions.
- [`PrestaShop/hummingbird` - `config/theme.yml`](https://github.com/PrestaShop/hummingbird/blob/develop/config/theme.yml) - a real, validator-clean `theme.yml` to diff against.
- [`PrestaShop/hummingbird` - `preview.png`](https://github.com/PrestaShop/hummingbird/blob/develop/preview.png) - reference 500 x 746 thumbnail.

## Related skills

- `theme-export` - the next step once validation passes; produces the marketplace-ready zip.
- `theme-asset-pipeline` - source of the `assets/css/*.css` and `assets/js/*.js` files referenced in `theme.yml`.
- `theme-rtl-support` - explains why no `assets.rtl_css` key is needed.
- `theme-create-from-scratch` and `theme-create-from-hummingbird` - upstream skills that produce the theme tree this skill validates.
