<!--
author:  André Dietrich

email:   LiaScript@web.de

version: 1.0.0

language: en

narrator: US English Female

comment: Collaborative drawing on a shared canvas. Everyone draws on an own
         layer, which is kept locally and joins the classroom when you
         connect. The classroom mode decides who sees which layer.

persistent: true

@onload
// A stroke is one value: [x, y] for a dot, [x1, y1, x2, y2, ...] for a line.
window.collabCanvas = function (id, mode) {
  const canvas = document.getElementById('canvas_' + id)
  const legend = document.getElementById('legend_' + id)
  const tools = document.getElementById('tools_' + id)

  if (!canvas || !legend || !tools) {
    return setTimeout(() => window.collabCanvas(id, mode), 100)
  }

  // "<template>:<name>" keeps it apart from layers of other templates
  const layer = window.LIA?.classroom?.layer?.('CollaborativeDrawing:' + id)
  if (!layer?.set) {
    legend.textContent = 'This LiaScript version does not support LIA.classroom.layer.'
    return
  }

  const ctx = canvas.getContext('2d')
  let layers = []
  let tool = 'draw'
  let action = null // the stroke or drag in progress
  const hidden = new Set() // ids of layers switched off in the legend
  let legendKey = ''
  const history = [] // { id, values } of a layer before each change
  const future = [] // the same for redo

  // the same color and emoji a person has in the chat
  const color = (l) => l.color || '#888'

  function draw(s, c, dx = 0, dy = 0) {
    ctx.strokeStyle = ctx.fillStyle = c
    ctx.lineWidth = 4
    ctx.lineCap = ctx.lineJoin = 'round'
    ctx.beginPath()
    if (s.length === 2) {
      ctx.arc(s[0] + dx, s[1] + dy, 6, 0, 2 * Math.PI)
      ctx.fill()
    } else {
      ctx.moveTo(s[0] + dx, s[1] + dy)
      for (let i = 2; i < s.length; i += 2) ctx.lineTo(s[i] + dx, s[i + 1] + dy)
      ctx.stroke()
    }
  }

  function redraw() {
    ctx.clearRect(0, 0, canvas.width, canvas.height)
    for (const l of layers) {
      if (hidden.has(l.id)) continue
      l.values.forEach((s, i) => {
        const moving = action?.type === 'move' && action.id === l.id && action.index === i
        draw(s, color(l), moving ? action.dx : 0, moving ? action.dy : 0)
      })
    }
    renderLegend()
  }

  function button(label, attrs) {
    const b = document.createElement('button')
    b.className = 'lia-btn lia-btn--outline'
    b.style.margin = '0.25em 0.5em 0 0'
    b.style.padding = '0.1em 0.6em'
    b.textContent = label
    for (const key in attrs) b.setAttribute(key, attrs[key])
    return b
  }

  // one toggle button per person, rebuilt only when people or their state
  // change, so a button keeps its focus while others draw
  function renderLegend() {
    // own drawing first, then alphabetically
    const people = layers
      .filter((l) => l.values.length > 0)
      .sort((a, b) => b.me - a.me || (a.name || '').localeCompare(b.name || ''))
    const key = JSON.stringify(people.map((l) => [l.id, l.name, l.me, hidden.has(l.id)]))
    if (key === legendKey) return
    legendKey = key

    const buttons = people.map((l) => {
      const on = !hidden.has(l.id)
      const b = button(`${l.symbol || '●'} ${l.me ? 'you' : l.name || 'anonymous'}`, {
        'data-id': l.id,
        'aria-pressed': on,
        title: (on ? 'Hide' : 'Show') + ' this drawing',
      })
      b.style.color = color(l)
      b.style.opacity = on ? '1' : '0.4'
      if (!on) b.style.textDecoration = 'line-through'
      return b
    })

    if (people.length > 1) {
      buttons.push(
        button('all', { 'data-show': 'all', title: 'Show all drawings' }),
        button('none', { 'data-show': 'none', title: 'Hide all drawings' }),
      )
    }

    legend.replaceChildren(...buttons)
  }

  legend.addEventListener('click', (e) => {
    const b = e.target.closest('button')
    if (!b) return
    if (b.dataset.show === 'all') hidden.clear()
    else if (b.dataset.show === 'none') layers.forEach((l) => hidden.add(l.id))
    else if (hidden.has(b.dataset.id)) hidden.delete(b.dataset.id)
    else hidden.add(b.dataset.id)
    redraw()
    // the buttons were rebuilt, keep the keyboard focus on the same one
    legend.querySelector(b.dataset.id ? `[data-id="${b.dataset.id}"]` : `[data-show="${b.dataset.show}"]`)?.focus()
  })

  function pos(e) {
    const r = canvas.getBoundingClientRect()
    return [
      Math.round(((e.clientX - r.left) * canvas.width) / r.width),
      Math.round(((e.clientY - r.top) * canvas.height) / r.height),
    ]
  }

  // the current strokes of a layer, the own one straight from the layer
  function strokes(l) {
    return l.me ? layer.get() : l.values.slice()
  }

  function write(id, values) {
    const l = layers.find((l) => l.id === id)
    if (!l || l.me) layer.set(values)
    else layer.set(values, id) // only the classroom owner may do this
  }

  // the topmost stroke near p, which this user may change
  function hit(p) {
    for (const l of [...layers].reverse()) {
      if (!l.editable || hidden.has(l.id)) continue
      const own = strokes(l)
      for (let i = own.length - 1; i >= 0; i--) {
        const s = own[i]
        for (let j = 0; j < s.length; j += 2) {
          const [x1, y1] = [s[j], s[j + 1]]
          const [x2, y2] = s.length > j + 2 ? [s[j + 2], s[j + 3]] : [x1, y1]
          const len = (x2 - x1) ** 2 + (y2 - y1) ** 2
          const t = len ? Math.max(0, Math.min(1, ((p[0] - x1) * (x2 - x1) + (p[1] - y1) * (y2 - y1)) / len)) : 0
          if (Math.hypot(p[0] - x1 - t * (x2 - x1), p[1] - y1 - t * (y2 - y1)) < 10) {
            return { id: l.id, own, index: i }
          }
        }
      }
    }
  }

  const mine = () => layers.find((l) => l.me) || { id: '', me: true, values: [] }

  function change(id, next, before) {
    const l = layers.find((l) => l.id === id) || mine()
    history.push({ id, values: before || strokes(l) })
    future.length = 0
    write(id, next)
  }

  function step(from, to) {
    const entry = from.pop()
    if (!entry) return
    const l = layers.find((l) => l.id === entry.id) || mine()
    to.push({ id: entry.id, values: strokes(l) })
    write(entry.id, entry.values)
  }

  const commands = {
    undo: () => step(history, future),
    redo: () => step(future, history),
    clear() {
      if (layer.get().length) change(mine().id, [])
    },
  }

  function select(button) {
    tool = button.dataset.tool
    for (const b of tools.querySelectorAll('[data-tool]')) {
      b.setAttribute('aria-pressed', b === button)
      b.style.outline = b === button ? '3px solid currentColor' : ''
    }
    canvas.style.cursor = { draw: 'crosshair', move: 'move', erase: 'cell' }[tool]
  }

  select(tools.querySelector('[data-tool="draw"]'))

  tools.addEventListener('click', (e) => {
    const button = e.target.closest('button')
    if (!button) return
    if (button.dataset.tool) select(button)
    else commands[button.dataset.command]()
  })

  canvas.addEventListener('keydown', (e) => {
    if (!(e.ctrlKey || e.metaKey)) return
    const key = e.key.toLowerCase()
    if (key === 'z' && !e.shiftKey) commands.undo()
    else if (key === 'y' || (key === 'z' && e.shiftKey)) commands.redo()
    else return
    e.preventDefault()
  })

  canvas.addEventListener('pointerdown', (e) => {
    const p = pos(e)
    canvas.setPointerCapture(e.pointerId)

    if (tool === 'erase') {
      const h = hit(p)
      if (h) change(h.id, h.own.filter((_, j) => j !== h.index), h.own)
    } else if (tool === 'move') {
      const h = hit(p)
      if (h) action = { type: 'move', ...h, from: p, dx: 0, dy: 0 }
    } else if (mode === 'dots') {
      change(mine().id, [...layer.get(), p])
    } else {
      action = { type: 'draw', start: layer.get().length, points: p }
    }
  })

  canvas.addEventListener('pointermove', (e) => {
    if (!action) return
    const p = pos(e)

    if (action.type === 'move') {
      action.dx = p[0] - action.from[0]
      action.dy = p[1] - action.from[1]
      return redraw()
    }

    const last = action.points.slice(-2)
    // fewer but longer segments, every value is stored and sent
    if (Math.hypot(p[0] - last[0], p[1] - last[1]) < 4) return
    action.points.push(...p)
    // live for everyone, joined to one stroke when the pointer is released
    layer.push([...last, ...p])
  })

  function stop() {
    const a = action
    action = null
    if (!a) return

    if (a.type === 'draw') {
      // the current layer, the owner may have changed other strokes meanwhile
      const before = layer.get().slice(0, a.start)
      change(mine().id, [...before, a.points], before)
    } else if (a.dx || a.dy) {
      const next = a.own.slice()
      next[a.index] = next[a.index].map((v, i) => v + (i % 2 ? a.dy : a.dx))
      change(a.id, next, a.own)
    } else {
      redraw()
    }
  }

  canvas.addEventListener('pointerup', stop)
  canvas.addEventListener('pointercancel', stop)

  layer.on((l, info) => {
    layers = l
    // the owner changed this drawing, an undo must not take that back
    if (info?.remote) history.length = future.length = 0
    redraw()
  })
}
@end

