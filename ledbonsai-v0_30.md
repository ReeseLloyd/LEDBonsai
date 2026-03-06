# LEDBonsai — Developer Handoff
**Current version:** v0.30  
**File:** `ledbonsai-v0_30.html`  
**Format:** Single self-contained HTML file, no dependencies, no build step.

---

## What It Is

LEDBonsai renders procedurally generated bonsai trees as glowing LED dot-matrix displays. Each tree is drawn as a grid of Unicode `●` characters styled with CSS `text-shadow` glow effects, giving the appearance of an illuminated LED panel. Trees are fully deterministic from a seed, shareable via URL, and exportable as PNG or SVG.

---

## Architecture Overview

Everything lives inside one `<script>` tag. The class hierarchy:

```
SeededRandom          — Mulberry32 PRNG, all randomness flows through this
Grid                  — Infinite 2D cell buffer; tracks bounding box; renders to HTML
Vector                — 2D math helper
AnimationQueue        — Deferred action queue; drives the drawing animation
COLOR_PALETTES        — Plain object: palette key → palette definition
Tree (base)           — Pot drawing, branch drawing, shared utilities
  ├── ClassicTree     — Recursive branching tree (type 0)
  ├── FibonacciTree   — Fibonacci-angle branching (type 1)
  ├── OffsetFibTree   — Fibonacci + offset (type 2)
  └── RandomOffsetFibTree — Fibonacci + random offset (type 3)
Leaves                — LED particle system; spawns leaf/fruit clusters at branch tips
LEDBonsai             — Main app: UI binding, generate(), export, screensaver, focus mode
```

### Rendering pipeline

1. `generate()` calls `animQueue.reset()`, reads all options from the DOM
2. A `Tree` subclass is instantiated with the grid and animQueue
3. `tree.draw()` runs **synchronously** — branch cells call `setLEDAnimated()` which either writes directly to the grid (instant mode) or enqueues a closure (animated mode)
4. In animated mode, a **pre-pass** runs `tree.draw()` a second time in instant mode on a throwaway grid/queue to capture the final bounding box (`finalBounds`), so the canvas never jumps during animation. The pre-pass result is also immediately written to the canvas so `fitToViewport()` can measure real DOM dimensions before the first animation frame.
5. `animQueue.run(delay, batchSize, onUpdate, genId)` drains the queue, calling `onUpdate` (which re-renders `grid.render()` to `innerHTML`) after each batch

### Critical: RNG discipline
Every visual decision must flow through the **seeded RNG** (`this.rng`) so trees are reproducible from their seed. The main pitfalls:
- **Pot color "Random"** is resolved before the seeded RNG is created (uses `Math.random()`), so choosing "Random" vs a specific color doesn't shift the tree's sequence
- **Bark RNG budget is always identical regardless of palette** — `drawLine` unconditionally makes 4 RNG calls per pixel (3 for branch color + 1 bark roll) and 3 more if the roll fires (bark mark color, or discarded via `getBarkMarkColorRaw()` if the palette has no bark). This means switching palettes never changes tree shape.
- **`randomizeAll()`** always clears the seed field so `getOptions()` generates a fresh random seed. Both the screensaver and the 🎰 button call `randomizeAll()`, so both behave consistently.

---

## Key Data Structures

### Palette definition
```js
{
  name: 'Display Name',
  branch: { r: [min, max], g: [min, max], b: [min, max] },
  leaves: [                                // array of color ranges; one is chosen per dot
    { r: [min, max], g: [min, max], b: [min, max] },
    ...
  ],
  base: 'rgb(...)',                        // trunk base / ground dots
  barkStyle: 'marks'|'lenticels'|'furrows'|'flake',  // optional
  barkMarks: { r: [min, max], g: [...], b: [...] },   // required if barkStyle set
  bare: true,                             // optional — skips leaf rendering entirely (Winter Bare)
  fruit: {
    style: 'berry'|'blossom'|'cone'|'catkin'|'snow',
    colors: [ { r, g, b ranges }, ... ],
    chance: 0.0–1.0                       // per-dot probability
  }
}
```

