# Working agreement for this repo

## Ship every update to the live demo

The prototype is a single static file (`index.html`) served via GitHub Pages
from `main`: https://damo1410.github.io/shopify-style-guide-prototype/

For **every** change, after the work is done and committed on the session's
feature branch:

1. Merge the feature branch into `main` (`git merge --no-ff`) and push `main`.
2. If the merge reports **conflicts**, stop and report them to the user with
   the conflicting files/hunks — do not try to resolve them unilaterally, since
   other sessions may be editing other parts of the prototype concurrently.
3. Give the user a **cache-busted view URL** so they see the latest without a
   stale browser cache — the Pages URL with the short commit SHA as a query
   param, e.g. `https://damo1410.github.io/shopify-style-guide-prototype/?v=<sha>`.

Concurrent sessions are expected. Each merges its own branch into `main`;
conflicts only surface for overlapping edits, and those get surfaced to the
user rather than auto-resolved.

## Operational notes (don't relearn the hard way)

- **`.nojekyll` must stay.** GitHub Pages runs Jekyll by default, which parses
  every file through Liquid and **fails the build** on the literal `{% schema %}`
  / `{% form %}` etc. we document. `.nojekyll` disables that. Without it the
  site silently freezes on the last good build.
- **A green `git push` is NOT a green deploy.** Pages builds can fail. After
  pushing `main`, verify the build via the GitHub MCP (the
  `pages build and deployment` run for the new SHA must be `completed/success`)
  before telling the user it's live. The MCP `actions_list` result is huge —
  save it and parse with python, don't read it raw.
