# Tested `run_code` recipes

Each recipe below was run against Sketch 2026.3.1 through the MCP `run_code` tool, and screenshots confirmed the result. Each one is a single `run_code` call. Replace `<…-ID>` placeholders with IDs logged by earlier calls. Submit scripts exactly as shown: **no comments** (Sketch's MCP rules forbid them).

Every script starts with this guard, so it fails loudly instead of writing into the wrong document:

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
if (!doc) {
  console.log(JSON.stringify({ ok: false, error: 'NO_DOCUMENT' }))
}
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
  { name: 'Color/Brand/On Primary', color: '#FFFFFFFF' }
]
const existing = doc.swatches.map(s => s.name)
tokens.forEach(t => {
  if (!existing.includes(t.name)) doc.swatches.push(t)
})
console.log(JSON.stringify({ ok: true, swatches: doc.swatches.map(s => ({ name: s.name, color: String(s.color) })) }))
```

To change a token later, set `swatch.color = '#…'` and then call `doc.processPendingChanges()`. Every layer that uses it updates. Library Color Variables are read-only: change them in the library itself.

## 2. Text Styles bound to Color Variables

First check that the font is installed. An uninstalled family is silently ignored:

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const probe = new sketch.Text({ text: 'Ag' })
probe.style.fontFamily = 'Inter'
console.log(JSON.stringify({ ok: true, requested: 'Inter', got: String(probe.style.fontFamily) }))
```

If `got` differs from what you requested, use `'system'` or another installed family, and tell the user which font to install.

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
  const probe = new sketch.Text({ text: 'Ag' })
  probe.style.fontFamily = family
  probe.style.fontSize = t.size
  probe.style.fontWeight = t.weight
  probe.style.textSwatch = sw(t.color)
  doc.sharedTextStyles.push({ name: t.name, style: probe.style })
})
console.log(JSON.stringify({ ok: true, textStyles: doc.sharedTextStyles.map(s => ({ name: s.name, id: s.id })) }))
```

`style.fontSize` and `style.fontWeight` read back as `null`. To verify a style, apply it to a temporary text layer, call `adjustToFit()`, read `frame.height`, then remove the layer.

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

## 4. Button Symbol that hugs its label

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const sw = name => doc.swatches.find(s => s.name === name)
let symbolsPage = sketch.Page.getSymbolsPage(doc)
if (!symbolsPage) {
  symbolsPage = sketch.Page.createSymbolsPage()
  symbolsPage.parent = doc
}
const label = new sketch.Text({ name: 'Label', text: 'Button' })
label.sharedStyle = doc.sharedTextStyles.find(s => s.name === 'Type/Label')
label.style.textSwatch = sw('Color/Brand/On Primary')
const frame = new sketch.Group.Frame({
  name: 'Button/Primary',
  parent: symbolsPage,
  frame: { x: 0, y: 0, width: 120, height: 44 },
  style: { corners: { radii: 10, style: sketch.Style.CornerStyle.Smooth } },
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
master.style.fills = [{ fillType: sketch.Style.FillType.Color, swatch: sw('Color/Brand/Primary') }]
doc.processPendingChanges()
master.frame = { x: 0, y: 0, width: master.frame.width, height: master.frame.height }
console.log(JSON.stringify({ ok: true, masterId: master.id, symbolId: master.symbolId, frame: master.frame, fills: master.style.fills.length }))
```

`fills` should be `1`. Resetting `style.fills` after the conversion removes the automatic white canvas background, which would otherwise cover the brand color. Then screenshot `masterId`. When placing more masters on the Symbols page, offset each one with the same `canvasLevelFrames` measuring used in recipe 5.

## 5. Screen frame placed beside existing ones

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const page = doc.selectedPage
const sw = name => doc.swatches.find(s => s.name === name)
const maxX = page.canvasLevelFrames.reduce((m, f) => Math.max(m, f.frame.x + f.frame.width), 0)
const x = page.canvasLevelFrames.length ? maxX + 200 : 0
const screen = new sketch.Group.Frame({
  name: 'Home — Desktop',
  parent: page,
  frame: { x: x, y: 0, width: 1440, height: 1024 },
  background: { enabled: false },
  style: { fills: [{ fillType: sketch.Style.FillType.Color, swatch: sw('Color/Background') }] }
})
console.log(JSON.stringify({ ok: true, screenId: screen.id, frame: screen.frame, fills: screen.style.fills.length }))
```

`background: { enabled: false }` together with `style.fills` gives exactly one fill, bound to the Color Variable.

## 6. Fixed-width content column (Stack)

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

On a 1440 frame, a 1128-wide column at x = 156 matches a centered 12-column grid with 72 px columns and 24 px gutters.

## 7. Section appended at the bottom of a column Stack

The `layers` array is in **reverse** reading order. `index = 0` puts the new section after the existing ones.

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
hero.index = 0
content.stackLayout.apply()
console.log(JSON.stringify({ ok: true, heroId: hero.id, order: content.layers.map(l => ({ name: l.name, index: l.index })) }))
```

## 8. Row of equal-width cards

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
  stackLayout: { direction: sketch.StackLayout.Direction.Row, alignItems: sketch.StackLayout.AlignItems.Stretch, gap: 24 }
})
row.clipsContents = false
row.index = 0
const items = ['Color Variables', 'Text Styles', 'Symbols']
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
})
row.horizontalSizing = sketch.FlexSizing.Fill
row.stackLayout.apply()
content.stackLayout.apply()
console.log(JSON.stringify({ ok: true, rowId: row.id, cards: row.layers.map(c => ({ name: c.name, id: c.id, w: c.frame.width })) }))
```

For a grid that wraps, use a fixed-width row with `wraps: true`, `crossAxisGap`, and fixed-width or `Relative` (`percentageSize`) items. Don't use `alignItems: Stretch` together with `wraps`.

## 9. Symbol instance with a text override

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const hero = sketch.find('#' + '<HERO-ID>', doc)[0]
const master = doc.getSymbols().find(m => m.name === 'Button/Primary')
const inst = master.createNewInstance()
inst.name = 'Primary CTA'
inst.parent = hero
inst.index = 0
hero.stackLayout.apply()
console.log(JSON.stringify({ ok: true, instanceId: inst.id }))
```

Next, call `get_symbol_overrides` (`kind: "text"`) for `instanceId`, then:

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const inst = sketch.find('#' + '<INSTANCE-ID>', doc)[0]
const o = inst.overrides.find(o => o.id === '<commonOverrideIDPrefix>' + '_stringValue')
if (o) o.value = 'Get started'
inst.parent.stackLayout && inst.parent.stackLayout.apply()
console.log(JSON.stringify({ ok: !!o, value: o ? String(o.value) : null, frame: inst.frame }))
```

Because the master is a Stack, the instance resizes to fit the new label.

## 10. Prototype link, start point, and export

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

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const home = sketch.find('#' + '<HOME-ID>', doc)[0]
const out = '/Users/<you>/Desktop/Sketch Exports'
sketch.export(home, { formats: 'png', scales: '2', output: out, filename: 'home@2x.png', overwriting: true })
console.log(JSON.stringify({ ok: true, file: out + '/home@2x.png' }))
```

Use `get_screenshot` to look at a design yourself. Use `sketch.export` only to produce files for the user.

## 11. Frame Template

```javascript
const sketch = require('sketch')
const doc = sketch.getSelectedDocument()
const f = sketch.find('#' + '<FRAME-ID>', doc)[0]
f.isTemplate = true
console.log(JSON.stringify({ ok: true, templates: doc.frameTemplates.map(t => t.name) }))
```
