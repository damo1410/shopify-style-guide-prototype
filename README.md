# Shopify Style Guide Prototype

A self-contained, phone-editable sandbox for testing the **global theme-settings**
of a premium Shopify theme. Left panel = simulated Shopify global settings.
Right side = a live sample store that updates in real time.

**Live demo:** https://damo1410.github.io/shopify-style-guide-prototype/

## Architecture: TOKENS → ROLES → PREVIEW

One-way flow, the whole point of the prototype:

1. **TOKENS** — raw global values set once (palette swatches, fonts, base size +
   ratio, named radius/spacing/stroke scales). Palette swatches are the **only**
   place a hex value is ever typed; each swatch auto-generates an 11-step shade
   ramp (`50…950`).
2. **ROLES** — semantic names (`heading color`, `surface`, `button fill`…),
   implemented as CSS custom properties on the `.store` element (e.g.
   `--c-heading`, `--radius`). A **color scheme** is just a mapping of *role →
   palette token* such as `neutral.950` (dropdowns only, never free hex — so a
   scheme can never reference an undefined color). Button colors auto-derive from
   the palette and contrast with each scheme's background.
3. **PREVIEW** — every store element binds to a role var (`var(--c-heading)`),
   never to a raw value. Change one global token and it ripples everywhere.

## How the control → preview wiring works

Everything lives in the `STATE` object in `index.html`. Each control mutates
`STATE`, then calls **`applyRoles()`**, which resolves tokens through the active
scheme and writes the role custom properties onto `<html>`. The preview CSS reads
only those role vars — that's the entire loop.

## Editing

Single `index.html`, inline CSS + JS, no build step, no frameworks. Start in the
`STATE` object (defaults) and the `render*()` functions (controls). This version
covers global settings only — local per-section overrides come later.