@Collaborative.lines: @Collaborative.canvas(@uid,lines,@0,@1,@2)

@Collaborative.dots: @Collaborative.canvas(@uid,dots,@0,@1,@2)

@Collaborative.canvas
<div>
<div id="tools_@0" role="toolbar" aria-label="Drawing tools" style="display: flex; gap: 0.25em; margin-bottom: 0.25em">
<button class="lia-btn lia-btn--outline" style="min-width: 2.5em; font-size: 1.1em" data-tool="draw" aria-pressed="true" title="Draw" aria-label="Draw">✏️</button>
<button class="lia-btn lia-btn--outline" style="min-width: 2.5em; font-size: 1.1em" data-tool="move" aria-pressed="false" title="Move a stroke" aria-label="Move a stroke">✋</button>
<button class="lia-btn lia-btn--outline" style="min-width: 2.5em; font-size: 1.1em" data-tool="erase" aria-pressed="false" title="Erase a stroke" aria-label="Erase a stroke">🧽</button>
<button class="lia-btn lia-btn--outline" style="min-width: 2.5em; font-size: 1.1em" data-command="undo" title="Undo (Ctrl+Z)" aria-label="Undo">↶</button>
<button class="lia-btn lia-btn--outline" style="min-width: 2.5em; font-size: 1.1em" data-command="redo" title="Redo (Ctrl+Y)" aria-label="Redo">↷</button>
<button class="lia-btn lia-btn--outline" style="min-width: 2.5em; font-size: 1.1em" data-command="clear" title="Clear my drawing" aria-label="Clear my drawing">🗑️</button>
</div>
<canvas
  id="canvas_@0"
  width="@2"
  height="@3"
  tabindex="0"
  aria-label="Drawing canvas"
  style="border: 1px solid #888; width: 100%; touch-action: none; cursor: crosshair; background: url('@4') center/cover no-repeat;">