- **Test without a browser via jsdom.** There's no headless browser here, but
  `npm i jsdom` in `/tmp` works. Load `index.html` with `runScripts:'dangerously'`,
  shim `matchMedia`/`scrollIntoView`/pointer-capture, then drive the app's
  global functions (they're plain function declarations on `window`) and assert
  on the DOM. Every feature this far was smoke-tested this way before shipping.

## What this prototype is now

A faithful, single-file (`index.html`, no build step) clone of **Shopify's
Online Store 2.0 theme editor**, on a sample "NOIR" sunglasses store. It exists
to design the theme's sections/blocks/settings under Shopify's real constraints
and **export real theme files**. `SHOPIFY_SPEC.md` is the constraint contract
(8-deep block nesting, 50 blocks/section, preset-driven pickers, the 36 setting
types, etc.) — keep changes within it.

## Architecture / mental model

**Data model — instances vs types.** A section on the page is an INSTANCE with a
unique id (`data-sec`); its TYPE (the schema/renderer key) is the id for the
originals, else looked up via `secType(id)` (`sectionTypes[]` map). State is
keyed by **instance id**: `STATE.sectionBlocks[id]`, `STATE.sectionSettings[id]`,
`STATE.sectionSchemes[id]`. Registries are keyed by **type**: `SECTION_SCHEMAS`,
`SECTION_RENDERERS` / `CARD_RENDERERS`, `SECTION_WIRE`, `SECTION_BUILDERS`.
Section order lives in `SECTION_GROUPS.{header,template,footer}.sections` (arrays
of instance ids). `allSectionIds()` = all instances.

**Blocks are a nested tree, addressed by PATH.** `STATE.sectionBlocks[id]` is an
ordered array; each block may have `.blocks` (children). A block is addressed by
`[section, path]` where `path` is an array of indices (`[2]` = 3rd top-level
block, `[2,0]` = its first child). Helpers: `blockAt`, `parentListOf`,
`childListAt`, `pathKey`. Theme blocks nest up to **8** deep (`subtreeDepth`
enforces it); `moveBlockPath` handles reorder + re-parenting + cycle guard.

**Two kinds of blocks.**
- *Generic theme blocks* (`THEME_BLOCK_TYPES`: group, heading_block, text_block,
  button_block, image_block, icon_block, email_form). `group` accepts `@theme`
  children → real nesting. Rendered by the recursive `renderThemeBlock`.
- *Bespoke card blocks* (collection_card, product_card, testimonial, logo,
  gallery_image, post_card, icon_column). Rendered per-block by `CARD_RENDERERS`.
- *Header/footer bespoke blocks* (`header_logo`/`header_menu`/`header_icons`,
  `footer_brand`/`footer_menu`/`footer_bottom`). The **nav and footer are now
  built entirely from block instances** (no hardcoded chrome): `SECTION_RENDERERS.nav`
  lays header blocks out in the bar; `SECTION_RENDERERS.footer` renders column
  blocks into `.cols` + a bottom-bar block. Seeded in `buildDefaultBlocks`; their
  `SECTION_SCHEMAS.blocks` list only their own types so the picker offers the right
  set. Per-block icons live in `BLOCK_ICON` (`blockTypeIcon`); section layer icon is
  one shared glyph (`layerIcon` → `ICONS.section`, Shopify shows one section icon).

**Sections in ALL three groups are editable** (header/template/footer): add /
remove / duplicate / reorder (drag stays within a group), hide, blocks. Helpers:
`sectionGroupId(id)`, `sectionIndexInGroup(id)`, `moveSection(gid,from,to)`,
`placeNewSection(node,id,gid,index)`, `addSection(type,gid,index)`. `reorderSectionDom`
re-sequences every `[data-sec]` after the `.announce` bar in `allSectionIds()` order.

**Three section render shapes** (`renderSectionContent(id)` picks one by type):
1. *Generic* (hero, guarantee, newsletter, groupsec, valueprops): a
   `[data-blocks]` container → `renderThemeBlocks`.
2. *Two-zone HEADER_GRID* (collections, bestsellers, ugc, journal, reviews,
   press): a `[data-blocks-head]` header zone (content blocks) **plus** a
   `[data-cards]` grid (card blocks). The ONE ordered block list is split by
   type at render time; **global indices are preserved** for selection.
3. *Pure card* (none currently) / fallback.
`SECTION_RENDERERS[type]` is the full-list/generic renderer; `CARD_RENDERERS[type]`
is the per-block card renderer. `renderBlockSections()` repaints all instances.

**DOM attributes that matter:** `data-sec`=instance id; `data-bpath`="i.j.k" on
every block element (drives canvas selection + `blockEl`); `[data-blocks]`,
`[data-blocks-head]`, `[data-cards]` are the per-instance render targets (scoped
via `secEl.querySelector`, never global `#ids`, so duplicates stay isolated).

**Selection:** three mutually-exclusive targets — `selectedSection` (instance id) +
`selectedBlock` (`{section, path}`), or `selectedScheme` (a color scheme being
edited). Click on canvas (nearest `[data-bpath]` → block, else the section), a
layers-tree row, or a scheme card. `renderSectionControls()` is the one entry point:
it branches scheme → block → section and renders the right **inspector**
(`#section-float`, docked in `#inspector` on desktop / bottom sheet on mobile).
When nothing is selected the inspector collapses (canvas full-width). Selecting
auto-scrolls + expands the layers tree to the row (`revealActiveLayer`). The
inspector header has a type icon + name + a `⋯` menu (`openMoreMenu`:
sections/blocks = Duplicate/Hide/Remove, schemes = Duplicate/Remove) + close; a
bottom Remove button mirrors the menu's Remove. Section/block schemas render via
the schema engine (`renderSettingsList` / `renderSetting`, one control per Shopify
setting type); the scheme editor renders grouped `colorField`s + a preview hero.

**Live settings wiring.**
- *Section settings* → `applySectionSettings(id)` sets padding + per-type
  `SECTION_WIRE[type](elm, id)` (CSS vars / classes / inline styles).
  `secSetting(id, key)` reads the stored value or schema default. Section
  `setVal` calls `applySectionSettings`.
- *Block settings* are live "for free": block `setVal` mutates the instance's
  `settings` then `renderSectionContent(section)` repaints from them. So any
  setting a renderer reads is immediately live.

**Export** (`buildThemeFiles` → inspect modal + dependency-free zip):
`sections/<type>.liquid` + `blocks/<type>.liquid` (each with a real `{% schema %}`
serialized from the registry + presets), `templates/index.json` (instances:
unique id + type + settings + nested block tree with `block_order`),
`sections/header.json|footer.json` (section groups), `config/settings_schema.json`
(a real `color_scheme_group` from `colorRoles`) + `config/settings_data.json`
(scheme values resolved to hex + global setting values). Per-section
`color_scheme` exports as a scheme id (`scheme-1/2/3`).

## Editor UI / chrome (Shopify-faithful — built out in the UI-overhaul session)

The shell is `.app` (column): a **`.topbar`** + `.app-body` (panel | preview |
inspector). Topbar: black brand mark, a segmented **Layers ⇄ Theme-settings** mode
toggle (`.tb-mode`, drives `panel[data-mode]`, `setPanelMode`), a centre **page
switcher**, a segmented **desktop/mobile device** toggle (`.tb-dev`; mobile adds
`.preview.dev-mobile` and the store's `@container` lives on `.store-frame` so the
narrow frame triggers real mobile layout), and **Reset + Export** on the right.

**Layers panel** (`buildLayers`/`renderBlockRows`): uniform 34px rows; the *row* is
the hover/active surface (solid-blue when active) so the highlight spans full width;
chevron + lead(icon→drag-handle on hover) + name + hover actions (hide/delete) all
live inside it. Hidden state is a `hiddenLayers` set (`applyHidden` toggles
`.canvas-hidden`). Drag uses `gripStart`/`dropTarget`/`dragState`.

**Canvas hover** (`initCanvasHover`/`updateCanvasHover`): a blue name-tag chip (icon
+ name, matching the layer) + blue `+` insertion buttons on the hovered element's
edges (sections: above & below; blocks: below only). `+` → `openSectionPicker`/
`openBlockPicker` at that index. Hover is **two-way synced** with the layers panel.
Overlays are fixed-position and live **inside `.preview`** (so moving onto a `+`
doesn't fire `mouseleave`). Top bar sits above them via `z-index`.

**Add picker** (`openPicker`): master/detail modal — searchable, collapsible
categorised list (rows mirror layer-row metrics) on the left + a live preview pane
on the right; no header strip (click-outside / Esc to dismiss). Section/block lists
carry per-item icons.

**Color system v2.** Brand colors are a *detached* concept: `STATE.palette` is the
brand list, managed in a **modal** (`openBrandModal`) launched from a button atop
Theme settings (Shopify has no in-theme palette; themes only *connect* to brand).
A scheme **role value is `'#hex'` | a brand-color id | `'transparent'`** —
`swatchHex(v)` resolves all three. The reusable **`colorField`** control: raw-hex
field (swatch + value) that opens an HSV picker popover (`openColorPicker`), with a
hover "database" icon → brand-connect popover (`openBrandConnect`); when connected
it shows a chip (brand name) + `⋯` (Edit brand / Replace / Select color / Remove→white).
Schemes show as a **3-col card grid** (`renderSchemes`); selecting one edits it in
the inspector. `popover()`/`closePopover()` is the shared anchored-popover helper;
`.cmenu` is the shared context-menu style.

**Shared sizing vocabulary** (keep to these): **26px** squares (swatches, db button,
scheme preview, layer action/chevron/lead buttons), **32px** square icon-buttons
(top-bar toggles inner buttons, inspector `⋯`/`×`), **34px** control/row height,
**4px** padding around the 26px squares. Type: body/labels/inputs **12px / weight 400**;
subheadings (layer group headers, settings `set-header`, picker categories,
`subgroup-label`) **13px / 600**; pane + accordion headers **14px / 600**.

## Recipes

**Add a generic theme block:** add to `BLOCK_SCHEMAS` (Object.assign block) +
`THEME_BLOCK_TYPES` + `BLOCK_CATEGORY`; add a `case` in `renderThemeBlock`. It's
instantly addable anywhere `@theme` is accepted, selectable, and exportable.

**Add a section:** add to `SECTION_SCHEMAS` (+ `category` so it's in the Add
picker) and `TEMPLATE_SECTION_TYPES`; give it a render path (a `[data-blocks]`
shell via `SECTION_BUILDERS` + `SECTION_RENDERERS.<type> = renderThemeBlocks`, or
a `CARD_RENDERERS` + `HEADER_GRID` two-zone), seed default blocks in
`buildDefaultBlocks`, and add live wiring in `SECTION_WIRE` if it has layout
settings. Card containers need `[data-cards]`; header zones `[data-blocks-head]`.

**Wire a section setting:** add it to the schema, then read it in the section's
`SECTION_WIRE` function via `secSetting(id, key)` and set a CSS var / class /
inline style. (Grid columns use `gridCols`, applied only when the user overrides
so CSS responsive defaults still win otherwise.)

## Known gaps / future work (the user's roadmap)

- **Next up (separate session): build out the sections & blocks library** — the
  exact set of section/block types and, critically, the **settings each exposes
  when selected**, plus tightening how the global style guide connects through to
  those per-section/block controls. The editor chrome (above) is the stable
  surface this plugs into: add a type to the registries + a renderer and it's
  selectable/editable/exportable.
- The color picker is HSV square + hue + alpha (no eyedropper / recent-colors —
  intentionally skipped). Brand-color names are derived from hex (`colorName`).
- Reset only resets theme (style-guide) settings; it does **not** restore
  added/removed/duplicated sections.
- Reviews aggregate score + a couple of section chromes are intentionally simple.
- Contact form (`{% form 'contact' %}`) not built — expected to come with
  non-homepage templates/pages.
- The user is refining, across other sessions: which sections/blocks exist, the
  exact section + block settings, the style-guide (global) controls, and **how
  the global style guide connects through to section/block settings** — aiming
  at a complete spec of everything needed to build the real theme.

