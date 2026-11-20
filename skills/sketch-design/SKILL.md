---
name: sketch-design
description: Design UI in the Sketch Mac app through Sketch's built-in MCP server (run_code, get_screenshot, get_guide, get_design_assets, list_comments, save_comment, …). Builds screens, components and design systems with native Sketch features — Frames, Stacks, Sections, Color Variables, Text/Layer Styles, Components with Variants and overrides, Frame Templates, prototype links and exports — verifies every step with canvas checks, and reads and answers Workspace comments. Use when the user asks to design, mock up, wireframe, lay out, restyle, or build a design system / component / variant / screen / prototype in Sketch, to organize a Sketch document into sections, to address or reply to Sketch comments, or to edit or audit an open Sketch document.
---

# Sketch Design

Design directly in the user's open Sketch document, the way a careful senior product designer would: system first, real Sketch features instead of loose shapes, small verifiable steps.

For reproducing an existing page/screenshot/codebase pixel-for-pixel, Sketch's own `sketch-design-from-reference` skill (github.com/sketch-hq/agents) goes deeper; this skill is for original design work, design-system work, comment handling and general document edits.

## 0. Preconditions

- Sketch 2026.4+ for everything below (Components with Variants, Sections, comment tools, `layer.scale`). Sketch 2025.2.4+ has a basic MCP server; on older builds skip the features the version probe doesn't expose. Not the Mac App Store build. 2026.4 requires macOS 15+, and files saved with it don't open in older Sketch.
- MCP server started: ⌘K → "Start MCP Server", or Settings → General → MCP Server. Use the URL Sketch shows (default `http://localhost:31126/mcp`).
- The agent must have the `sketch` MCP server connected. If the Sketch tools aren't available, stop and tell the user how to connect it (see `references/setup.md`). Do not fall back to AppleScript, writing `.sketch` files, or guessing.
- Work in the document the user has open (or one they name by path). Only create a new document if the user asks for one. Never touch other open documents — when several are open, resolve yours by ID: `sketch.getDocuments().find(d => d.id === '<DOC-ID>')`.

## 1. Load Sketch's built-in guides first (mandatory)

Sketch ships version-matched guides through `get_guide`. They are the source of truth for the running build and override anything below if they disagree.

1. `get_guide` `topic: "mcp"` — always, before any other Sketch tool.
2. `get_guide` `topic: "use"` — before any inspection or edit of content.
3. On demand, before the matching work: `layout` (Stacks, sizing, pins, order), `organization` (pages, Components page, Sections, placing top-level frames), `styling` (Color Variables, shared styles, corners, borders, blurs, glass, Fade, typography), `symbols` (components, Variants, overrides), `prototyping`, `assets` (export/import), `troubleshooting` (any failed call).

Then run the version probe once per session (`sketch.version.sketch`, `sketch.version.api`, document ID) and keep the document ID for `get_screenshot` / `get_design_assets` / comment tools.

When talking to the user, call a screenshot verification a **canvas check** ("A canvas check confirms the button now hugs its label"), as Sketch's guide asks.

## 2. Workflow

### Step 1 — Brief and plan (no tool calls that mutate)

Pin down, asking only if genuinely ambiguous: platform and frame size(s), screens/states needed, content (real, specific copy — never lorem ipsum), brand inputs (colors, fonts, existing library), light/dark. Write a short plan: token list, components **and their variant properties** (e.g. Button: `Style` Primary/Secondary × `Size` M/L), sections, screens, and for each screen its regions and the Stack model of each region (direction, gap, padding, alignment, sizing).

Common frame sizes: iPhone 393×852, Android 412×917, iPad 1024×1366, Desktop 1440×1024 (or 1440 wide, height to content).

### Step 2 — Inspect and reuse before creating

1. `get_document_info` → pages, top-level frames (with `section`/`sectionID`), and `sections` per page.
2. `get_libraries`, then `get_design_assets` for `swatch`, `textStyle`, `layerStyle`, `symbol`, `frameTemplate`, `graphicTemplate` (use `nameContains` on big libraries). Components with Variants are listed once as `[variant set, N variants; Property: values]`.
3. `get_layer_tree_summary` on anything you'll edit — it shows `Section`, `VariantSet`, `Variant`, `Component` and `Instance [of "Button"; variant Size: L; 3 overrides]` lines.

Reuse what exists: library/document Color Variables, text styles, layer styles, components and templates beat new ones. Only create what is missing, and follow the document's existing naming scheme.

