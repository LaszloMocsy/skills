# Sketch features: what to use and how to drive each one

Checked against Sketch 2026.4 (API 2.0.0). Columns: the Sketch feature (as the docs name it), the API for it, and whether an agent can set it through `run_code` or an MCP tool. "Manual" means it is only available in the Sketch UI, so give the user the exact steps. Never reach for private APIs.

## Containers and layout

| Feature | When to use | API | Agent? |
|---|---|---|---|
| Frame | Screens, sections of a screen, any layout container | `new sketch.Group.Frame({...})`; `group.convertToFrame()` | Yes |
| Graphic | Icons and illustrations (they scale proportionally; their children have no pins) | `new sketch.Group.Graphic({...})`; `convertToGraphic()` | Yes |
| Section (2026.4) | Group whole screens on the canvas: a flow, a feature, a breakpoint set | `new sketch.Group.Section({ name, parent, layers })`; `isSection`, `layer.getParentSection()`, `convertToSection()`, `section.ungroup()`; can take a `stackLayout` | Yes. Fixed size; can't be a component, link target or Start Point |
| Stack (stack layout) | Rows, columns, lists, navs, forms, buttons, cards, grids (`wraps`) | `group.stackLayout = { direction, gap, padding, alignItems, justifyContent, wraps, alignContent, crossAxisGap, stackingOrder, bordersAffectLayout }`, then `stackLayout.apply()` | Yes. Negative `gap` overlaps items |
| Sizing (Fixed / Fit / Fill / Relative) | Responsive behavior inside Frames and Stacks | `layer.horizontalSizing` / `verticalSizing` = `sketch.FlexSizing.*`; `percentageSize`, `minSize`, `maxSize` | Yes |
| Pins | Freeform children of a Frame that must stick to edges (tab bars, FABs) | `layer.horizontalPins` / `verticalPins` = `sketch.Pin.Min / Max / All / None` (bitmask) | Yes |
| Ignore stack / keep space when hidden | Badges and overlays inside a Stack | `ignoresStackLayout`, `preservesSpaceInStackLayoutWhenHidden` | Yes |
| Stacking order (z-index) | Overlapping Stack items, e.g. avatar piles with negative gap | `stackLayout.stackingOrder = sketch.StackLayout.StackingOrder.FirstOnTop \| LastOnTop` | Yes |
| Clip content | Turn off on structural wrappers so shadows aren't cut | `group.clipsContents` | Yes |
| Scale (Scale tool) | Resize a layer with its text, borders, radii, shadows | `layer.scale(factor, { origin: sketch.ScaleOrigin.Center })` | Yes |
| Rotate / flip | | `layer.rotation`, `flippedHorizontally`, `flippedVertically` (also `layer.transform`) | Yes |
| Smart Layout | Legacy. Use Stacks instead | `smartLayout`, `instance.resizeWithSmartLayout()` | Avoid |
| Layout grids and guides | Visual column guides | Not in the public API | Manual (View → Canvas → Layout Settings) |

## Pages and organization

| Feature | API | Agent? |
|---|---|---|
| Pages | `new sketch.Page({ name, parent: doc })`, `doc.pages` | Yes |
| Components page (formerly Symbols page) | `sketch.Page.getSymbolsPage(doc)`; `Page.createSymbolsPage()` creates "Components" | Yes |
| Placing top-level frames | Measure `page.layers` (not `canvasLevelFrames`) and place to the right | Yes |
| Send To page/section, Move to Frame | | Manual (Layer menu) |

## Design system

| Feature | API | Agent? |
|---|---|---|
| Color Variables | `doc.swatches.push({ name, color })`; on layers: `fill.swatch`, `border.swatch`, `shadow.swatch`, `text.style.textSwatch`, `style.tint = { swatch }` | Yes |
| Text Styles | `doc.sharedTextStyles.push({ name, style: { fontFamily, fontSize, fontWeight, textSwatch } })`; `text.sharedStyle = style` | Yes |
| Layer Styles | `doc.sharedLayerStyles.push({ name, style })`; `layer.sharedStyle = style` | Yes |
| Components (formerly Symbols) | `sketch.SymbolMaster.fromFrame(frame)`, `master.createNewInstance()`, `doc.getSymbols()`, `master.duplicate()` | Yes |
| Components with Variants (2026.4) | `new sketch.Group.VariantSet({ name, parent, variantProperties, layers })`; `master.variantConfiguration`; `set.variantProperties`, `set.defaultVariantMaster`, `set.getBestVariantMasterForConfiguration({...})`; `doc.getVariantSets()`, `doc.getVariantSetWithID()`; `master.isVariant`, `master.getParentVariantSet()`; `set.ungroup()` | Yes. Add Variant / Combine as Variants in the UI do the same |
| Swap variant on an instance | `instance.symbolId = otherVariant.symbolId` | Yes |
| Overrides | `get_symbol_overrides`, then `instance.overrides.find(o => o.id === prefix + '_' + property).value = …`; several at once: `instance.updateOverrides(overrides => …)` | Yes |
| Override properties | `stringValue`, `textStyle`, `layerStyle`, `textSize`, `textWeight`, `textColor`, `textDecoration`, `textHAlign`, `fillColor`, `color:fill-N` (also border, shadow, innershadow), `blendMode:*`, `opacity:*`, `symbolID`, `image`, `imageResizeBehavior`, `isVisible`, `preservesSpaceWhenHidden`, `flowDestination`, `horizontalSizing`, `verticalSizing` | Yes |
| Nested components | Nest instances inside a source; swap with the `symbolID` override; inspect with `instance.overridesForExpandedLayer(layer)` | Yes |
| Allow/lock overrides | `override.editable` (read) | Setting: Manual (Component settings in the Inspector) |
| Frame / Graphic Templates | `frame.isTemplate = true`; `doc.frameTemplates`; library templates come from `get_design_assets` `frameTemplate`, then `.import()` | Yes |
| Libraries | Read with `get_libraries` / `get_design_assets`; import with `ref.import()` (assign it in the same statement); variant sets via `library.getImportableVariantSetReferencesForDocument(doc)` | Use: yes. Creating or sharing libraries: manual |
| Assets view (formerly Components view) | | Manual |
| Document templates (Workspace) | | Manual (File → Document Settings → Template) |

