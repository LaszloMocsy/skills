# Tested `run_code` recipes

Each recipe below was run against Sketch 2026.4 (API 2.0.0) through the MCP `run_code` tool, and canvas checks (`get_screenshot`) confirmed the result. Each one is a single `run_code` call. Replace `<…-ID>` placeholders with IDs logged by earlier calls. Submit scripts exactly as shown: **no comments** (Sketch's MCP rules forbid them).

Every script starts with this guard, so it fails loudly instead of writing into the wrong document:

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
if (!doc) {
  console.log(JSON.stringify({ ok: false, error: 'NO_DOCUMENT' }))
}
```

When more than one document is open, resolve yours by ID instead of by selection:

```javascript
const doc = sketch.getDocuments().find(d => d.id === '<DOC-ID>')
```

The recipes below leave the guard out for brevity. Include it anyway.

---

## 1. Color Variables (idempotent)

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const tokens = [
  { name: 'Color/Background', color: '#F8FAFCFF' },
  { name: 'Color/Surface', color: '#FFFFFFFF' },
  { name: 'Color/Border', color: '#E2E8F0FF' },
  { name: 'Color/Text/Primary', color: '#0F172AFF' },
  { name: 'Color/Text/Secondary', color: '#475569FF' },
  { name: 'Color/Brand/Primary', color: '#4F46E5FF' },
  { name: 'Color/Brand/On Primary', color: '#FFFFFFFF' },
  { name: 'Color/Brand/Subtle', color: '#EEF2FFFF' }
]
const existing = doc.swatches.map(s => s.name)
tokens.forEach(t => {
  if (!existing.includes(t.name)) doc.swatches.push(t)
})
console.log(JSON.stringify({ ok: true, swatches: doc.swatches.map(s => ({ name: s.name, color: String(s.color) })) }))
```

To change a token later, set `swatch.color = '#…'` and then call `doc.processPendingChanges()`. Every layer that uses it updates. Library Color Variables are read-only: change them in the library itself.

## 2. Text Styles bound to Color Variables

First check that the font is installed. An uninstalled family is silently ignored (a fresh text keeps Helvetica):

```javascript
const sketch = require('sketch')
const probe = new sketch.Text({ text: 'Ag' })
probe.style.fontFamily = 'Inter'
console.log(JSON.stringify({ ok: true, requested: 'Inter', got: String(probe.style.fontFamily) }))
```

If `got` differs from what you requested, use `'system'` or another installed family, and tell the user which font to install.

Since 2026.4 a plain style object keeps all text attributes and the `textSwatch` link:

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const sw = name => doc.swatches.find(s => s.name === name)
const family = 'system'
const scale = [
  { name: 'Type/Display', size: 48, weight: 9, color: 'Color/Text/Primary' },
  { name: 'Type/Heading', size: 24, weight: 7, color: 'Color/Text/Primary' },
  { name: 'Type/Body', size: 16, weight: 5, color: 'Color/Text/Secondary' },
  { name: 'Type/Label', size: 14, weight: 6, color: 'Color/Text/Primary' }
]
const existing = doc.sharedTextStyles.map(s => s.name)
scale.forEach(t => {
  if (existing.includes(t.name)) return
  doc.sharedTextStyles.push({
    name: t.name,
    style: { fontFamily: family, fontSize: t.size, fontWeight: t.weight, textSwatch: sw(t.color) }
  })
})
console.log(JSON.stringify({ ok: true, textStyles: doc.sharedTextStyles.map(s => ({ name: s.name, id: s.id, swatch: s.style.textSwatch ? s.style.textSwatch.name : null })) }))
```

`style.fontSize` and `style.fontWeight` still read back as `null`. To verify a style, apply it to a temporary text layer on a page, call `adjustToFit()`, read `frame.height` (48 pt system → 56, 24 → 28, 16 → 18, 14 → 17), then `remove()` the layer.

## 3. Layer Style (surface)

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const sw = name => doc.swatches.find(s => s.name === name)
if (!doc.sharedLayerStyles.find(s => s.name === 'Surface/Card')) {
  doc.sharedLayerStyles.push({
    name: 'Surface/Card',
    style: {
      fills: [{ fillType: sketch.Style.FillType.Color, swatch: sw('Color/Surface') }],
      borders: [{ swatch: sw('Color/Border'), thickness: 1 }],
      shadows: [{ color: '#0F172A14', x: 0, y: 4, blur: 16, spread: 0 }]
    }
  })
}
const s = doc.sharedLayerStyles.find(s => s.name === 'Surface/Card')
console.log(JSON.stringify({ ok: true, id: s.id }))
```

## 4. Button component that hugs its label

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const sw = name => doc.swatches.find(s => s.name === name)
let page = sketch.Page.getSymbolsPage(doc)
if (!page) {
  page = sketch.Page.createSymbolsPage()
  page.parent = doc
}
const label = new sketch.Text({ name: 'Label', text: 'Button' })
label.sharedStyle = doc.sharedTextStyles.find(s => s.name === 'Type/Label')
label.style.textSwatch = sw('Color/Brand/On Primary')
const frame = new sketch.Group.Frame({
  name: 'Button/Primary/M',
  parent: page,
  frame: { x: 0, y: 0, width: 120, height: 44 },
  style: {
    fills: [{ fillType: sketch.Style.FillType.Color, swatch: sw('Color/Brand/Primary') }],
    corners: { radii: 10, style: sketch.Style.CornerStyle.Smooth }
  },
  stackLayout: {
    direction: sketch.StackLayout.Direction.Row,
    alignItems: sketch.StackLayout.AlignItems.Center,
    justifyContent: sketch.StackLayout.JustifyContent.Center,
    padding: { vertical: 12, horizontal: 20 },
    gap: 8
  },
  layers: [label]
})
const master = sketch.SymbolMaster.fromFrame(frame)
doc.processPendingChanges()
master.frame = { x: 0, y: 0, width: master.frame.width, height: master.frame.height }
console.log(JSON.stringify({ ok: true, pageName: page.name, masterId: master.id, symbolId: master.symbolId, frame: master.frame, fills: master.style.fills.length }))
```

`createSymbolsPage()` creates a page named "Components" in 2026.4. `fills` should be `1` (the frame's own fill, no extra white background). The stack hugs the label, so the frame shrinks and shifts; resetting `x`/`y` puts it back. Then do a canvas check on `masterId`. Name it with a path (`Button/Primary/M`) even if you'll combine it into variants: the names stay meaningful in recipe 5.

## 5. Component with Variants

First make one source per combination. Duplicate the base and change only what differs:

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const sw = name => doc.swatches.find(s => s.name === name)
const base = sketch.find('#' + '<MASTER-ID>', doc)[0]
const specs = [
  { name: 'Button/Secondary/M', secondary: true, large: false, x: 0, y: 80 },
  { name: 'Button/Primary/L', secondary: false, large: true, x: 160, y: 0 },
  { name: 'Button/Secondary/L', secondary: true, large: true, x: 160, y: 80 }
]
const made = specs.map(s => {
  const m = base.duplicate()
  m.name = s.name
  if (s.secondary) {
    m.style.fills = [{ fillType: sketch.Style.FillType.Color, swatch: sw('Color/Brand/Subtle') }]
    m.layers[0].style.textSwatch = sw('Color/Brand/Primary')
  }
  if (s.large) {
    m.stackLayout.padding = { vertical: 16, horizontal: 28 }
    m.stackLayout.apply()
  }
  m.frame = { x: s.x, y: s.y, width: m.frame.width, height: m.frame.height }
  return { id: m.id, symbolId: m.symbolId, name: m.name }
})
console.log(JSON.stringify({ ok: true, made: made }))
```

`duplicate()` on a Component Source gives a new source with its own `symbolId`. Then combine them. Always pass `variantProperties` (otherwise Sketch names them "Property 1", "Property 2"), list each property's **default value first**, and set every source's `variantConfiguration`:

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const page = sketch.Page.getSymbolsPage(doc)
const plan = [
  { id: '<PRIMARY-M-ID>', config: { Style: 'Primary', Size: 'M' } },
  { id: '<SECONDARY-M-ID>', config: { Style: 'Secondary', Size: 'M' } },
  { id: '<PRIMARY-L-ID>', config: { Style: 'Primary', Size: 'L' } },
  { id: '<SECONDARY-L-ID>', config: { Style: 'Secondary', Size: 'L' } }
]
const masters = plan.map(p => sketch.find('#' + p.id, doc)[0])
const set = new sketch.Group.VariantSet({
  name: 'Button',
  parent: page,
  variantProperties: [
    { name: 'Style', values: ['Primary', 'Secondary'] },
    { name: 'Size', values: ['M', 'L'] }
  ],
  layers: masters
})
plan.forEach((p, i) => {
  masters[i].variantConfiguration = p.config
})
doc.processPendingChanges()
console.log(JSON.stringify({
  ok: true,
  setId: set.id,
  props: set.variantProperties.map(p => ({ name: p.name, values: p.values })),
  variants: set.layers.map(m => ({ id: m.id, name: m.name, config: m.variantConfiguration })),
  defaultVariant: set.defaultVariantMaster.name
}))
```

Sources are renamed to their configuration (`Primary/M`). `defaultVariant` should be `Primary/M`. Rename an existing property with `set.variantProperties.find(p => p.name === 'Property 1').name = 'Style'`; dissolve a set with `set.ungroup()` (returns the sources, clears their configuration — only when the user asks).

Optional: lay the variants out in a padded grid. The set has a fixed size, so a wrapping Stack is the easiest way to keep it tidy:

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const set = doc.getVariantSetWithID('<SET-ID>')
set.stackLayout = {
  direction: sketch.StackLayout.Direction.Row,
  wraps: true,
  gap: 24,
  crossAxisGap: 24,
  padding: 24,
  alignItems: sketch.StackLayout.AlignItems.Center
}
set.horizontalSizing = sketch.FlexSizing.Fixed
set.frame = { x: set.frame.x, y: set.frame.y, width: 274, height: set.frame.height }
const drawn = ['Primary/M', 'Secondary/M', 'Primary/L', 'Secondary/L']
drawn.forEach((name, i) => {
  const l = set.layers.find(x => x.name === name)
  if (l) l.index = drawn.length - 1 - i
})
set.stackLayout.apply()
set.frame = { x: 0, y: 0, width: set.frame.width, height: set.frame.height }
console.log(JSON.stringify({ ok: true, frame: set.frame }))
```

Width = two widest variants + gap + 2 × padding. `drawn` lists reading order; indices are reversed because Stacks pack the highest index first. Do a canvas check on `setId`.

## 6. Screen frame placed beside everything on the page

Measure `page.layers` (page coordinates, includes Sections). Do **not** measure `page.canvasLevelFrames`: frames inside a section report section coordinates and the section itself is missing, so the new frame overlaps.

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const page = doc.pages.find(p => p.id === '<PAGE-ID>')
const sw = name => doc.swatches.find(s => s.name === name)
const maxX = page.layers.reduce((m, l) => Math.max(m, l.frame.x + l.frame.width), 0)
const x = page.layers.length ? maxX + 100 : 0
const screen = new sketch.Group.Frame({
  name: 'Home — Desktop',
  parent: page,
  frame: { x: x, y: 0, width: 1440, height: 1024 },
  style: { fills: [{ fillType: sketch.Style.FillType.Color, swatch: sw('Color/Background') }] }
})
console.log(JSON.stringify({ ok: true, screenId: screen.id, frame: screen.frame, fills: screen.style.fills.length }))
```

Passing `style.fills` gives exactly one fill bound to the Color Variable. Take `<PAGE-ID>` from `get_document_info`, or use `doc.selectedPage` if the user means the page they're viewing.

## 7. Section of screens

Wrap existing top-level frames in a section, give it a Row Stack so frames tile with even spacing, and let it grow as frames are added. Top-level frames come with `Relative` sizing: set them to `Fixed` first, or the Stack collapses them to 0.1 × 0.1.

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const page = doc.pages.find(p => p.id === '<PAGE-ID>')
const frames = ['<SCREEN-1-ID>', '<SCREEN-2-ID>'].map(id => sketch.find('#' + id, doc)[0])
const sizes = frames.map(f => ({ w: f.frame.width, h: f.frame.height }))
frames.forEach((f, i) => {
  f.horizontalSizing = sketch.FlexSizing.Fixed
  f.verticalSizing = sketch.FlexSizing.Fixed
  f.frame = { x: f.frame.x, y: f.frame.y, width: sizes[i].w, height: sizes[i].h }
})
const section = new sketch.Group.Section({ name: 'Onboarding', parent: page, layers: frames.slice().reverse() })
section.stackLayout = {
  direction: sketch.StackLayout.Direction.Row,
  alignItems: sketch.StackLayout.AlignItems.Start,
  gap: 100,
  padding: 80
}
section.stackLayout.apply()
console.log(JSON.stringify({ ok: true, sectionId: section.id, frame: section.frame, kids: section.layers.map(l => ({ name: l.name, frame: l.frame })) }))
```

To add another screen later: create it with `parent: section`, set both sizings to `Fixed`, restore its `frame`, set `index = 0` (to append it at the end), then `section.stackLayout.apply()`. Check with `get_document_info` (frames list `section`/`sectionID`; pages list `sections`). Without a Stack, a section keeps its size, so resize it yourself (`get_guide` `organization`).

## 8. Fixed-width content column (Stack)

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const screen = sketch.find('#' + '<SCREEN-ID>', doc)[0]
const content = new sketch.Group.Frame({
  name: 'Content',
  parent: screen,
  frame: { x: 156, y: 96, width: 1128, height: 100 },
  background: { enabled: false },
  stackLayout: {
    direction: sketch.StackLayout.Direction.Column,
    alignItems: sketch.StackLayout.AlignItems.Stretch,
    gap: 48
  }
})
content.clipsContents = false
content.horizontalSizing = sketch.FlexSizing.Fixed
content.frame = { x: 156, y: 96, width: 1128, height: content.frame.height }
content.stackLayout.apply()
console.log(JSON.stringify({ ok: true, contentId: content.id, frame: content.frame }))
```

On a 1440 frame, a 1128-wide column at x = 156 matches a centered 12-column grid with 72 px columns and 24 px gutters. On a 393 mobile frame use x = 24, width = 345.

## 9. Section of text appended at the bottom of a column Stack

The `layers` array is in **reverse** reading order. `index = 0` puts the new block after the existing ones. Long text must be set to `Fill` or it runs past the column instead of wrapping.

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const content = sketch.find('#' + '<CONTENT-ID>', doc)[0]
const ts = name => doc.sharedTextStyles.find(s => s.name === name)
const title = new sketch.Text({ name: 'Headline', text: 'Design faster with a shared system' })
title.sharedStyle = ts('Type/Display')
const body = new sketch.Text({ name: 'Subhead', text: 'Tokens, components and layouts that stay in sync across every screen.' })
body.sharedStyle = ts('Type/Body')
const hero = new sketch.Group.Frame({
  name: 'Hero',
  parent: content,
  background: { enabled: false },
  stackLayout: { direction: sketch.StackLayout.Direction.Column, alignItems: sketch.StackLayout.AlignItems.Start, gap: 16 },
  layers: [body, title]
})
hero.clipsContents = false
hero.horizontalSizing = sketch.FlexSizing.Fill
title.horizontalSizing = sketch.FlexSizing.Fill
body.horizontalSizing = sketch.FlexSizing.Fill
hero.index = 0
hero.stackLayout.apply()
content.stackLayout.apply()
console.log(JSON.stringify({ ok: true, heroId: hero.id, body: body.frame, order: content.layers.map(l => ({ name: l.name, index: l.index })) }))
```

`body.frame.width` should equal the column width, with a height of several lines when the copy is long.

## 10. Row of equal-height cards

Build the row with `alignItems: Start` so cards can grow to fit wrapped text, apply inside-out, then switch to `Stretch` for equal heights. A Fit-height row created with `Stretch` stays stuck at its initial height and text runs into the card padding.

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const content = sketch.find('#' + '<CONTENT-ID>', doc)[0]
const ts = name => doc.sharedTextStyles.find(s => s.name === name)
const cardStyle = doc.sharedLayerStyles.find(s => s.name === 'Surface/Card')
const row = new sketch.Group.Frame({
  name: 'Features',
  parent: content,
  background: { enabled: false },
  stackLayout: { direction: sketch.StackLayout.Direction.Row, alignItems: sketch.StackLayout.AlignItems.Start, gap: 24 }
})
row.clipsContents = false
row.index = 0
const items = ['Color Variables', 'Text Styles', 'Components']
items.slice().reverse().forEach(label => {
  const h = new sketch.Text({ name: 'Title', text: label })
  h.sharedStyle = ts('Type/Heading')
  const p = new sketch.Text({ name: 'Description', text: 'Change it once and every instance updates.' })
  p.sharedStyle = ts('Type/Body')
  const card = new sketch.Group.Frame({
    name: 'Card/' + label,
    parent: row,
    stackLayout: { direction: sketch.StackLayout.Direction.Column, alignItems: sketch.StackLayout.AlignItems.Start, gap: 8, padding: 24 },
    layers: [p, h]
  })
  card.sharedStyle = cardStyle
  card.style.corners.radii = 16
  card.horizontalSizing = sketch.FlexSizing.Fill
  p.horizontalSizing = sketch.FlexSizing.Fill
})
row.horizontalSizing = sketch.FlexSizing.Fill
row.layers.forEach(c => c.stackLayout.apply())
row.stackLayout.apply()
row.stackLayout.alignItems = sketch.StackLayout.AlignItems.Stretch
row.stackLayout.apply()
content.stackLayout.apply()
console.log(JSON.stringify({ ok: true, rowId: row.id, cards: row.layers.map(c => ({ name: c.name, id: c.id, w: c.frame.width, h: c.frame.height })) }))
```

For a grid that wraps, use a fixed-width row with `wraps: true`, `crossAxisGap`, and fixed-width or `Relative` (`percentageSize`) items. Don't use `alignItems: Stretch` together with `wraps`.

## 11. Variant instance, batched overrides, variant swap

Place the variant you need:

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const parent = sketch.find('#' + '<PARENT-ID>', doc)[0]
const set = doc.getVariantSets().find(s => s.name === 'Button')
const master = set.getBestVariantMasterForConfiguration({ Style: 'Primary', Size: 'L' })
const inst = master.createNewInstance()
inst.name = 'Primary CTA'
inst.parent = parent
inst.index = 0
inst.horizontalSizing = sketch.FlexSizing.Fill
parent.stackLayout.apply()
console.log(JSON.stringify({ ok: true, instanceId: inst.id, variant: master.variantConfiguration, frame: inst.frame }))
```

Set `horizontalSizing = Fill` only when the button should span its column; a plain `Stretch` from the parent widens the instance but leaves its label at the left. For a hugging button, leave the default sizing and use a parent with `alignItems: Start`. For a standalone component (no variants) use `doc.getSymbols().find(m => m.name === '…').createNewInstance()`.

Next, call `get_symbol_overrides` (`kind: "text"` or `"all"`) for `instanceId`, then change several overrides in one batch:

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const inst = sketch.find('#' + '<INSTANCE-ID>', doc)[0]
const prefix = '<commonOverrideIDPrefix>'
let hit = 0
inst.updateOverrides(overrides => {
  overrides.forEach(o => {
    if (o.id === prefix + '_stringValue') {
      o.value = 'Get started'
      hit++
    }
  })
})
inst.parent.stackLayout && inst.parent.stackLayout.apply()
console.log(JSON.stringify({ ok: hit === 1, frame: inst.frame }))
```

Use the `overrides` argument inside the callback, never `inst.overrides`. Color overrides take a `Swatch` (`o.value = doc.swatches.find(…)`). The instance's own fill is overridable too (`<INSTANCE-ID>_color:fill-0`).

Switch an instance to another variant without losing its place:

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const inst = sketch.find('#' + '<INSTANCE-ID>', doc)[0]
const set = inst.master.getParentVariantSet()
const target = set.getBestVariantMasterForConfiguration({ Style: 'Secondary' })
inst.symbolId = target.symbolId
inst.parent.stackLayout && inst.parent.stackLayout.apply()
console.log(JSON.stringify({ ok: true, now: inst.master.variantConfiguration }))
```

A partial configuration returns the nearest match. Re-fetch `get_symbol_overrides` after a swap; text overrides on layers shared by both variants carry over.

## 12. Scale, rotate, Fade, per-side borders, perceptual gradients

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const layer = sketch.find('#' + '<LAYER-ID>', doc)[0]
layer.scale(2, { origin: sketch.ScaleOrigin.Center })
layer.rotation = 15
layer.flippedHorizontally = false
console.log(JSON.stringify({ ok: true, frame: layer.frame, radius: layer.style.corners.radiusAt(0), rotation: layer.rotation }))
```

`scale()` multiplies size, corner radii, border thickness, shadows, blurs and text sizes (a 100×60 shape with radius 10 and a 2 pt border becomes 200×120, radius 20, 4 pt border). Assigning `frame` only resizes. `ScaleOrigin`: `TopLeft` (default), `Top`, `TopRight`, `Left`, `Center`, `Right`, `BottomLeft`, `Bottom`, `BottomRight`.

Styling options verified on 2026.4 (assign the ones you need):

```javascript
layer.style.borders[0].sides = { top: 0, right: 0, bottom: 1, left: 0 }
layer.style.progressiveAlpha = {
  gradientType: sketch.Style.GradientType.Linear,
  from: { x: 0.5, y: 0 },
  to: { x: 0.5, y: 1 },
  stops: [{ position: 0, color: '#000000ff' }, { position: 1, color: '#00000000' }]
}
layer.style.fills = [{
  fillType: sketch.Style.FillType.Gradient,
  gradient: {
    gradientType: sketch.Style.GradientType.Linear,
    colorInterpolation: sketch.Style.GradientColorInterpolation.Oklch,
    from: { x: 0, y: 0.5 },
    to: { x: 1, y: 0.5 },
    stops: [{ position: 0, color: '#4F46E5ff' }, { position: 1, color: '#F59E0Bff' }]
  }
}]
```

Per-side borders give bottom-only dividers. `progressiveAlpha` is the Fade effect (`null` removes it). `Oklch`/`Oklab` interpolation avoids muddy midpoints between saturated colors.

## 13. Prototype link, start point, and export

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const cta = sketch.find('#' + '<INSTANCE-ID>', doc)[0]
const home = sketch.find('#' + '<HOME-ID>', doc)[0]
const target = sketch.find('#' + '<TARGET-FRAME-ID>', doc)[0]
cta.flow = { target: target, animationType: sketch.Flow.AnimationType.slideFromRight }
home.flowStartPoint = true
console.log(JSON.stringify({ ok: true, valid: cta.flow.isValidConnection() }))
```

Frames inside sections work as targets and Start Points. A section or a Component with Variants throws ("Can't make a Section a prototype start point").

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const home = sketch.find('#' + '<HOME-ID>', doc)[0]
const out = '/Users/<you>/Desktop/Sketch Exports'
sketch.export(home, { output: out, filename: 'home@2x', formats: 'png', scales: '2', overwriting: true })
sketch.export(home, { output: out, filename: 'home', formats: 'svg', overwriting: true })
console.log(JSON.stringify({ ok: true, files: [out + '/home@2x.png', out + '/home.svg'] }))
```

Sketch adds the extension from the format. Use one call per scale when you need deterministic `@2x`/`@3x` names. Check the files with `ls` afterwards. Use `get_screenshot` to look at a design yourself; use `sketch.export` only to produce files for the user.

## 14. Frame Template

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const f = sketch.find('#' + '<FRAME-ID>', doc)[0]
f.isTemplate = true
console.log(JSON.stringify({ ok: true, templates: doc.frameTemplates.map(t => t.name) }))
```

## 15. Comments (MCP tools, not `run_code`)

Only for documents saved to a Workspace, with Sketch signed in. Otherwise the tools return "Comments are only available in documents saved to a Workspace." or "Not signed in to a Sketch account."

1. `list_comments` `{ targetDocumentID, filter: "open" }` → annotations with `annotationID`, subject (page/frame/section ID), position, status and first comment. Add `subjectID: "<FRAME-ID>"` to scope to one frame.
2. `list_comments` `{ targetDocumentID, annotationID }` → the thread's comments, newest first.
3. Fix what the comment asks with the normal workflow, then do a canvas check.
4. Show the user the reply text, and after they confirm: `save_comment` `{ targetDocumentID, annotationID, body: "Updated the CTA contrast to 4.8:1.", resolved: true }`.

Other calls: new thread `save_comment` `{ subjectID, body, x, y }`; edit your comment `save_comment` `{ commentID, body }`; reopen `{ annotationID, resolved: false }`; `delete_comment` `{ commentID }` or `{ annotationID }` (permanent). `list_notifications` finds comments and mentions across documents; pass its `document.documentID` and `annotationID` to `list_comments`.