### Step 3 — Tokens (the design system layer)

Create missing tokens as native Sketch assets, never as hardcoded values on layers:

- **Color Variables** (`doc.swatches`) — semantic names with `/` grouping: `Color/Background`, `Color/Surface`, `Color/Border`, `Color/Text/Primary`, `Color/Text/Secondary`, `Color/Brand/Primary`, `Color/Brand/On Primary`, plus status colors as needed.
- **Text Styles** (`doc.sharedTextStyles`) — a small type scale (`Type/Display`, `Type/Heading`, `Type/Body`, `Type/Label`), each with `textSwatch` bound to a Color Variable.
- **Layer Styles** (`doc.sharedLayerStyles`) — recurring surfaces, e.g. `Surface/Card` (fill + border + shadow bound to swatches).

Apply with `fill.swatch = swatch`, `text.style.textSwatch = swatch`, `layer.sharedStyle = style`. Hex values are a last resort. Recipes: `references/recipes.md`.

### Step 4 — Components and Variants

Anything used twice (buttons, inputs, nav items, list rows, cards, avatars, tags) becomes a Component (API: `SymbolMaster`):

- Build it as a Frame **with a Stack** so instances hug content, then `sketch.SymbolMaster.fromFrame(frame)`.
- Keep Component Sources on the document's Components page (`Page.getSymbolsPage(doc)` / `Page.createSymbolsPage()` — in 2026.4 it creates a page named "Components"); instances live on working pages.
- **States and sizes of one component are Variants, not separate components.** Build one source per combination, then combine them with `new sketch.Group.VariantSet({ name, parent, variantProperties, layers })` and set each source's `variantConfiguration` (recipe 5). Always pass `variantProperties` explicitly — derived names come out as "Property 1/2" — and list each property's default value first: `set.defaultVariantMaster` is the first value of every property.
- Place a variant with `set.getBestVariantMasterForConfiguration({ Style: 'Primary', Size: 'L' }).createNewInstance()`; switch an instance to another variant with `instance.symbolId = otherVariant.symbolId`. Read `set.variantProperties` before building a configuration — unknown names or values throw.
- Standalone components (no variants): name with `/` for the Insert menu hierarchy (`Icon/Search`, `Avatar/M`).
- Customize instances only through overrides: `get_symbol_overrides`, then target `commonOverrideIDPrefix + '_' + property`; for several overrides on one instance use `instance.updateOverrides(overrides => …)`. Never detach to edit.

### Step 5 — Screens and Sections

Build outside-in, one small `run_code` per step, canvas check after each region:

1. Canvas-level Frame for the screen, placed right of everything on the page — measure `page.layers` (page coordinates, includes sections), **not** `page.canvasLevelFrames` (it includes frames inside sections in section coordinates and omits the sections, so new frames overlap). Background fill bound to `Color/Background` via `style.fills`.
2. Region containers (header, content column, sidebar, footer) as nested Frames with `stackLayout` and `background: { enabled: false }`.
3. Children inside each region in small sibling batches: text with shared text styles, component instances, simple shapes.
4. Pins on non-stack elements that must stick to edges (e.g. bottom tab bar: `verticalPins = Pin.Max`, `horizontalPins = Pin.All`).

Use Stacks for every row, column, list, nav, toolbar, form, card and grid (`wraps: true` for grids). Manual x/y only for page-level placement, overlays, decorative artwork.

Group related screens into **Sections** (a flow, a feature, a breakpoint set): `new sketch.Group.Section({ name, parent: page, layers: [frames] })`, or give the section a Row Stack so added frames tile and the section grows (recipe 7). Sections can't be components, link targets or Start Points; the frames inside can.

Use Graphics (`Group.Graphic`) for icons and illustrations (they scale proportionally); import real SVG/PNG assets instead of redrawing logos, photos or detailed icons — see `get_guide` `assets`. To resize a layer the way the Scale tool does (text sizes, borders, radii, shadows included) use `layer.scale(factor, { origin: sketch.ScaleOrigin.Center })`; assigning `frame` doesn't scale styles.

Variants of one screen (states, breakpoints, dark mode) are separate frames named consistently: `Home — Desktop`, `Home — Mobile`, `Home — Empty`, usually in one section.

### Step 6 — Prototype (when asked or when building a flow)

