---
name: sketch-design
description: Design UI in the Sketch Mac app through Sketch's built-in MCP server (run_code, get_screenshot, get_guide, get_design_assets, …). Builds screens, components and design systems with native Sketch features — Frames, Stacks, Color Variables, Text/Layer Styles, Symbols with overrides, Frame Templates, prototype links and exports — and verifies every step with screenshots. Use when the user asks to design, mock up, wireframe, lay out, restyle, or build a design system / component / screen / prototype in Sketch, or to edit or audit an open Sketch document.
---

# Sketch Design

Design directly in the user's open Sketch document, the way a careful senior product designer would: system first, real Sketch features instead of loose shapes, small verifiable steps.

For reproducing an existing page/screenshot/codebase pixel-for-pixel, Sketch's own `sketch-design-from-reference` skill (github.com/sketch-hq/agents) goes deeper; this skill is for original design work and general document edits.

## 0. Preconditions

- Sketch 2025.2.4+ (non-App-Store build), MCP server started: ⌘K → "Start MCP Server", or Settings → General → MCP Server. Default endpoint `http://localhost:31126/mcp`.
- The agent must have the `sketch` MCP server connected. If the Sketch tools aren't available, stop and tell the user how to connect it (see `references/setup.md`). Do not fall back to AppleScript, writing `.sketch` files, or guessing.
- Work in the document the user has open. Only create a new document if the user asks for one. Never touch other open documents.

## 1. Load Sketch's built-in guides first (mandatory)

Sketch ships version-matched guides through the MCP `get_guide` tool. They are the source of truth for this Sketch build and override anything below if they disagree.

1. `get_guide` `topic: "mcp"` — always, before any other Sketch tool.
2. `get_guide` `topic: "use"` — before any inspection or edit of content.
3. On demand, before the matching work: `layout` (Frames, Stacks, sizing, pins, order), `styling` (Color Variables, shared styles, corners, blurs, glass, shadows, typography), `symbols` (components, overrides), `prototyping`, `assets` (export/import), `troubleshooting` (any failed call).

Then run a version probe once per session (`sketch.version.sketch`, document ID) and keep the document ID for `get_screenshot` / `get_design_assets`.

## 2. Workflow

### Step 1 — Brief and plan (no tool calls that mutate)

Pin down, asking only if genuinely ambiguous: platform and frame size(s), screens/states needed, content (use real, specific copy — never lorem ipsum), brand inputs (colors, fonts, existing library), light/dark. Write a short plan: token list, components, screens, and for each screen its regions and the Stack model of each region (direction, gap, padding, alignment, sizing).

Common frame sizes: iPhone 393×852, Android 412×917, iPad 1024×1366, Desktop 1440×1024 (or 1440 wide, height to content).

### Step 2 — Inspect and reuse before creating

1. `get_document_info` → pages, top-level frames, layer counts.
2. `get_libraries`, then `get_design_assets` for `swatch`, `textStyle`, `layerStyle`, `symbol`, `frameTemplate`, `graphicTemplate` (use `nameContains` on big libraries).
3. `get_layer_tree_summary` on anything you'll edit.

Reuse what exists: library/document Color Variables, text styles, layer styles, symbols and templates beat new ones. Only create tokens/components that are missing, and follow the document's existing naming scheme.

### Step 3 — Tokens (the design system layer)

Create missing tokens as native Sketch assets, never as hardcoded values on layers:

- **Color Variables** (`doc.swatches`) — semantic names with `/` grouping: `Color/Background`, `Color/Surface`, `Color/Border`, `Color/Text/Primary`, `Color/Text/Secondary`, `Color/Brand/Primary`, `Color/Brand/On Primary`, plus status colors as needed.
- **Text Styles** (`doc.sharedTextStyles`) — a small type scale, e.g. `Type/Display`, `Type/Heading`, `Type/Body`, `Type/Label`, with `textSwatch` bound to a Color Variable.
- **Layer Styles** (`doc.sharedLayerStyles`) — recurring surfaces, e.g. `Surface/Card` (fill + border + shadow bound to swatches).

Apply with `fill.swatch = swatch`, `text.style.textSwatch = swatch`, `layer.sharedStyle = style`. Hex values are a last resort. Recipes: `references/recipes.md`.

### Step 4 — Components (Symbols)

Anything used twice (buttons, inputs, nav items, list rows, cards, avatars, tags) becomes a Symbol:

- Build it as a Frame **with a Stack** so instances hug content (a button grows with its label), then `sketch.SymbolMaster.fromFrame(frame)`.
- Keep Symbol Sources on the document's Symbols page (`Page.getSymbolsPage` / `createSymbolsPage`); instances live on working pages.
- Name with `/` for the Insert menu hierarchy: `Button/Primary`, `Button/Secondary`, `Input/Default`.
- Customize instances only through overrides: call `get_symbol_overrides`, then target `commonOverrideIDPrefix + '_' + property`. Never detach to edit.

### Step 5 — Screens

Build outside-in, one small `run_code` per step, screenshot after each region:

1. Canvas-level Frame for the screen, placed right of existing frames (measure `page.canvasLevelFrames`; never stack new frames at 0,0). Background fill bound to `Color/Background`.
2. Region containers (header, content column, sidebar, footer) as nested Frames with `stackLayout`.
3. Children inside each region in small sibling batches: text with shared text styles, symbol instances, simple shapes.
4. Pins on non-stack elements that must stick to edges (e.g. bottom tab bar: `verticalPins = Pin.Max`, `horizontalPins = Pin.All`).