</canvas>
<div id="legend_@0" role="group" aria-label="Show or hide drawings" style="font-size: 0.85em; min-height: 2em"></div>
</div>
<script run-once="true">
// on a revisit the slide can be faster than @onload
;(function start() {
  if (window.collabCanvas) window.collabCanvas("@0", "@1")
  else setTimeout(start, 50)
})()
""
</script>
@end
-->

# Collaborative Drawing

    --{{0}}--
A collaborative drawing canvas for classrooms, built on the LiaScript layer API.
Everyone draws on an own layer.
Without a classroom you only see your own drawing, it is stored locally and is
still there after a reload.
As soon as you join a classroom, your layer is synchronized with all others.

**Try it on LiaScript:**

https://liascript.github.io/course/?https://raw.githubusercontent.com/LiaTemplates/CollaborativeDrawing/main/README.md

    --{{1}}--
Load the macros via import and set the persistence flag to true, so the
drawings keep being updated while you are on another slide.

      {{1}}
```text
import: https://raw.githubusercontent.com/LiaTemplates/CollaborativeDrawing/main/README.md

persistent: true
```

      {{1}}
This version needs LiaScript 2.5.0 or newer (`LIA.classroom.layer`).
The first version, which only synchronizes drawings between connected
participants (pubsub, nothing is stored), stays available as tag `0.0.1`:

      {{1}}