### Pot color definition
```js
{
  name: 'Display Name',
  color: 'rgb(...)',        // wall faces
  rimColor: 'rgb(...)',     // rim row + interior fill
  accentColor: 'rgb(...)',  // bottom row + feet
  gloss: true,              // optional
  highlight: 'rgb(...)'     // optional, unused in rendering currently (reserved)
}
```

### Glow keys
Glow intensity is set per-cell via a string key, resolved to blur radii:
```
trunk_thick, trunk_medium, trunk_thin
leaf_dense, leaf_medium, leaf_sparse
fruit
pot
```

### DEFAULTS
```js
{ treeType: '0', palette: 'autumn', monoColor: '#39ff14',
  layers: 8, startLen: 15, angle: 40, leafLen: 4 }
```

---

## Animation Speed System

Speed slider: 1–100. Mapped to `(delay, batchSize)` in three zones:

| Speed | Delay | Batch | Character |
|-------|-------|-------|-----------|
| 1–40  | 50ms → 2ms | 1 | Slow crawl, every dot visible |
| 41–70 | 1ms | 1 → 4 | Medium, smooth reveal |
| 71–100 | 1ms | 4 → 12 | Fast; 100 = current max |

`AnimationQueue.run(delayMs, batchSize, onUpdate, genId)` — processes `batchSize` actions per frame, calls `onUpdate()` once per batch, then sleeps `delayMs`. The `genId` guard cancels stale runs when a new tree is requested mid-animation.

---

## Palettes (v0.30)

| Key | Name | Bark style | Fruit style |
|-----|------|-----------|-------------|
| `autumn` | Autumn | — | berry |
| `maple` | Autumn Maple | flake | berry |
| `cherry` | Cherry Blossom | lenticels | blossom |
| `willow` | Weeping Willow | — | catkin |
| `juniper` | Juniper | furrows | berry |
| `winter` | Winter Spruce | — | cone |
| `birch` | Birch | marks | catkin |
| `jacaranda` | Jacaranda | — | blossom |
| `wisteria` | Wisteria | — | blossom |
| `winterbare` | Winter (Bare) | — | snow (bare=true) |
| `sakuranight` | Sakura Night | lenticels | blossom |
| `monochrome` | Monochrome | — | monochrome tinted |

**Bark styles** — all consume exactly 1 `rng.next()` call per pixel unconditionally (the roll), plus 3 calls if the roll fires (the mark color). Palettes without bark still consume these calls via `getBarkMarkColorRaw()`, which uses a `{r:[0,0],g:[0,0],b:[0,0]}` fallback range and discards the result. This keeps RNG sequences identical across all palettes, meaning **any palette can have bark added at any time without shifting tree shapes**.
- `marks` — dark horizontal dashes (Birch); frequency scales with trunk width
- `lenticels` — lighter horizontal bands; flat probability, any width
- `furrows` — dark vertical lines; only on width ≥ 3, sparse on thin branches
- `flake` — lighter patches suggesting peeling bark; scales with width

**Fruit styles** affect glow key: `cone` and `catkin` use `leaf_medium` (subtle); `berry`, `blossom`, `snow` use `fruit` (bright).

**`bare: true`** palettes skip the `Leaves.draw()` loop entirely. If fruit/flowers is enabled, a single snow/tip dot may appear at each branch tip.

---

## Pot Colors (v0.30)

8 matte: `terracotta`, `slate`, `jade`, `ocean`, `oxblood`, `charcoal`, `cream`, `copper`  
7 gloss: `cobalt`, `vermillion`, `obsidian`, `forest`, `aubergine`, `gold`, `teal`  
Plus `random` (resolved to a random key before tree RNG starts).

Gloss pots render a 3-dot specular highlight cluster in the upper-left corner of the pot body using `rimColor` at `pot` glow intensity.

---

## UI Features

### Palette picker
Custom dropdown (not a `<select>`). The hidden `<input id="color-palette">` holds the value. Always use `this.setPalette(key)` to change the palette — it updates the hidden input, the trigger display, and the active state of dropdown rows simultaneously. Never set `color-palette.value` directly without also calling `setPalette`.