Use Stacks for every row, column, list, nav, toolbar, form, card and grid (`wraps: true` for grids). Manual x/y only for page-level placement, overlays, decorative artwork.

Use Graphics (`Group.Graphic`) for icons and illustrations (they scale proportionally); import real SVG/PNG assets instead of redrawing logos, photos or detailed icons — see `get_guide` `assets`.

Variants of one screen (states, breakpoints, dark mode) are separate frames named consistently: `Home — Desktop`, `Home — Mobile`, `Home — Empty`.

### Step 6 — Prototype (when asked or when building a flow)

`layer.flow = { target: frame, animationType: sketch.Flow.AnimationType.slideFromRight }`, `Flow.BackTarget` for back, `frame.flowStartPoint = true` for the entry screen, `sketch.HotSpot` for click areas without a layer. Overlays, fixed elements, scroll areas, Smart Animate and Dissolve are **not** in the public API — list them for the user as manual steps (Prototype tab in the Inspector).

### Step 7 — Verify, then hand off

After every appearance-changing batch: `get_screenshot` of the changed frame, critique, fix one thing at a time, re-screenshot. Before finishing, check:

- Hierarchy reads at a glance; one primary action per view.
- Spacing on a 4/8-pt rhythm; consistent gaps in sibling Stacks; aligned edges.
- Text contrast ≥ 4.5:1 for body, ≥ 3:1 for large text and UI boundaries; touch targets ≥ 44×44 (iOS) / 48×48 (Android).
- No clipped text or shadows, no overlaps, nothing outside its frame.
- Every color/text/surface uses a Color Variable or shared style; repeated UI is symbol instances.
- Every layer has a meaningful name (`Header`, `Primary CTA`, not `Rectangle 3`).

Only then call `doc.centerOnLayer(frame)` once. Summarize: frames created (names + IDs), tokens/styles/symbols added, what the last screenshot showed, and any manual steps left (fonts to install, prototype overlays, etc.). Offer exports (`sketch.export`, see `get_guide` `assets`) or Frame Templates (`frame.isTemplate = true`) if useful.

## 3. `run_code` rules (from Sketch's guide — follow strictly)

- First line `const sketch = require('sketch')`. Public SketchAPI only — no `sketchObject`, `MS*` classes or Objective-C bridges.
- **One small action per script.** Never build a whole screen or region in one call. Re-resolve the document and layers by ID every script; nothing persists between calls.
- **No comments inside submitted scripts.** Multiline, not minified. Required `title` < 10 words.
- Report with `console.log(JSON.stringify({ ok, ...ids }))`; log the IDs the next script needs. `sketch.UI.message` is invisible to you.
- Never assign `parent.layers = [...]` (can delete layers or crash Sketch with no Undo). Reorder with `layer.index`, reparent with `layer.parent`, then `stackLayout.apply()`. Log child counts before/after structural edits; if they don't match, stop and ask the user to Undo.
- Check `Number.isFinite` on computed frame values before assigning.
- On failure: load `get_guide` `troubleshooting`, read the full error, change one thing, retry.

## 4. Verified gotchas (Sketch 2026.3)

- **Stacks pack in reverse index order.** The highest index is drawn first (left/top). Pass `layers: [...]` in reverse reading order; a layer you add to an existing stack lands at the **start** — set `child.index = 0` to append it at the end.
- **Canvas-level Frames get an automatic white background fill.** If you also pass `style.fills`, the white fill ends up on top and hides yours. Create screen frames with `background: { enabled: false }` plus your `style.fills`; when converting a canvas Frame to a Symbol, reset `style.fills` to just your fill.
- **New Stacks hug content.** For a fixed-width column set `horizontalSizing = sketch.FlexSizing.Fixed`, then the width, then `stackLayout.apply()`. Use `FlexSizing.Fill` for children that should share space (e.g. equal cards in a row).
- **Nested Frames clip by default** — set `clipsContents = false` on structural Stack wrappers, or card shadows get cut off.
- **Fonts fail silently.** Setting a `fontFamily` that isn't installed keeps the previous font. Read `style.fontFamily` back after setting it; fall back to `'system'` and tell the user which font to install. Missing weights snap to the nearest available weight (`fontWeight` is 0–12; 5 regular, 6 medium, 7 semibold, 9 bold — approximate per font).
- **`style.fontSize` / `fontWeight` read back as `null`** even though they're applied. Verify typography by frame height or screenshot, not those getters.
- **Deferred updates (2026.2+):** after changing Color Variables, shared styles, Stacks or concentric corners, call `doc.processPendingChanges()` before reading dependent values.
- Corners: `layer.style.corners.radii = 12` (or `[tl, tr, br, bl]`), `style.corners.style = sketch.Style.CornerStyle.Smooth` for iOS-like squircles, `corners.concentric = true` to follow the parent frame.
- Product → API names: Frame/Graphic = `Group.Frame`/`Group.Graphic` (canvas-level ones report `type === 'Artboard'`); Color Variable = `Swatch`; Component/Symbol Source = `SymbolMaster`; Stack = `stackLayout`.

## References

- `references/recipes.md` — tested scripts: tokens, text/layer styles, button symbol, screen frame, Stacks, card grid, instances + overrides, prototype links, export.
- `references/feature-map.md` — Sketch features and how to drive each one (API vs manual).
- `references/setup.md` — connecting Sketch MCP to Claude Code / T3 Code / Codex / Cursor.
- Sketch MCP docs: https://www.sketch.com/docs/mcp-server/
- SketchAPI (plain text for agents): https://developer.sketch.com/reference/api/llms-full.txt — recent API changes: https://developer.sketch.com/plugins/updates/llms.txt
