---
name: theme-export
description: Package a PrestaShop 9 theme into a distributable zip via `bin/console prestashop:theme:export <name>`, then verify the archive contents (no `node_modules/`, no `.git`, no source SCSS / TS) and submit it to the PrestaShop Validator before publishing. Use when the theme has passed `theme-validate` and is ready for the marketplace or an internal release.
---

## Requirements

Ask the user:
* Theme technical name (the directory name under `themes/`, must match the `name:` key in `config/theme.yml`).
* Target environment: **Marketplace** (strict policies, Validator must be clean), **internal** (less strict but still keep `node_modules/` out), **client handover** (zip handed off, sometimes wants the source - state the policy explicitly).
* Whether the build outputs are committed (recommended). The export only zips what is on disk; missing compiled assets translates to a broken theme.
* Whether to bump `version` in `config/theme.yml` (and `package.json`) before exporting. Marketplace re-submissions need a unique version.

## Steps

1. Pre-flight. Run `theme-validate` and `theme-asset-pipeline` first. The export does not validate, it only zips:

    ```bash
    npm run build                                          # rebuild assets/
    bin/console prestashop:theme:enable mytheme           # round-trip activate
    bin/console prestashop:theme:enable classic           # ... and back
    ```

2. Run the export command from the PrestaShop root. It writes a zip under `var/themes/`:

    ```bash
    bin/console prestashop:theme:export mytheme
    # ...
    # Theme has been successfully exported to var/themes/mytheme.zip
    ```

    The exporter automatically excludes development-only directories (`_dev/` if present, `node_modules/`, `.git/`, `.github/`, `tests/`) and keeps everything PrestaShop loads at runtime.

3. Verify the zip contents. Open it (`unzip -l var/themes/mytheme.zip | less`) and confirm:

    ```
    mytheme/config/theme.yml             # required
    mytheme/preview.png                  # 500 x 746 PNG, see theme-validate
    mytheme/templates/...                # full Smarty tree
    mytheme/assets/css/theme.css         # built artefacts
    mytheme/assets/js/theme.js
    mytheme/modules/.../templates/...    # module template overrides, if any
    mytheme/index.php                    # blank guard file at every directory
    mytheme/Resources/locale/...         # XLIFF translation catalogues
    ```

    None of the following should appear in the archive:

    * `_dev/` (legacy source directory if you happened to keep one).
    * `node_modules/`.
    * `package-lock.json` is fine to ship; lockfiles inside `node_modules/` are not.
    * `.git/`, `.github/`, `.idea/`, `.vscode/`.
    * `*.map` sourcemaps.
    * `tests/`, `playwright-report/`, `coverage/`, `storybook-static/`.
    * Editor scratch files (`*.swp`, `.DS_Store`, `Thumbs.db`).

    If any of those slipped in, add the matching pattern to the theme's `.gitignore` AND rebuild from a clean checkout - the exporter zips what is on disk.

4. Confirm an `index.php` guard file lives in every directory of the archive. The PrestaShop standard ships a one-line `header('Location: ../');` redirect inside each folder to prevent directory listing; the exporter generates them when missing, but legacy themes sometimes have folders without one:

    ```bash
    unzip -l var/themes/mytheme.zip | awk '{print $4}' | grep -E '/$' \
      | while read dir; do unzip -l var/themes/mytheme.zip | grep -q "${dir}index.php" || echo "MISSING index.php in $dir"; done
    ```

5. Submit to the PrestaShop Validator at [validator.prestashop.com](https://validator.prestashop.com/) for any external publication. The Validator reruns structural checks plus marketplace policies (forbidden binaries, file-size budgets, naming rules). Capture the report and address every error and warning before publication.

6. Tag the release in git so the version in `config/theme.yml` matches a permanent reference:

    ```bash
    git tag -a v$(yq .version config/theme.yml) -m "Release $(yq .version config/theme.yml)"
    git push --tags
    ```

7. Verify:
    * `unzip -l var/themes/<name>.zip` shows the structure above and **no** dev-only directories.
    * Uploading the zip via Back Office > Design > Theme & Logo > Add new theme installs the theme on a fresh PS install.
    * Activating the freshly-installed theme renders the front office without errors (re-run `theme-validate`'s smoke test).
    * The Validator report is clean.

## Do

- Run `npm run build` immediately before `prestashop:theme:export`. The exporter zips disk state, not source state.
- Bump `version` in `config/theme.yml` (and `package.json`) for every release. The marketplace refuses duplicate versions.
- Keep `.gitignore` and the exporter exclusion patterns aligned. Anything you do not want in the zip should also not be in git.
- Smoke-test the resulting zip on a fresh PS install before submitting it. The Validator catches structure but not "every page renders".

## Don't

- Don't ship `_dev/`, `node_modules/`, `.git/`, `tests/`, `storybook-static/`, `coverage/`, `playwright-report/` or sourcemaps. They bloat the archive and the Validator rejects them.
- Don't ship uncompiled SCSS or TypeScript expecting merchants to build them. The marketplace contract is "compiled assets only".
- Don't reuse a marketplace version number after a hotfix. Bump the patch version (1.0.0 -> 1.0.1) so merchants see the update.
- Don't hand-edit the zip after export. Re-export from a clean source tree instead.

## Canonical examples

- [devdocs - Theme exporting](https://devdocs.prestashop-project.org/9/themes/distribution/exporting/) - the exporter command, its output location, and what it includes / excludes.
- [devdocs - `prestashop:theme:export`](https://devdocs.prestashop-project.org/9/development/components/console/prestashop-theme-export/) - command reference (arguments, exit codes).
- [devdocs - Theme validation](https://devdocs.prestashop-project.org/9/themes/distribution/validation/) - structural rules the Validator re-applies post-export.
- [PrestaShop Validator](https://validator.prestashop.com/) - online checker for marketplace submissions.
- [`PrestaShop/hummingbird` - `config/theme.yml`](https://github.com/PrestaShop/hummingbird/blob/develop/config/theme.yml) - shipping `theme.yml` with a real `version`, `meta.compatibility`, and asset declarations.

## Related skills

- `theme-validate` - run before exporting; the exporter does not validate.
- `theme-asset-pipeline` - source of the compiled `assets/` the exporter zips.
- `theme-rtl-support` - confirms `*_rtl.css` siblings are present in the build before they ship.
- `theme-create-from-hummingbird` - whose `.gitignore` already excludes the right development directories.