### Unlit panel toggle
The `unlit-panel` checkbox has a `change` listener that immediately re-renders the current grid without regenerating. After each generation, the completed grid is stored as `this.lastGrid` (and `this.lastGridBounds` for the animated case) so the toggle always has something to work with.

### Screensaver / auto-advance
State held in `this.screensaverTimer` and `this.screensaverCountdown`. Key methods:
- `startScreesaver()` — note the typo in the method name (single 'e' in 'Screesaver') — starts the countdown
- `stopScreensaver()` — clears both timers, hides indicator, stops focus progress bar
- `scheduleNextTree()` — sets countdown label, starts focus progress bar, fires `randomizeAll()` + `generate()` after interval
- `startFocusProgress(durationMs)` — animates the focus mode progress bar for the given duration
- `stopFocusProgress()` — removes the running animation from the progress bar

`generate()` cancels any pending screensaver timer at its start, then reschedules at its end if the checkbox is still checked.

### Focus mode
Toggled by the `⛶` button fixed in the top-right corner (always visible). Adds `focus-mode` to `<body>`, which via CSS hides the header, controls panel, and zoom controls, and switches `.canvas-container` to centered flex with `overflow: hidden`.

- Exit via the `⊠` button or `Escape` key — both restore the manual zoom transform
- `fitToViewport()` is called on enter, after each `generate()` (instant and animated paths), and on `window.resize`
- In animated mode, `fitToViewport()` is called after the pre-pass result is written to the canvas (before animation starts), so there is no snap-zoom at the end
- The focus progress bar (`#focus-progress` / `#focus-progress-bar`) is a 2px accent-colored line pinned to the bottom of the screen, visible only in focus mode while screensaver is active. It shrinks via CSS `scaleX` animation driven by `--progress-duration` custom property.

### `fitToViewport()`
Measures the `.canvas-container` element's `clientWidth/clientHeight` (not `window.innerWidth/Height`) to get the true available space. Resets the transform to `none` first to measure the tree's natural size, then applies `scale(min(vpW/treeW, vpH/treeH, 1.5))`. Deferred by one `requestAnimationFrame` when called from the focus mode toggle button, to allow CSS reflow to complete before measuring.

### Zoom
Manual zoom: `+`/`−` buttons and Ctrl/Cmd+scroll wheel. Step size is 10% (range 10%–200%). Zoom state is preserved in `this.zoomLevel` and restored when exiting focus mode.

### Keyboard shortcuts
- `Space` or `Enter` — generate a new tree (when focus is not in an input field)
- `Escape` — exit focus mode

### Export
The export modal has a PNG/SVG format toggle. PNG uses an offscreen `<canvas>` to rasterize at device pixel ratio. SVG generates vector `<circle>` elements with `<filter>` glow (feGaussianBlur + feFlood + feComposite). Both parse the same rendered `innerHTML` rows to extract cell positions and colors.

### URL sharing
`buildShareUrl()` serializes tree options to query params. `loadFromUrl()` restores them on load. Speed, instant-mode, and focus mode are intentionally excluded (playback/display preferences, not tree state). After restoring palette from URL, always call `this.setPalette(paletteVal)` to sync the custom picker UI.

---

## Known Quirks & Non-obvious Decisions

- **`#tree-display` vs `#tree-canvas`**: The `white-space: pre` that makes newlines work is on `#tree-display`. `#tree-canvas` (the innerHTML target) is a block child inside it — this works because `white-space` inherits in CSS.
- **Canvas anchored top in normal mode**: `.canvas-container` uses `align-items: flex-start` with `padding-top: 2rem`. Focus mode overrides this to `align-items: center` / `padding-top: 0` via the `body.focus-mode` CSS rule.
- **Pre-pass for bounds + focus fit**: In animated mode, `generate()` runs a second instant-mode tree draw on a throwaway grid to capture `finalBounds`. That result is immediately written to `canvas.innerHTML` so `fitToViewport()` can measure real DOM dimensions. The animation then overwrites cells from the same stable bounds — dimensions never change mid-animation.
- **Monochrome palette**: The palette key is `'monochrome'` but it doesn't exist in `COLOR_PALETTES`. It's handled as a special case in `getOptions()` / the rendering path, using the `monoColor` hex value from the color picker to tint a generated palette at runtime.
- **`randomizeAll()` and `applyDefaults()`** must call `this.setPalette(key)` not `document.getElementById('color-palette').value = key` — the latter skips the picker UI update.
- **`randomizeAll()` clears the seed field** — `getOptions()` then generates a fresh random seed and writes it back to the input. This is intentional so every screensaver cycle and every 🎰 press produces a genuinely new tree.
- **`startScreesaver()` typo** — single `e` in method name. Don't fix without updating all call sites.
- **Focus progress bar animation restart** — `void bar.offsetWidth` forces a reflow between removing and re-adding the `running` class, which is required for the CSS animation to restart from the beginning each cycle.
- **`fitToViewport()` measures `.canvas-container`**, not `window.innerWidth/Height`. The container's `clientWidth/clientHeight` reflects the actual layout space after the sidebar and header are hidden, whereas `window` dimensions include browser chrome and can disagree with layout at non-100% OS display scales.