## Styling

| Feature | API | Notes |
|---|---|---|
| Fills: color, gradient, pattern/image | `style.fills = [{ fillType: sketch.Style.FillType.Color \| Gradient \| Pattern, … }]` | Gradient `colorInterpolation`: `RGB` (default), `Oklab`, `Oklch` |
| Borders | `style.borders = [{ swatch \| color, thickness, position }]`, `borderOptions` (dashes, caps) | Per-side: `border.sides = { top, right, bottom, left }`, `null` resets; `hasIndividualSides` |
| Corners | `style.corners = { radii, style: sketch.Style.CornerStyle.Auto \| Rounded \| Smooth \| Angled \| InsideSquare \| InsideArc, smoothing, concentric }` | Smooth for iOS squircles. `concentric` follows the parent frame; don't also set radii |
| Shadows | `style.shadows` / `style.innerShadows = [{ color \| swatch, x, y, blur, spread }]` | Subtle alpha, e.g. `#0F172A14`. Multiple shadows work on fill-less frames and groups |
| Blurs | `style.blurs = [{ blurType: sketch.Style.BlurType.Gaussian \| Motion \| Zoom \| Background \| Glass, radius, … }]`; `progressive: true` + `gradient` | Glass: `brightness`, `distortion`, `depth`, `chromaticAberration`, `saturation`, `hasSpecularHighlights` |
| Fade | `style.progressiveAlpha = { gradient… }`, `null` removes | Gradient alpha controls the layer's opacity. Combines with Glass |
| Tints | `group.style.tint = { swatch }` | Recolors a whole Graphic or icon |
| Masks | `layer.masksSiblings`, `breaksMaskChain`, `closestMaskingLayer` | See `get_guide` `styling` |
| Opacity and blend modes | `style.opacity`, `style.blendingMode` | |
| Copy style between layers | `fill.toJSON()` + reattach `swatch` | Prefer a shared style for 2+ layers |

## Content

| Feature | API | Notes |
|---|---|---|
| Text | `new sketch.Text({ text, name })`, `fixedWidth`, `style.alignment`, `adjustToFit()` | Prefer shared text styles. In a Stack, set long text to `FlexSizing.Fill` so it wraps. Avoid setting `lineHeight` / `kerning` unless the spec calls for it |
| Shapes | `new sketch.ShapePath({ shapeType: sketch.ShapePath.ShapeType.Rectangle \| Oval \| … })` | Use a Frame fill rather than a background rectangle |
| Images | `new sketch.Image({ image: { path } })`, `imageLayer.image = { path \| base64 }`, image overrides | See `get_guide` `assets` |
| SVG / PDF / EPS import | `sketch.createLayerFromData(data, 'svg' \| 'pdf' \| 'eps' \| 'bitmap')` | Don't hand-draw complex icons |
| Data (random names, avatars) | | Manual (the Data feature) |

## Prototyping

| Feature | API | Agent? |
|---|---|---|
| Links between frames, slide animations | `layer.flow = { target, animationType, maintainScrollPosition }` | Yes |
| Back link | `layer.flow = { target: sketch.Flow.BackTarget }` | Yes |
| Start Points | `frame.flowStartPoint = true` (Frames only; throws on Sections and Components with Variants) | Yes |
| Hotspots | `new sketch.HotSpot(...)` / `HotSpot.fromLayer(layer)` | Yes |
| Overlays, fixed elements, scroll areas, Smart Animate, Dissolve, hover/press, visibility interactions | Not in the public API | Manual (Prototype tab in the Inspector) |

## Collaboration (Workspace documents, signed in)

| Feature | MCP tool | Agent? |
|---|---|---|
| Read comment threads (annotations) | `list_comments` (`filter`, `subjectID`, `annotationID`, `cursor`) | Yes |
| Reply, start a thread, edit own comment, resolve / reopen | `save_comment` | Yes, after the user confirms the text |
| Delete reply or thread | `delete_comment` | Only when asked; permanent |
| Comments and mentions across documents | `list_notifications` | Yes |
| Share links, permissions, Workspace management, Download | | Manual |

## Handoff

| Feature | API |
|---|---|
| Export presets on layers | `layer.exportFormats = [{ size: '2x', fileFormat: 'png' }]` |
| Export files | `sketch.export(layer, { formats, scales, exportFormats, output, filename, overwriting, trimmed })` — png, jpg, webp, heic, tiff, svg, pdf, json |
| CSS / SVG code | Manual: right-click → Copy CSS / Copy SVG Code (Copy CSS includes the corner shape), or Inspect in the web app |
| Command line | `sketchtool export frames` / `export assets` (2026.4 names; old `artboards` / `components` aliases still work) |
