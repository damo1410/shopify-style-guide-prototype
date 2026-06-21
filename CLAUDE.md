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

## Section/block library (Notion) — source of truth for the build-out

Curated in Notion: page **Shopify Theme Library** (`35d0334212f180a8a61fd2eb3cb75fdd`) with 3
inline DBs — **Categories** (`29090c8c-ba01-4a9d-a333-70215b621959`, 44 research/purpose rows,
**deferred/untouched**), **Sections** (`37203342-12f1-8035-b3c0-000ba5251299`, 13 rows),
**Blocks** (`ca403342-12f1-839e-9709-0799f33e6467`, 19 rows). Synced to this prototype via each
type's **`Type key`** (= the registry key / export filename — the Notion↔index.html join).

**Model (matches Shopify + the registries):**
- `Type key` = `snake_case(display name)`; prefix only on collisions (`header_logo`,
  `footer_menu`). Exports as `sections/<key>.liquid` / `blocks/<key>.liquid`.
- `Section Category` (8) / `Block category` (6) = Shopify's preset **`category`** (the Add-picker
  grouping) — NOT the 44 Categories DB.
- `@theme` / `@app` are **flags** (`Accepts @theme` / `Accepts @app`), not enumerations. The
  `Recommended blocks` / `Recommended child blocks` relations = **named types only** (the
  shortlist shown first). Any block is addable to any container; relations are recommendations
  only. Only **`group`** is a container (accepts `@theme`); `@theme` ⇒ `THEME_BLOCK_TYPES` =
  group, heading, text, button, image, icon, email_signup_form.
- **Sections props:** Name, Type key, Section Category, Accepts @theme/@app, Recommended blocks
  (→Blocks). *(No link to Categories DB — that relation was removed.)* **Blocks props:** Name,
  Type key, Block category, Container?, Accepts @theme/@app, Recommended child blocks ↔
  Recommended in blocks (self), Recommended in sections (back-ref).

**Sections (13)** — `key` (category; @theme; recommended blocks): `header` (Header & Footer; —;
header_logo/menu/icons) · `image_banner` (Banners & Hero; ✓) · `multicolumn` (Storytelling &
Content; ✓) · `collection_list` (Products; ✓; collection_card) · `featured_collection` (Products;
✓; product_card) · `image_with_text` (Storytelling & Content; ✓) · `testimonials` (Social Proof;
✓; testimonial) · `logo_list` (Social Proof; ✓; logo) · `image_gallery` (Media & Gallery; ✓;
gallery_image) · `blog_posts` (Storytelling & Content; ✓; post_card) · `email_signup` (Promotions
& Capture; ✓) · `footer` (Header & Footer; —; footer_brand/menu/bottom) · `group` (Storytelling &
Content; ✓). *Categories (8): Banners & Hero, Products, Storytelling & Content, Social Proof,
Media & Gallery, Promotions & Capture, Trust & Support, Header & Footer.*

**Blocks (19)** — `key` (category): `group` (Layout — container,@theme) · `heading` (Text) ·
`text` (Text) · `button` (Actions) · `image` (Media) · `icon` (Media) · `email_signup_form`
(Social Proof & Forms) · `collection_card` (Commerce) · `product_card` (Commerce) · `testimonial`
(Social Proof & Forms) · `logo` (Social Proof & Forms) · `gallery_image` (Media) · `post_card`
(Media) · `header_logo` (Media) · `header_menu` (Actions) · `header_icons` (Actions) ·
`footer_brand` (Media) · `footer_menu` (Actions) · `footer_bottom` (Text). *Categories (6): Text,
Media, Actions, Layout, Commerce, Social Proof & Forms.* (Header/footer keys keep their prefix;
their index.html display names are short — "Logo"/"Menu" — so the `Type key` is the bridge.)