```text
import: https://raw.githubusercontent.com/LiaTemplates/CollaborativeDrawing/0.0.1/README.md
```

## `@Collaborative.lines`

    --{{0}}--
Draw lines with the mouse, a pen or a finger.
Width and height define the resolution, the canvas always uses the full
available width.

`@Collaborative.lines(width,height)`

@Collaborative.lines(640,320)

### With background image

    --{{0}}--
The optional third parameter is a background image.
The link style is preferred, since relative paths get resolved then.

`@[Collaborative.lines(640,320)](url)`

@[Collaborative.lines(640,320)](https://upload.wikimedia.org/wikipedia/commons/thumb/7/7e/Pieter_Brueghel_the_Elder_-_The_Dutch_Proverbs_-_Google_Art_Project.jpg/1280px-Pieter_Brueghel_the_Elder_-_The_Dutch_Proverbs_-_Google_Art_Project.jpg)

## `@Collaborative.dots`

    --{{0}}--
Every click or tap places a dot.
Combined with a background image this can be used to mark places on a map.

`@[Collaborative.dots(width,height)](url)`

@[Collaborative.dots(640,320)](https://upload.wikimedia.org/wikipedia/commons/3/31/A_large_blank_world_map_with_oceans_marked_in_blue-edited.png)

## Tools

    --{{0}}--
Above every canvas there is a small toolbar.
The tools change your own layer, the drawings of others stay untouched.
Only the owner of a classroom can also move and erase the strokes of all
others, and they see these changes on their own canvas.

| Tool | What it does |
| ---- | ------------ |
| ✏️ | draw lines, or place dots in `@Collaborative.dots` |
| ✋ | drag one of your strokes to another place |
| 🧽 | click one of your strokes to remove it |
| ↶ ↷ | undo and redo your last steps, also `Ctrl+Z` and `Ctrl+Y` on the canvas |
| 🗑️ | clear your own drawing, this can be undone as well |

    --{{1}}--
Undo works per stroke: while you draw, the others see the line growing, when
you release it becomes one stroke, which is undone as a whole.
The undo history lasts until the page is reloaded, or until the owner
changes your drawing, so that an undo cannot take back the owner's change.

## Who sees what?

    --{{0}}--
Visibility follows the classroom mode, in the same way as quiz answers.

| Situation             | You see                                    |
| --------------------- | ------------------------------------------ |
| no classroom          | your own layer                             |
| classroom, Shared     | all layers                                 |
| classroom, Summary    | your own layer, the owner sees all layers  |
| classroom, Details    | your own layer, the owner sees all with names |
| after leaving         | your own layer, including what you drew in the classroom |

    --{{1}}--
Every person draws in the same color and with the same emoji as in the chat,
the buttons below a canvas name who drew what.
A click on a name hides or shows that drawing, `all` and `none` switch all of
them at once.
This only changes your own view, for example to look at the solutions of a
class one by one, and hidden drawings cannot be moved or erased.

## Implementation

    --{{0}}--
The whole logic lives in the `@onload` block of this file.
It only uses the layer part of the LiaScript classroom API.

```js
// an own layer per peer, stored locally and synchronized in a classroom,
// named "<template>:<name>", so it does not mix with other templates
const layer = LIA.classroom.layer('CollaborativeDrawing:' + id)

// called with all visible layers, whenever one of them changes
layer.on((layers) => {
  // layers = [{ id, name, me, editable, color, symbol, values: [...] }]
})

// one value per stroke: [x, y] for a dot, [x1, y1, x2, y2, ...] for a line
layer.push(stroke)

// undo, erase, move: read the own layer, change it, write it back
const own = layer.get()
layer.set(own.slice(0, -1))

// the classroom owner may also change the layers of others,
// layers[i].editable tells which ones
layer.set(values, layers[i].id)
```

    --{{1}}--
`set` only sends what actually changed: removing the last stroke is a single
delete for the others, not a copy of the whole drawing.

    --{{2}}--
Layer names and pubsub topics are shared by all templates of a course.
That is why the name starts with the template, `CollaborativeDrawing:`, and
on purpose the same name lets several templates work on the same drawing.
The rest of the name is built from `@uid`, so it stays the same for everyone
using the same version of a course.
If you reorder canvases in a course, previously drawn layers are no longer
found.
