# Sketch features: what to use and how to drive each one

Columns: the Sketch feature (as the docs name it), the API for it, and whether an agent can set it through `run_code`. "Manual" means it is only available in the Sketch UI, so give the user the exact steps. Never reach for private APIs.

## Containers and layout

| Feature | When to use | API | Agent? |
|---|---|---|---|
| Frame | Screens, sections, any layout container | `new sketch.Group.Frame({...})`; `group.convertToFrame()` | Yes |
| Graphic | Icons and illustrations (they scale proportionally; their children have no pins) | `new sketch.Group.Graphic({...})`; `convertToGraphic()` | Yes |
| Stack (stack layout) | Rows, columns, lists, navs, forms, buttons, cards, grids (`wraps`) | `group.stackLayout = { direction, gap, padding, alignItems, justifyContent, wraps, crossAxisGap, stackingOrder, bordersAffectLayout }`, then `stackLayout.apply()` | Yes |
| Sizing (Fixed / Fit / Fill / Relative) | Responsive behavior inside Frames and Stacks | `layer.horizontalSizing` / `verticalSizing` = `sketch.FlexSizing.*`; `percentageSize`, `minSize`, `maxSize` | Yes |
| Pins | Freeform children of a Frame that must stick to edges (tab bars, FABs) | `layer.horizontalPins` / `verticalPins` = `sketch.Pin.Min / Max / All / None` (bitmask) | Yes |
| Ignore stack / keep space when hidden | Badges and overlays inside a Stack | `ignoresStackLayout`, `preservesSpaceInStackLayoutWhenHidden` | Yes |
| Clip content | Turn off on structural wrappers so shadows aren't cut | `group.clipsContents` | Yes |
| Smart Layout | Legacy. Use Stacks instead | `smartLayout` | Avoid |
| Layout grids and guides | Visual column guides | Not in the public API | Manual (View → Canvas → Layout Settings) |

## Design system

| Feature | API | Agent? |
|---|---|---|
| Color Variables | `doc.swatches.push({ name, color })`; on layers: `fill.swatch`, `border.swatch`, `shadow.swatch`, `text.style.textSwatch`, `style.tint = { swatch }` | Yes |
| Text Styles | `doc.sharedTextStyles.push({ name, style })`; `text.sharedStyle = style` | Yes |
| Layer Styles | `doc.sharedLayerStyles.push({ name, style })`; `layer.sharedStyle = style` | Yes |
| Symbols (components) | `sketch.SymbolMaster.fromFrame(frame)`, `master.createNewInstance()`, `doc.getSymbols()` | Yes |
| Overrides | `get_symbol_overrides`, then `instance.overrides.find(o => o.id === prefix + '_' + property).value = …` | Yes |
| Nested symbols | Nest instances inside a master; swap them with the `symbolID` override | Yes |
| Frame / Graphic Templates | `frame.isTemplate = true`; `doc.frameTemplates`; library templates come from `get_design_assets` `frameTemplate`, then `.import()` | Yes |
| Libraries | Read with `get_libraries` / `get_design_assets`; import with `ref.import()` (assign it in the same statement) | Use: yes. Creating or sharing libraries: manual |
| Document templates (Workspace) | | Manual (File → Document Settings → Template) |

## Styling

| Feature | API | Notes |
|---|---|---|
| Fills: color, gradient, pattern/image | `style.fills = [{ fillType: sketch.Style.FillType.Color \| Gradient \| Pattern, … }]` | `fillType` values are `Color`, `Gradient`, `Pattern` |
| Borders | `style.borders = [{ swatch \| color, thickness, position }]`, `borderOptions` (dashes, caps) | Per-side thickness is supported on rectangles and Frames |
| Corners | `style.corners = { radii, style: sketch.Style.CornerStyle.Rounded \| Smooth \| Angled \| InsideSquare \| InsideArc, smoothing, concentric }` | Use Smooth for iOS-style squircles. `concentric` follows the parent frame's corners |
| Shadows | `style.shadows` / `style.innerShadows = [{ color, x, y, blur, spread }]` | Use a subtle alpha, e.g. `#0F172A14` |
| Blurs and effects | `style.blurs = [{ blurType: sketch.Style.BlurType.Gaussian \| Motion \| Zoom \| Background \| Glass, radius, … }]`, progressive blur | Glass has `brightness`, `distortion`, `depth`, `chromaticAberration`. See `get_guide` `styling` |
| Tints | `group.style.tint = { swatch }` | Recolors a whole Graphic or icon |
| Masks | See `get_guide` `styling` → Masks | |
| Opacity and blend modes | `style.opacity`, `style.blendingMode` | |

## Content

| Feature | API | Notes |
|---|---|---|
| Text | `new sketch.Text({ text, name })`, `fixedWidth`, `style.alignment`, `adjustToFit()` | Prefer shared text styles. Avoid setting `lineHeight` / `kerning` unless the spec calls for it |
| Shapes | `new sketch.ShapePath({ shapeType: sketch.ShapePath.ShapeType.Rectangle \| Oval \| … })` | Use a Frame fill rather than a background rectangle |
| Images | `new sketch.Image({ image: { path } })`, image fills, image overrides in symbols | See `get_guide` `assets` |
| SVG icons | Import as a Graphic (see `get_guide` `assets`) | Don't hand-draw complex icons |
| Data (random names, avatars) | | Manual (the Data feature) |

## Prototyping

| Feature | API | Agent? |
|---|---|---|
| Links between frames, slide animations | `layer.flow = { target, animationType }` | Yes |
| Back link | `layer.flow = { target: sketch.Flow.BackTarget }` | Yes |
| Start Points | `frame.flowStartPoint = true` | Yes |
| Hotspots | `new sketch.HotSpot(...)` / `HotSpot.fromLayer(layer)` | Yes |
| Overlays, fixed elements, scroll areas, Smart Animate, Dissolve, visibility interactions | Not in the public API | Manual (Prototype tab in the Inspector) |

## Handoff

| Feature | API |
|---|---|
| Export presets on layers | `layer.exportFormats = [{ size: '2x', fileFormat: 'png' }]` |
| Export files | `sketch.export(layer, { formats, scales, output, filename, overwriting })` |
| CSS / SVG code | Manual: right-click → Copy CSS / Copy SVG Code, or Inspect in the web app |
| Share links / comments / Workspace | Manual |
