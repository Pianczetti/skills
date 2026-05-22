---
name: migrate-theme-to-child
description: Convert a fully-forked theme into a proper child theme that inherits from its parent, keeping only actual overrides. Use when a theme was forked wholesale and maintaining it against parent updates has become costly.
---

## Requirements

- Identify the forked theme directory and its original parent (Classic or Hummingbird).
- Confirm the parent theme is still installed under `themes/`.
- Assess the diff size between fork and parent to estimate effort.

## Steps

1. **Diff the fork against the parent** to find actual customizations:

    ```bash
    diff -rq themes/myfork/templates themes/hummingbird/templates | grep 'differ\|Only in myfork'
    diff -rq themes/myfork/config themes/hummingbird/config | grep 'differ'
    ```

    Files reported as identical can be deleted from the child; they will fall through to the parent.

2. **Create the child theme directory:**

    ```bash
    mkdir -p themes/myfork-child/config
    mkdir -p themes/myfork-child/templates
    mkdir -p themes/myfork-child/assets
    ```

3. **Configure `theme.yml`** with the `parent` key:

    ```yaml
    # themes/myfork-child/config/theme.yml
    name: myfork-child
    display_name: My Fork (Child)
    parent: hummingbird          # MUST match parent's directory name
    version: 1.0.0
    author:
      name: "Your Name"
    meta:
      compatibility:
        from: 9.0.0
        to: ~9.1.0
    ```

4. **Copy only override files.** For each file that differs from the parent:
   - Templates: copy to the same relative path in `themes/myfork-child/templates/`.
   - Assets (compiled): copy only if custom; otherwise the child inherits parent assets.
   - Modules templates: copy `themes/myfork-child/modules/<module>/views/templates/` overrides.

    ```bash
    # Example: copy only differing templates
    for f in $(diff -rq themes/myfork/templates themes/hummingbird/templates \
      | grep 'differ' | awk '{print $2}'); do
      rel="${f#themes/myfork/}"
      mkdir -p "themes/myfork-child/$(dirname "$rel")"
      cp "$f" "themes/myfork-child/$rel"
    done
    ```

5. **Handle assets.** If the child has custom SCSS/JS:
   - Copy `src/` and `package.json` to the child.
   - Import parent partials where possible rather than duplicating.
   - Build: `cd themes/myfork-child && npm ci && npm run build`.

    If no asset customization exists, the child inherits the parent build output automatically.

6. **Delete everything identical to the parent.** The child should contain ONLY:
   - `config/theme.yml` (with `parent:`)
   - Template files that differ
   - Custom asset source and build output (if any)
   - `preview.png`

7. **Activate and test:**

    ```bash
    bin/console prestashop:theme:enable myfork-child
    ```

    Verify template inheritance: modify a parent template and confirm the child renders the parent version for non-overridden pages.

8. **Verify the inheritance chain.** PS resolves templates in order:
   1. `themes/myfork-child/templates/<path>`
   2. `themes/hummingbird/templates/<path>`

    If a template is missing from the child, the parent version is used. This is the mechanism that reduces maintenance.

## Do

- Keep the child as thin as possible; fewer overrides means easier parent updates.
- Re-diff against the parent after each parent update to remove overrides that are no longer needed.
- Use `{extends file='parent:...'}` in Smarty when you only need to override one block.

## Don't

- Don't copy files that are byte-identical to the parent; they add maintenance burden with no benefit.
- Don't forget the `parent:` key in `theme.yml`; without it PS treats the theme as standalone.
- Don't override `_partials/` templates unless absolutely necessary; they change frequently across parent versions.
- Don't delete the parent theme from `themes/`; the child depends on it at runtime.

## Canonical examples

- [devdocs - Parent/child feature](https://devdocs.prestashop-project.org/9/themes/reference/template-inheritance/parent-child-feature/)
- [devdocs - Theme inheritance](https://devdocs.prestashop-project.org/9/themes/create-a-theme/from-hummingbird/)

## Related skills

- `theme-create-child-theme` - creating a child theme from scratch.
- `theme-override-template` - overriding a single template in a child.
- `migrate-classic-to-hummingbird` - full theme migration when child is not enough.
