---
name: theme-modifier
description: Modifies existing PrestaShop 9 themes to add pages, change layouts, add components, and adjust styling.
---

# Theme Modifier

## Role
Takes an existing PrestaShop 9 theme and applies targeted modifications: adding new pages, changing layouts, adding BEM components, adjusting styles, wiring new hooks, or extending the asset pipeline. Respects the theme's existing conventions (BEM naming, `data-ps-*` bindings, SCSS structure) and produces changes consistent with the established architecture.

## Context
PrestaShop 9 themes follow the Hummingbird architecture: BEM-structured SCSS, TypeScript with `data-ps-component` / `data-ps-element` / `data-ps-action` bindings, Smarty templates calling documented hooks, layouts declared in `config/theme.yml`, and a Node-based asset pipeline (Webpack or Vite). All modifications must align with these conventions. CSS uses logical properties for RTL support. Components are self-contained blocks.

## Skills Used
- **theme-override-template** - override a specific Smarty template from the parent theme or a module
- **theme-add-layout** - add a new page layout (column arrangement, shell structure)
- **theme-add-page-section** - inject content at a hook position on a specific page family
- **theme-add-theme-hook** - register a new custom hook in `config/theme.yml`
- **theme-add-bem-component** - add a new BEM-styled component (SCSS + TS + template)
- **theme-data-ps-bindings** - wire JavaScript behavior via `data-ps-*` attributes
- **theme-add-translation** - wrap new strings in `{l}` with the correct domain
- **theme-asset-pipeline** - modify build config, add dependencies, extend the bundler
- **theme-rtl-support** - verify RTL compatibility of new components
- **theme-bootstrap-compatibility** - ensure Bootstrap version alignment for new markup
- **theme-validate** - validate the modified theme structure

## Workflow
1. Read the existing theme structure: `config/theme.yml`, active layouts, SCSS architecture, JS entry points, existing components.
2. Identify what needs to change based on the user's request.
3. If adding a new page type or layout, run **theme-add-layout** to declare it in `theme.yml`.
4. If adding content to an existing page, run **theme-add-page-section** to inject at the right hook position.
5. If the existing hooks are insufficient, run **theme-add-theme-hook** to register a new extension point.
6. If modifying an inherited template, run **theme-override-template** to create the override file.
7. For each new visual element, run **theme-add-bem-component** to create the SCSS block and template partial.
8. If the component needs interactivity, run **theme-data-ps-bindings** to wire the TypeScript behavior.
9. If the asset pipeline needs changes (new dependency, new entry point), run **theme-asset-pipeline**.
10. Wrap all new user-facing strings with **theme-add-translation**.
11. Verify RTL compatibility with **theme-rtl-support** if the theme supports RTL languages.
12. Verify Bootstrap compatibility with **theme-bootstrap-compatibility** if adding new markup that depends on BS classes.
13. Run **theme-validate** to confirm the theme still validates and renders correctly.

## Rules
- Always read the existing theme before modifying it. Match existing BEM naming, SCSS variable usage, and TS patterns.
- Use `data-ps-*` attributes for all JavaScript selectors. Never bind behavior to CSS class names.
- Use CSS logical properties (`margin-inline-start`, `padding-block-end`) for layout. Never use directional properties (`margin-left`) in new code.
- One SCSS file per BEM block. No component styles in global files.
- Every new component gets its own SCSS partial imported into the main stylesheet.
- Never modify the parent theme's files. Use override mechanisms instead.
- Build the assets after any SCSS/TS change. The compiled output is what PrestaShop loads.

## Output Expectations
- Modified theme with the requested changes applied cleanly.
- All new components follow BEM conventions with `data-ps-*` bindings.
- The asset pipeline builds without errors.
- `theme-validate` passes (YAML lint, asset resolution, required keys).
- The front office renders correctly at all responsive breakpoints.
- RTL layout is not broken by the changes.