`layer.flow = { target: frame, animationType: sketch.Flow.AnimationType.slideFromRight }`, `Flow.BackTarget` for back, `frame.flowStartPoint = true` for the entry screen, `sketch.HotSpot` for click areas without a layer. Targets and Start Points must be Frames (Sections and Components with Variants throw). Overlays, fixed elements, scroll areas, Smart Animate, Dissolve and hover/press interactions are **not** in the public API — list them for the user as manual steps (Prototype tab in the Inspector).

### Step 7 — Verify, then hand off

After every appearance-changing batch: `get_screenshot` of the changed frame, critique, fix one thing at a time, re-check. Before finishing, check:

- Hierarchy reads at a glance; one primary action per view.
- Spacing on a 4/8-pt rhythm; consistent gaps in sibling Stacks; aligned edges.
- Text contrast ≥ 4.5:1 for body, ≥ 3:1 for large text and UI boundaries; touch targets ≥ 44×44 (iOS) / 48×48 (Android).
- No clipped or overflowing text, no clipped shadows, no overlaps, nothing outside its frame.
- Every color/text/surface uses a Color Variable or shared style; repeated UI is component instances; states are Variants.
- Every layer has a meaningful name (`Header`, `Primary CTA`, not `Rectangle 3`).

Only then call `doc.centerOnLayer(frame)` once. Summarize: frames/sections created (names + IDs), tokens/styles/components/variants added, what the last canvas check showed, and manual steps left (fonts to install, prototype overlays, etc.). Offer exports (`sketch.export`, see `get_guide` `assets`) or Frame Templates (`frame.isTemplate = true`) if useful.

## 3. Auditing an existing document

When asked to audit, review or clean up a file, inspect before changing anything:

1. `get_document_info`, then `get_design_assets` for the document's tokens, styles and components.
2. Run recipe 16 per page (structure) and per screen frame (tokens). Audit Component sources on the Components page too, since instances are skipped.
3. Canvas check each frame for what scripts can't see: spacing rhythm, alignment, clipped text, visual hierarchy.
4. Report findings grouped by severity (accessibility, broken system links, structure, naming) with layer names and IDs, and propose fixes. Change nothing until the user picks what to fix; then fix in small verified steps.

## 4. Comments (Workspace documents only)

Use when the user asks to read, address, or answer feedback. Comments exist only on documents saved to a Sketch Workspace with the user signed in; otherwise the tools return "Comments are only available in documents saved to a Workspace." / "Not signed in to a Sketch account." — tell the user and stop.

1. Find threads: `list_comments` (`filter`: `open` default, `unread`, `all`; `subjectID` to scope to a page, frame or section). An annotation is a thread pinned to a page/frame/section; call again with `annotationID` to read its comments (newest first). Follow `nextCursor` for more pages.
2. Across documents: `list_notifications` lists comments, mentions and invites; each gives `document.documentID` (if open) and an `annotationID`. If the document isn't open, ask the user to open `document.url` (or `open -a Sketch "<url>"`), then call again.
3. Address design feedback with the normal workflow (Steps 2–7), then verify with a canvas check.
4. Reply / resolve with `save_comment` (`annotationID` + `body` to reply; `annotationID` + `resolved: true` [+ `body`] to resolve). Start a new thread with `subjectID` + `body` (+ `x`, `y`). Edit your own comment with `commentID` + `body`. Mentions: `@[Name](user:ID)`.
5. Comments post as the signed-in user, are visible to everyone with access and may notify people: **show the user the exact text and get confirmation before posting, resolving, or deleting.** `delete_comment` can't be undone — use it only when asked.

## 5. `run_code` rules (from Sketch's guide — follow strictly)

- First line `const sketch = require('sketch')`. Public SketchAPI only — no `sketchObject`, `MS*` classes or Objective-C bridges.
- **One small action per script.** Never build a whole screen or region in one call. Re-resolve the document and layers by ID every script; nothing persists between calls.
- **No comments inside submitted scripts.** Multiline, not minified. Required `title` < 10 words.
- Report with `console.log(JSON.stringify({ ok, ...ids }))`; log the IDs the next script needs. `sketch.UI.message` is invisible to you.
- Never assign `parent.layers = [...]` (can delete layers or crash Sketch with no Undo). Reorder with `layer.index`, reparent with `layer.parent`, then `stackLayout.apply()`. Log child counts before/after structural edits; if they don't match, stop and ask the user to Undo.
- Check `Number.isFinite` on computed frame values before assigning.
- On failure: load `get_guide` `troubleshooting`, read the full error, change one thing, retry.
- `get_screenshot` refuses captures over 100 megapixels — screenshot a frame, not a wide page. Large PNGs come back as a file path; read that file.