---

## Version History (sessions)

| Versions | Work |
|----------|------|
| v0.1–v0.10 | Initial build: LED panel aesthetic, 10 palettes, 4 tree types, URL sharing, PNG export, monochrome mode, zoom, pot colors |
| v0.11 | SVG favicon |
| v0.12 | Bug fixes: animation speed reset, pot color RNG shift, clipboard fallback |
| v0.13 | Filled pot interior, speed increase, Birch RNG fix |
| v0.14–v0.15 | Gloss pot colors with specular highlights |
| v0.16 | Phase 1: bark variation (4 styles), palette-specific fruit tuning (5 styles), Winter (Bare) palette, Sakura Night palette |
| v0.17 | Winter (Bare) branch colors fixed to match Cherry Blossom |
| v0.18 | Phase 2: palette preview swatches (custom dropdown), screensaver/auto-advance mode |
| v0.19 | Screensaver interval default 60s, max 300s |
| v0.20 | Fixed canvas jumping during animation (pre-pass bounds capture) |
| v0.21 | Canvas anchored to top; animation speed batching |
| v0.22 | Speed slider fully functional across range (unified delay+batch mapping) |
| v0.23 | Phase 3: SVG export added to export modal |
| v0.24 | Bug fixes: screensaver now randomizes seed; unlit panel toggle re-renders immediately without regenerating; `lastGrid`/`lastGridBounds` stored on instance |
| v0.25 | Version bump only |
| v0.26 | Zoom step reduced to 10% (min 10%); focus mode added (hides header/controls/zoom, `⛶`/`⊠` toggle button, `Escape` to exit) |
| v0.27 | Version bump only |
| v0.28 | Focus mode auto-fits tree to viewport via `fitToViewport()`; pre-pass result seeded into canvas before animation starts to eliminate end-of-animation snap-zoom |
| v0.29 | `fitToViewport()` measures `.canvas-container` instead of `window` for correct available space; `requestAnimationFrame` defer on focus toggle and pre-pass seed for reflow timing |
| v0.30 | Screensaver progress bar in focus mode (shrinking accent line at bottom); Space bar shortcut to generate; screensaver/🎰 RNG discipline confirmed consistent |

---

## Possible Future Work

These were discussed and deferred:

- **Panel size presets** — 32×16, 64×32, 128×64 dot grid sizes
- **Animated wind** — sine wave sway on branches
- **Ground moss/roots** — horizontal LED lines at trunk base
- **Multiple trees** — forest planting mode
- **Undo/redo** — history stack for settings changes
- **Palette preview thumbnails** — small pre-rendered tree swatches (heavier than dot swatches)
- **Branch taper** — visible width reduction toward tips (currently width steps by integer)
- **Leaf cluster shapes** — weeping, globe, flat-top variants
- **More bark styles** — Cherry lenticels already done; could add Pine, Oak furrows; RNG budget is pre-reserved so any palette can gain bark without shifting tree shapes
- **Additional palettes** — Halloween/Dead (bare gray-black, orange/purple), Tropical (bright saturated)
- **Stop button `lastGrid` fix** — stopping mid-animation leaves `lastGrid` stale; unlit toggle won't work until next full generation