**Notion ops gotchas:** creating/moving pages needs the connector's content-creation scope
(schema-only edits worked before content did — a "MCP tool call requires approval" means it's
off). **No hard page delete via MCP** — `notion-move-pages` a row to the `workspace` to remove it
from a DB (then delete in the Notion UI). Orphans `icon_column`/`checklist_item` were removed this
way + deleted from `index.html`.

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
(`#section-float`, docked in `#inspector` on desktop / bottom sheet on mobile — see
**Mobile** below). When nothing is selected the inspector collapses (canvas full-width).
Selecting auto-scrolls (directional — `afterSelect(origin)`: from the layers tree →
center the preview; from the canvas → center the layers row via `revealActiveLayer`).
The inspector header has a type icon + name + a `⋯` menu (shared `sectionMoreEntries` /
`blockMoreEntries` / `schemeMoreEntries`: Duplicate / Hide / Add before / Add after /
Remove; schemes = Duplicate / Remove) + close; a bottom Remove button mirrors it. Section/block schemas render via
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

**Add picker** (`openPicker`): on **desktop**, a master/detail modal — searchable,
collapsible categorised list (rows mirror layer-row metrics) on the left + a live
preview pane on the right; no header strip (click-outside / Esc to dismiss). On
**mobile** it branches to `renderPickerInSheet` (list-only sheet modal — see Mobile).
Section/block lists carry per-item icons.

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

**Theme setups (color schemes ⇄ color palette).** The prototype ships **two
fully-independent theme "setups"**, switched from the **black brand mark** (top-left;
`openSetupSwitcher` → a `popover()` menu). Each setup is a complete editor *document*:
`serializeDoc()` snapshots `clone(STATE)` + `SECTION_GROUPS` + `sectionTypes` +
`hiddenLayers` + `#store` innerHTML; `restoreDoc()` restores them (the const globals are
restored **in place**; selection is delegated off `#store`, so swapping its innerHTML keeps
listeners). `switchSetup(id)` serializes the outgoing setup, restores the target. Registry =
`THEME_SETUPS` (seeded by `seedSetups()` after boot); `activeSetupId`. This is an **experiment
harness** — add more named setups freely. The two seeded setups differ only by
**`STATE.colorMode`** (`'schemes'` | `'palette'`); the palette setup seeds its per-section
overrides from the schemes setup so it starts visually identical.
- **`'schemes'`** (classic): `STATE.schemes` (role→palette maps) assigned per section via
  `STATE.sectionSchemes` + the `color_scheme` section setting; brand colors in the detached
  `openBrandModal`. Exports `color_scheme_group` + per-section `color_scheme`.
- **`'palette'`** (Horizon-style, matches the real Horizon 4.0 UX): **Theme settings → Colors**
  is a **flat swatch grid** of `STATE.palette` (`renderPaletteGroup`: click a tile to edit,
  `+` to add, right-click to remove — no names list, no role list). `STATE.paletteDefaults` is a
  **hidden baseline** role→value map that paints `#store` so unstyled content looks right (not
  surfaced in the panel). Color is set **granularly**: a **section** exposes a single
  **Background color** (`appendSectionColors` → `STATE.sectionColors[id].background`,
  `applySectionColors` sets `--c-bg`/`--c-surface`); **text/background colors live on the text
  ELEMENT** — `appendBlockColors` adds **Text color** (on `TEXT_COLOR_BLOCKS`) + a **Background**
  toggle/color to each block's own `settings` (`text_color`/`bg_on`/`bg_color`), read inline by
  `renderThemeBlock`. The shared nullable control is **`paletteColorControl`** (null = "Default"
  = inherit; else palette id | hex). Exports a real `color_palette` setting, a per-section
  `background` `color_background`, and per-block `text_color`/`bg_color` settings
  (`sectionBgSetting`/`blockPaletteSettings`; palette refs via `exportColorVal`).
  - **Contrast auto-flip:** a section whose Background resolves to a *dark* color flips its Default
    text/icons/border to light (`isDarkColor` / `CONTRAST_FLIP_ROLES` in `applySectionColors`);
    block `text_color` overrides are inline on the element so they still win. This is what "flicks
    over" when you pick a dark color.
  - **Color picker** (`openColorPicker(anchor, value, onChange, opts)`): no alpha; has an eyedropper.
    Setting context passes `{palette:true, allowNone:true, onPick, onNone}` → an in-picker palette
    swatch row + a None/Default swatch. Editing a palette swatch passes `{onDelete}` → a Delete
    button (no right-click delete). The palette panel is a flush swatch **grid** (30px tiles).
