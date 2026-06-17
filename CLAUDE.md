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

**Selection:** `selectedSection` (instance id) + `selectedBlock` (`{section, path}`
or null). Click on canvas (nearest `[data-bpath]` → block, else the section) OR a
layers-tree row. The right **inspector** (`#section-float`, docked in `#inspector`
on desktop / bottom sheet on mobile) renders the selection's schema via the
schema engine (`renderSettingsList` / `renderSetting`, one control per Shopify
setting type).

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

- Reset only resets theme (style-guide) settings; it does **not** restore
  added/removed/duplicated sections.
- Reviews aggregate score + a couple of section chromes are intentionally simple.
- Contact form (`{% form 'contact' %}`) not built — expected to come with
  non-homepage templates/pages.
- The user is refining, across other sessions: which sections/blocks exist, the
  exact section + block settings, the style-guide (global) controls, and **how
  the global style guide connects through to section/block settings** — aiming
  at a complete spec of everything needed to build the real theme.

