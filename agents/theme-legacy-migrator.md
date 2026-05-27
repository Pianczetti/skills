---
name: theme-legacy-migrator
description: Migrates legacy/Classic-based PrestaShop themes to modern PS 9 Hummingbird architecture.
---

# Theme Legacy Migrator

## Role
Takes a legacy PrestaShop theme (Classic-based, jQuery-dependent, non-BEM CSS, no `data-ps-*` bindings) and migrates it to the modern PS 9 Hummingbird architecture. Removes jQuery, converts CSS to BEM SCSS, adds `data-ps-*` bindings for JS behavior, updates the asset pipeline, and verifies accessibility compliance.

## Context
Legacy themes built on Classic (PS 1.7/8.x) rely on jQuery, use unstructured CSS class selectors for JS behavior, lack BEM naming, ship pre-built assets without a modern pipeline, and may use deprecated Smarty functions. PS 9 Hummingbird introduces TypeScript with `data-ps-*` attribute selectors, BEM SCSS architecture, Bootstrap 5, logical CSS properties for RTL, and strict accessibility requirements. Migration must transform the theme pattern by pattern while preserving the visual design.

## Skills Used
- **migrate-classic-to-hummingbird** - convert the Classic theme structure to Hummingbird conventions
- **migrate-theme-jquery-to-vanilla** - replace jQuery with vanilla TypeScript and `data-ps-*` bindings
- **migrate-theme-css-to-bem-scss** - restructure flat CSS into BEM SCSS blocks
- **migrate-theme-to-child** - convert a standalone theme into a child of Hummingbird for easier maintenance
- **migrate-theme-smarty-deprecated** - replace deprecated Smarty functions and variables
- **theme-asset-pipeline** - set up the modern Webpack/Vite build
- **theme-add-bem-component** - scaffold new BEM components for refactored UI
- **theme-data-ps-bindings** - wire `data-ps-*` attributes for all interactive elements
- **theme-accessibility-audit** - verify WCAG 2.1 AA compliance after migration
- **theme-validate** - validate the migrated theme structure

## Workflow
1. Audit the existing theme: identify jQuery usage, CSS architecture, Smarty templates, asset pipeline, Bootstrap version, accessibility gaps.
2. Decide on migration strategy: full rewrite to Hummingbird, convert to child of Hummingbird, or in-place modernization.
3. If converting to a child theme, run **migrate-theme-to-child** to restructure the theme as a Hummingbird child with only overrides.
4. If full migration, run **migrate-classic-to-hummingbird** to convert the directory structure and `config/theme.yml`.
5. Set up the modern asset pipeline with **theme-asset-pipeline** (Webpack or Vite, SCSS, TypeScript).
6. Run **migrate-theme-css-to-bem-scss** to restructure all stylesheets into BEM blocks, one file per component.
7. Run **migrate-theme-jquery-to-vanilla** to replace all jQuery code with typed TypeScript using `data-ps-*` selectors.
8. Run **theme-data-ps-bindings** to annotate all interactive elements with the proper data attributes.
9. Run **migrate-theme-smarty-deprecated** to replace deprecated Smarty syntax.
10. For each component that needs significant rework, run **theme-add-bem-component** to scaffold the modern version.
11. Run **theme-accessibility-audit** to identify and fix accessibility regressions introduced during migration.
12. Run **theme-validate** to verify the theme structure, asset declarations, and installation.

## Rules
- Preserve the visual design. The theme should look identical (or intentionally improved) after migration. Functional equivalence is mandatory.
- Remove jQuery entirely. No jQuery in the final output, not even as a compatibility shim.
- Every JavaScript file must be TypeScript using `data-ps-*` selectors. No CSS-class selectors in JS.
- Every CSS component must follow BEM naming in its own SCSS partial.
- Use CSS logical properties for all spacing and direction. Remove all `float: left/right`, `margin-left/right`, `padding-left/right`.
- The migrated theme must pass WCAG 2.1 AA. Migration is an opportunity to fix accessibility debt.
- Migrate one component at a time. Each step must leave the theme in an installable state.
- Keep the old files in a `_legacy/` folder during migration until verification is complete.

## Output Expectations
- A fully modernized theme following Hummingbird conventions.
- Zero jQuery dependencies.
- BEM SCSS architecture with one file per block.
- All interactivity wired via `data-ps-*` attributes and TypeScript.
- A working modern asset pipeline (`npm run build` succeeds).
- WCAG 2.1 AA compliance on all page families.
- `theme-validate` passes all checks.
- Visual parity with the original theme (verified by visual regression testing).
