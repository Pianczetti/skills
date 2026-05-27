---
name: migrate-theme-to-child
description: Convert a fully-forked theme into a minimal child theme by diffing against the parent, keeping only overridden templates and assets, and declaring the parent in config/theme.yml. Use when a theme is a full fork that should inherit updates from its parent.
---

## Requirements

- Forked theme name and directory.
- Parent theme name and version (e.g. `classic`, `hummingbird`).
- Access to the parent theme source for diffing.
- List of intentional customizations vs. accidental copies.

## Steps

1. Obtain the parent theme source at the same version the fork was based on:

    ```bash
    git show origin/main:themes/classic > /tmp/classic-ref 2>/dev/null || \
      cp -r /path/to/prestashop/themes/classic /tmp/classic-ref
    ```

2. Diff every file in the forked theme against the parent to identify actual changes:

    ```bash
    diff -rq /tmp/classic-ref themes/mytheme/ | grep -v 'Only in' | grep 'differ'
    ```

3. For files that are identical to the parent, delete them from the child theme. The parent will serve them automatically:

    ```bash
    # For each identical file
    rm themes/mytheme/templates/catalog/product.tpl  # if unchanged
    ```

4. Update `config/theme.yml` to declare the parent:

    ```yaml
    parent: classic
    name: mytheme
    display_name: My Theme
    version: 1.0.0
    ```

5. Keep only these in the child theme:
   - `config/theme.yml` (with `parent:` declaration)
   - `preview.png`
   - Templates that differ from the parent (under `templates/`)
   - Custom assets not present in the parent (under `assets/`)
   - Module template overrides (under `modules/`)

6. Remove everything else: identical templates, parent assets, `node_modules/`, duplicated config sections that match the parent.

7. Verify the child theme resolves templates correctly:

    ```bash
    bin/console prestashop:theme:enable mytheme
    ```

8. Browse all major pages (home, category, product, cart, checkout, login, my-account) and confirm:
   - Overridden templates render your custom markup.
   - Non-overridden templates fall through to the parent.
   - Assets load correctly (child assets override, parent assets fill gaps).

9. Test that updating the parent theme propagates changes to non-overridden pages.

## Do

- Use `parent:` in `config/theme.yml`; this is the only required declaration for child theme inheritance.
- Keep `preview.png` in the child; it identifies the theme in the BO.
- Document which templates are intentionally overridden (add a comment at the top of each).

## Don't

- Don't keep files identical to the parent; they prevent parent updates from flowing through.
- Don't override `config/theme.yml` sections that match the parent (hooks, global_settings) unless intentionally changed.
- Don't remove `preview.png`; the BO theme selector requires it.
- Don't forget module template overrides under `modules/<mod>/views/templates/`; they stay in the child if customized.

## Related skills

- `theme-create-child-theme` - creating a child theme from scratch.
- `migrate-classic-to-hummingbird` - full architectural migration (alternative to child).
- `theme-validate` - validating the child theme structure.
