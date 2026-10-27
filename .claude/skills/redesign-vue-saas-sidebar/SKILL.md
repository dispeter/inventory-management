---
name: redesign-vue-saas-sidebar
description: Use when asked to modernize, restyle, or polish a Vue 3 app's UI, replace a top navigation bar with a left vertical sidebar, or give an app a modern SaaS dashboard look with consistent spacing.
---

# Redesign a Vue 3 App into a SaaS Sidebar Layout

## Overview

Change the shell and the visual system, not the behavior. The result is a left sidebar, a slim top bar, and a content area built on one set of design tokens (spacing, color, radius, shadow). Routes, props, emits, API calls, and i18n keys stay as they are.

## Process

1. **Audit before editing.** Read the router file, the root component (`App.vue`), global styles, and project instructions (`CLAUDE.md`). Note the nav links, which controls live in the header (language switcher, profile menu, filters), and any palette or "no emojis" rules. Existing rules win over this skill.
2. **Define tokens once.** Put CSS variables in the root component's unscoped style block: a 4px spacing scale (`--space-1` 4px through `--space-8` 32px), radii (6px controls, 10px cards), 2 shadow levels, the existing palette, `--sidebar-width: 240px`. Replace hard-coded values in the shell with these. Do not rename or restructure view-level classes.
3. **Rebuild the shell** in the root component:
   - Layout: CSS Grid, `grid-template-columns: var(--sidebar-width) 1fr`, `min-height: 100vh`.
   - Sidebar: `position: sticky; top: 0; height: 100vh; overflow-y: auto`. Brand at top, nav list in the middle, profile and language at the bottom.
   - Nav items: inline SVG icon (16-20px, `currentColor`), label from the existing i18n key, 40px row height, 8px gap. Active state uses `router-link-active` (or `exact-active-class` for `/`): tinted background, accent text, no underline bar.
   - Top bar: slim (56-64px), page-level controls only (filters, search). Move anything that was in the old header here or into the sidebar footer. Keep components and their events intact.
   - Content: `padding: var(--space-6) var(--space-8)`, `max-width` container, vertical rhythm from the scale.
4. **Normalize shared surfaces** (cards, tables, badges, page headers, stat tiles) to the tokens: same card padding and border, one table row height, one heading scale. Touch view files only to remove one-off spacing that fights the tokens.
5. **Make it responsive.** Below ~1024px collapse the sidebar to an off-canvas drawer with a toggle button in the top bar. Close it on route change.
6. **Verify in a browser** (see below) before saying it is done.

## Quick Reference

| Concern | Rule |
|---------|------|
| Spacing | Only scale values, no ad-hoc `13px` or `1.1rem` |
| Icons | Inline SVG, `currentColor`. No icon library unless already installed. No emoji |
| Active link | Driven by the router, not by comparing `$route.path` strings |
| Accessibility | `<nav aria-label>`, `aria-current="page"`, visible `:focus-visible` ring, 4.5:1 text contrast |
| Long labels | Check the longest locale (Japanese or German) at 240px; truncate with ellipsis and `title` |
| Modals | Keep their z-index above the sidebar; fixed overlays must cover the whole viewport, not just the content column |

## Verification

1. Start the dev server and open every route.
2. Screenshot at 1440px and 768px. Confirm: sidebar fixed while content scrolls, active item matches the route, no horizontal page scroll, tables scroll inside their own container.
3. Open one modal and the profile menu; confirm they layer above the sidebar.
4. Switch language; confirm nav labels fit.
5. Console shows no new errors or warnings (compare against the baseline taken before editing).

## Common Mistakes

- **Rewriting views.** Restyling should not change data flow. If a view's script block changes, stop and re-check scope.
- **Leaving the old header.** Delete the old top-nav markup and its CSS, including `sticky` and height assumptions in other files.
- **`100vh` plus padding overflow.** Use `min-height: 100vh` on the grid and `box-sizing: border-box` everywhere.
- **Hard-coded colors left behind.** Search for hex values in the shell after the change; they should reference variables.
- **Sidebar over modals.** A sidebar with a high z-index hides modal overlays.
- **New dependencies.** A UI kit or icon package is a separate decision for the user, not part of this skill.

## Repo Rules

If the project requires specialist delegation or review for `.vue` edits (for example a `vue-expert` agent), follow that. This skill describes what to build, not who builds it.
