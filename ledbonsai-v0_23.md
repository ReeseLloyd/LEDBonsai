# LEDBonsai — Developer Handoff
**Current version:** v0.23  
**File:** `ledbonsai-v0_23.html`  
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
LEDBonsai             — Main app: UI binding, generate(), export, screensaver
```

### Rendering pipeline

1. `generate()` calls `animQueue.reset()`, reads all options from the DOM
2. A `Tree` subclass is instantiated with the grid and animQueue
3. `tree.draw()` runs **synchronously** — branch cells call `setLEDAnimated()` which either writes directly to the grid (instant mode) or enqueues a closure (animated mode)
4. In animated mode, a **pre-pass** runs `tree.draw()` a second time in instant mode on a throwaway grid/queue to capture the final bounding box (`finalBounds`), so the canvas never jumps during animation
5. `animQueue.run(delay, batchSize, onUpdate, genId)` drains the queue, calling `onUpdate` (which re-renders `grid.render()` to `innerHTML`) after each batch

### Critical: RNG discipline
Every visual decision must flow through the **seeded RNG** (`this.rng`) so trees are reproducible from their seed. The main pitfalls:
- **Pot color "Random"** is resolved before the seeded RNG is created (uses `Math.random()`), so choosing "Random" vs a specific color doesn't shift the tree's sequence
- **Birch bark marks** historically caused RNG drift; the fix was collapsing to a single `rng.next()` call per pixel using a combined probability. All other bark styles follow the same one-call-per-pixel rule in `getBarkMarkChance()`

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

## Palettes (v0.23)

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

**Bark styles** (all use a single `rng.next()` call per pixel):
- `marks` — dark horizontal dashes (Birch); frequency scales with trunk width
- `lenticels` — lighter horizontal bands; flat probability, any width
- `furrows` — dark vertical lines; only on width ≥ 3, sparse on thin branches
- `flake` — lighter patches suggesting peeling bark; scales with width

**Fruit styles** affect glow key: `cone` and `catkin` use `leaf_medium` (subtle); `berry`, `blossom`, `snow` use `fruit` (bright).

**`bare: true`** palettes skip the `Leaves.draw()` loop entirely. If fruit/flowers is enabled, a single snow/tip dot may appear at each branch tip.

---

## Pot Colors (v0.23)

8 matte: `terracotta`, `slate`, `jade`, `ocean`, `oxblood`, `charcoal`, `cream`, `copper`  
7 gloss: `cobalt`, `vermillion`, `obsidian`, `forest`, `aubergine`, `gold`, `teal`  
Plus `random` (resolved to a random key before tree RNG starts).

Gloss pots render a 3-dot specular highlight cluster in the upper-left corner of the pot body using `rimColor` at `pot` glow intensity.

---

## UI Features

### Palette picker
Custom dropdown (not a `<select>`). The hidden `<input id="color-palette">` holds the value. Always use `this.setPalette(key)` to change the palette — it updates the hidden input, the trigger display, and the active state of dropdown rows simultaneously. Never set `color-palette.value` directly without also calling `setPalette`.

### Screensaver / auto-advance
State held in `this.screensaverTimer` and `this.screensaverCountdown`. Key methods:
- `startScreesaver()` — note the typo in the method name (single 'e' in 'Screesaver') — starts the countdown
- `stopScreensaver()` — clears both timers, hides indicator
- `scheduleNextTree()` — sets countdown label + fires `randomizeAll()` + `generate()` after interval

`generate()` cancels any pending screensaver timer at its start, then reschedules at its end if the checkbox is still checked.

### Export
The export modal has a PNG/SVG format toggle. PNG uses an offscreen `<canvas>` to rasterize at device pixel ratio. SVG generates vector `<circle>` elements with `<filter>` glow (feGaussianBlur + feFlood + feComposite). Both parse the same rendered `innerHTML` rows to extract cell positions and colors.

### URL sharing
`buildShareUrl()` serializes tree options to query params. `loadFromUrl()` restores them on load. Speed and instant-mode are intentionally excluded (playback preferences, not tree state). After restoring palette from URL, always call `this.setPalette(paletteVal)` to sync the custom picker UI.

---

## Known Quirks & Non-obvious Decisions

- **`#tree-display` vs `#tree-canvas`**: The `white-space: pre` that makes newlines work is on `#tree-display`. `#tree-canvas` (the innerHTML target) is a block child inside it — this works because `white-space` inherits in CSS.
- **Canvas anchored top**: `.canvas-container` uses `align-items: flex-start` with `padding-top: 2rem`. Changing to `center` will push the pot below the viewport on tall trees.
- **Pre-pass for bounds**: In animated mode, `generate()` runs a second instant-mode tree draw on a throwaway grid to capture `finalBounds` before animation starts. This is necessary because animated cells are deferred closures — the real grid is nearly empty when `tree.draw()` returns.
- **Monochrome palette**: The palette key is `'monochrome'` but it doesn't exist in `COLOR_PALETTES`. It's handled as a special case in `getOptions()` / the rendering path, using the `monoColor` hex value from the color picker to tint a generated palette at runtime.
- **`randomizeAll()` and `applyDefaults()`** must call `this.setPalette(key)` not `document.getElementById('color-palette').value = key` — the latter skips the picker UI update.

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
- **More bark styles** — Cherry lenticels already done; could add Pine, Oak furrows
- **Additional palettes** — Halloween/Dead (bare gray-black, orange/purple), Tropical (bright saturated)
