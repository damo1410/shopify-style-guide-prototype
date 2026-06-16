# Shopify editor replica — build contract

This prototype is a **strict replica of Shopify's Online Store 2.0 theme editor**
(modern *theme blocks* architecture). Its job is to impose Shopify's real limits
on our design decisions so that sections, blocks, and settings we author here port
cleanly to a real theme. Every constraint below is enforced in `index.html` and
mirrored by the export (`sections/*.liquid`, `blocks/*.liquid`, `templates/*.json`,
`config/settings_schema.json`).

Sources: Shopify dev docs + `Shopify/theme-liquid-docs/schemas/theme/setting.json`
(the validator schema). Verified 2026-06.

## Hard limits (enforced)

| Rule | Value |
|---|---|
| Theme-block nesting depth | **8 levels** (section level not counted) |
| Legacy section-block nesting | **none** (single level) |
| Blocks per section (`max_blocks`) | **50** ceiling; may set lower; static blocks excluded |
| Sections per JSON template | **25** |
| JSON templates per theme | **1,000** |
| Section `limit` (times addable) | **1 or 2** only (default = unlimited); `@app` blocks can't set it |
| `color_palette` per theme | **1**, in `settings_schema.json`; 2–20 colors |
| List pickers `limit` (`*_list`) | default = max = **50** |

## Add picker (Add section / Add block)

- A file is addable in the editor **only if it defines `presets`**. No presets ⇒ it
  can only be placed manually in JSON and can't be added/removed in the editor.
- Each **preset** = one picker entry; one file may expose several presets.
- Preset fields: `name` (title shown), `category` (optional — the **single** level of
  grouping in the picker), `settings`, `blocks`.
- Picker grouping is **one level only** (category heading → preset entries). No
  sub-folders. `category` values are developer-supplied free text (no fixed list).
- A section/block may **recommend** specific child block types by listing them
  alongside `@theme` in its `blocks` array.

## Settings hierarchy (the only allowed grouping)

- **Section / block settings**: a **flat array**. Only `header` (+ optional `paragraph`)
  acts as a grouping label. No nested groups.
- **Theme settings** (`config/settings_schema.json`): `theme_info` object, then
  **category** objects `{ name, settings }`. Inside a category, again only
  `header`/`paragraph` group things. Max depth: category → header → settings.
- `select` options may carry an optional `group` (dropdown option headings).

## Setting types (36) — required / notable optional fields

`id`, `label` required on input types; `info` optional everywhere; `default` optional
unless noted. `visible_if` allowed on most inputs (NOT on resource pickers or inside
`color_scheme_group`).

**Basic**: `text`, `textarea`, `richtext`, `inline_richtext`, `number`, `checkbox`,
`html`, `liquid`, `url`, `image_picker`, `video`, `text_alignment` (`left|center|right`).
- `range` — **requires** `default`, `min`, `max`; optional `step` (default 1, multipleOf 0.1), `unit`; (max-min)/step ≤ 101 steps.
- `radio` / `select` — **require** `options:[{value,label}]`; `select` option may add `group`.
- `video_url` — **requires** `accept:["youtube"|"vimeo"]`.

**Color**: `color` (opt `alpha`), `color_background`, `color_scheme`,
`color_scheme_group` (no `label`; `definition` of `header|color|color_background`; `role`
map), `color_palette` (theme-only), `font_picker` (**requires** `default` font handle).

**Resource pickers** (no `visible_if`, no `placeholder`): `collection`, `product`,
`blog`, `page`, `article`, `link_list`; lists `collection_list`, `product_list`,
`article_list` (opt `limit`≤50); `metaobject`, `metaobject_list` (**require** `metaobject_type`).

**Sidebar / presentational** (no `id`, hold no value): `header` (`content`, opt `info`),
`paragraph` (`content`).

## Theme tree / files

- `templates/*.json` — `{ sections: { id: {type, settings, blocks} }, order: [ids] }`.
- `sections/*.json` (section groups, e.g. `header.json`, `footer.json`) — group of sections
  pinned to header/footer areas.
- `sections/*.liquid` + `{% schema %}` — `settings`, `blocks` (`@theme`/`@app`/type),
  `max_blocks`, `presets`, `default`, `enabled_on`/`disabled_on` (one or the other),
  `limit`, `tag`, `class`.
- `blocks/*.liquid` + `{% schema %}` — reusable theme block; may nest child `blocks`.
</content>
</invoke>