## 6. Verified gotchas (Sketch 2026.4)

- **Stacks pack in reverse index order.** The highest index is drawn first (left/top). Pass `layers: [...]` in reverse reading order; a layer you add to an existing stack lands at the **start** — set `child.index = 0` to append it at the end. Same for variants inside a Component with a Stack.
- **Frame backgrounds:** a canvas Frame created with `style.fills` gets exactly that fill (the 2026.3 white-fill-on-top bug is fixed). Without `style.fills` it gets a default white background. Structural wrappers: pass `background: { enabled: false }`. `fromFrame` keeps the frame's fill.
- **New Stacks hug content.** For a fixed-width column set `horizontalSizing = sketch.FlexSizing.Fixed`, then the width, then `stackLayout.apply()`.
- **Long text in a Stack doesn't wrap by itself** — even with `alignItems: Stretch` it overflows the column. Set the text's `horizontalSizing = sketch.FlexSizing.Fill`, then `apply()`.
- **A hugging row created with `alignItems: Stretch` doesn't grow** when its cards' text wraps — the text runs into the card padding. Build the row with `Start`, apply the cards' and the row's stacks, then switch to `Stretch` and apply again (recipe 10).
- **Stretched instances don't re-center their content.** A component instance stretched by its parent's `alignItems: Stretch` keeps its label at the left. Set `instance.horizontalSizing = sketch.FlexSizing.Fill` so the source's Stack re-runs at the new width.
- **Top-level frames have `Relative` sizing.** Before giving a Section (or other container) a `stackLayout`, set each child frame's `horizontalSizing`/`verticalSizing` to `Fixed` and restore its `frame`, or it collapses to 0.1×0.1.
- **Sections and variant sets have a fixed size** and don't grow when you add a frame — give them a Stack (then `apply()`) or resize them yourself (`get_guide` `organization`). A variant set also reports `isSection`: test `isVariantSet` first. Never add a plain frame to a variant set.
- **Nested Frames clip by default** — set `clipsContents = false` on structural Stack wrappers, or card shadows get cut off.
- **Fonts fail silently.** Setting a `fontFamily` that isn't installed keeps the previous font (Helvetica on a fresh text). Read `style.fontFamily` back after setting it; fall back to `'system'` and tell the user which font to install. `fontWeight` is 0–12 (5 regular, 6 medium, 7 semibold, 9 bold — approximate per font).
- **`style.fontSize` / `fontWeight` read back as `null`** on text layers and shared text styles, though they're applied. Verify typography by frame height (`adjustToFit()`) or canvas check.
- **Deferred updates (2026.2+):** after changing Color Variables, shared styles, Stacks, variant properties or concentric corners, call `doc.processPendingChanges()` before reading dependent values.
- Corners: `layer.style.corners.radii = 12` (or `[tl, tr, br, bl]`), `style.corners.style = sketch.Style.CornerStyle.Smooth` for iOS-like squircles, `corners.concentric = true` to follow the parent frame (don't also set radii).
- Product → API names: Frame/Graphic = `Group.Frame`/`Group.Graphic` (canvas-level ones report `type === 'Artboard'`); Color Variable = `Swatch`; Component / Component Source (formerly Symbol Source) = `SymbolMaster`, instance = `SymbolInstance`; Component with Variants = `Group.VariantSet`, each Variant a `SymbolMaster` with `isVariant`; Section = `Group.Section`; Stack = `stackLayout`; Components page = `Page.getSymbolsPage`.

## References

- `references/recipes.md` — tested scripts: tokens, text/layer styles, button component, Components with Variants, screen placement, Stacks, card grid, instances + batched overrides, Sections, scale/transform, prototype links, export, comments, document audit.
- `references/feature-map.md` — Sketch features and how to drive each one (API vs manual).
- `references/setup.md` — connecting Sketch MCP to Claude Code / T3 Code / Codex / Cursor, the tool list, troubleshooting.
- Sketch MCP docs: https://www.sketch.com/docs/mcp-server/
- SketchAPI (plain text for agents): https://developer.sketch.com/reference/api/llms-full.txt — recent API changes: https://developer.sketch.com/plugins/updates/llms.txt