The color-model branches: **paint** (`applySectionSchemes` → `applySectionColors`; `renderThemeBlock`
reads block colors), **Colors panel** (`buildPanel`: `renderSchemesGroup` vs `renderPaletteGroup`) +
the section/block inspectors, and **export** (`shopifySchema` extra settings / `settingsSchemaJson` /
`sectionInstanceJson` / `settingsDataJson`).
Schemes-mode code is untouched and fully intact. (Switcher is desktop chrome; mobile hides the
top bar, so it's desktop-only for now.)

**Shared sizing vocabulary** (keep to these): **26px** squares (swatches, db button,
scheme preview, layer action/chevron/lead buttons), **32px** square icon-buttons
(top-bar toggles inner buttons, inspector `⋯`/`×`), **34px** control/row height,
**4px** padding around the 26px squares. Type: body/labels/inputs **12px / weight 400**;
subheadings (layer group headers, settings `set-header`, picker categories,
`subgroup-label`) **13px / 600**; pane + accordion headers **14px / 600**.

## Mobile (bottom sheet — built out in the mobile-overhaul sessions)

Everything mobile is gated on **`@media (max-width: 860px)`** + the JS mirror
**`mqMobile`** (`matchMedia('(max-width:860px)')`). On mobile the `.topbar` and the
right `.inspector` are hidden; the `.preview` goes full-screen; and the left `.panel`
becomes a **draggable bottom sheet** over it. Aim: *everything* on mobile uses the
sheet — same draggable/collapsible chrome for the panel, selection settings, the Add
picker, and Brand colors.

**Sheet controller** (the IIFE at the very bottom of the script). Height is a CSS var
`--sheet-h`; three snaps — **peek** (`handle + header` height), **half** (`50vh`),
**full** (`95vh`, leaving a 5vh strip of preview). `window.__sheetSnap(name)` snaps
from elsewhere. Drag surface is the **whole `.panel-header`** (incl. its buttons):
it *arms* on pointerdown and only *commits* to a drag past a ~6px threshold, so a tap
still actuates a header button while a swipe moves the sheet; a committed drag calls
the shared **`suppressNextClick()`** (a one-shot capture-phase click swallow, also
used by touch reorder) so the button doesn't also fire, and `closePopover()` so any
open menu dismisses. `.panel-header *` is `touch-action:none` so the gesture isn't
hijacked when starting on a button.

**Three swappable header views** inside `.panel-header`, toggled by body classes:
`.ph-global` (default — Layers⇄Theme segmented toggle + page switcher + Export/Reset,
all 36px), `.ph-section` (`body.sec-selected` — type icon + name + `⋯` + close, filled
by **`syncMobileHeader`**), `.ph-modal` (`body.sheet-modal-open` — title + close). The
Layers⇄Theme toggle reuses `.tb-mode`/`setPanelMode`; `data-mode` drives which of
`#layers`/`#groups` shows, exactly like desktop (no left-rail tabs on mobile).

**Selection on mobile** — the `#section-float` settings card renders **into the sheet**
(`.in-sheet`, after the header); its own `.sf-head` is hidden because the panel header
carries icon/name/`⋯`/close instead. Gotcha: an empty/hidden card must collapse
(`.section-float.in-sheet[hidden]{display:none}`) or it eats half the sheet. The `⋯`
opens the shared more-menu helpers **`sectionMoreEntries` / `blockMoreEntries` /
`schemeMoreEntries`** (Duplicate / Hide / **Add before** / **Add after** / Remove) —
the *same* arrays power the desktop inspector `⋯`, the mobile header `⋯`, and the
per-row persistent `⋯` (**`makeMoreBtn`**, mobile-only; desktop keeps hover hide/delete).

**Sheet modal** — generic full-height overlay-in-the-sheet: `openSheetModal(title, build)`
/ `closeSheetModal()` render `build(host)` into `#sheet-modal` with a consistent
`.ph-modal` header. The **Add picker** (`openPicker` branches on `mqMobile` →
`renderPickerInSheet`: list-only, no preview, tap-to-add, opens at full) and **Brand
colors** (`openBrandModal` → `buildBrandBody`) both use it; desktop keeps its own
master/detail modal + `.bm-ov` overlay.

**Popovers** (`popover()`) dismiss on outside **`pointerdown`** (capture, so touch
works) and when a sheet drag starts.

**Touch reorder of layers** — `attachTouchReorder(row, payload)` uses **touch events**
(not pointer events: `pointermove.preventDefault` can't stop touch scroll). Long-press
~300ms arms; once armed, the non-passive `touchmove` calls `preventDefault()` to hold
the list still and the row follows the finger; drop reuses the **same `__accepts`/`__drop`
closures** `dropTarget` stores on every row (so semantics match desktop HTML5 DnD).
Guard skips `.lyr-tw`/`.lyr-act` only — **not all buttons** (`.lyr-main`, the row name,
*is* a button).

**Selection auto-scroll** is directional, via **`afterSelect(origin)`** with `origin`
`'layers'`/`'canvas'`/undefined, threaded through `selectSection(key, origin)` /
`selectBlock(section, path, origin)`. Mobile (either origin): snap **half** + center the
selection in the preview area above the sheet, **once** (`scrollPreviewToSelection`,
target = `innerHeight*0.5`; manual resizing never re-scrolls). Desktop: `'layers'` →
center the preview; `'canvas'` → center the layers row (`revealActiveLayer`, centered in
the list). Canvas hover chips (`.hv-tag`/`.hv-add`) are hidden on mobile.

**Testing mobile** — the jsdom harness (`/tmp/smoke.js`) forces `matchMedia` to
mobile/desktop and drives the global `window.*` functions; selection/scroll math is
exercised by calling them directly (jsdom has no real touch engine, so the long-press
*gesture* itself still needs a real-device check). Note: in web sessions the sandbox
blocks `github.io`, so verify deploys via the GitHub Actions run status, not by fetching
the live URL.

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

- **Library inventory + structure is DONE** (see **Section/block library (Notion)**
  above): 13 sections + 19 blocks mirrored into Notion, type keys name-matched to the
  registries, categories + container/@theme model in place. **Next up: per-type
  SETTINGS** — the exact settings each section/block exposes when selected (Shopify's 36
  setting types; `SHOPIFY_SPEC.md`), and tightening how the global style guide connects
  through to those per-section/block controls. Then: a `Variant strategy` field on
  Sections (same skeleton+data → presets of one file; different skeleton/data/template →
  separate `.liquid`), the Categories-DB audit + picker-`category` mapping, and growing
  the type set. The editor chrome is the stable surface this plugs into: add a type to
  the registries + a renderer and it's selectable/editable/exportable.
- The color picker is HSV square + hue + alpha (no eyedropper / recent-colors —
  intentionally skipped). Brand-color names are derived from hex (`colorName`).
- Mobile editor is built out (see **Mobile**). Open tuning items: the touch-reorder
  long-press feel (300ms / scroll-vs-drag handoff) wasn't verifiable here (no device);
  the page switcher is single-page (`PAGES` has only `index`).
- Reset only resets theme (style-guide) settings; it does **not** restore
  added/removed/duplicated sections.
- Reviews aggregate score + a couple of section chromes are intentionally simple.
- Contact form (`{% form 'contact' %}`) not built — expected to come with
  non-homepage templates/pages.
- The user is refining, across other sessions: which sections/blocks exist, the
  exact section + block settings, the style-guide (global) controls, and **how
  the global style guide connects through to section/block settings** — aiming
  at a complete spec of everything needed to build the real theme.

