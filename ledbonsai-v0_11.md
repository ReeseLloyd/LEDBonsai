# LEDBonsai Project Documentation

## Overview

LEDBonsai is a single-file web application that procedurally generates bonsai trees rendered as glowing LED dot-matrix displays. It is a fork of **JSBonsai** (itself a JavaScript port of [PyBonsai](https://github.com/Ben-Edwards44/PyBonsai)), reimagined as an LED panel aesthetic rather than traditional ASCII art.

Every character in the tree is rendered as a filled circle (`●`, U+25CF), with CSS `text-shadow` glow effects to simulate luminous LEDs. The canvas background uses a dark gray with a subtle radial dot grid to evoke a physical LED panel.

**Current version:** v0.10  
**File naming convention:** `ledbonsai-ClaudeOpus-YYYYMMDD-v{version}.html`  
**Format:** Single self-contained HTML file (no external dependencies)

---

## Architecture

### Class Structure

```
SeededRandom          - Mulberry32 PRNG (deterministic, seed-based)
Grid                  - Sparse infinite LED buffer (Map-based, stores color + glowKey per cell)
Vector                - 2D vector math
AnimationQueue        - Async animation with generation ID for race condition prevention
Tree (base)           - Pot rendering, branch drawing, LED helpers
├── ClassicTree       - Random branch count per layer (normal distribution)
├── FibonacciTree     - Fibonacci branch count per layer
│   ├── OffsetFibTree - Branches grow from midpoints along parent
│   └── RandomOffsetFibTree - Random placement along parent
Leaves                - Particle system; gravity-influenced leaf/foliage clusters
LEDBonsai             - Main application controller
```

### Key Design Decisions vs. JSBonsai

| Concern | JSBonsai | LEDBonsai |
|---|---|---|
| Characters | Varied ASCII/Unicode per element type | Always `●` (U+25CF) |
| Depth/texture | Character variety (block chars, bark chars, etc.) | Glow intensity via CSS `text-shadow` |
| Grid cells | `{ char, color, bold, bgColor }` | `{ color, glowKey, isUnlit }` |
| Rendering | HTML spans with varied chars | HTML spans all with `●`, differing shadow |
| Pot style | 5 styles with box-drawing chars | Single style — filled LED dots |
| Trunk texture | Character-based (thick/medium/thin char sets) | Glow-key-based (`trunk_thick/medium/thin`) |
| Foliage gradient | Dense/medium/sparse char sets | Dense/medium/sparse glow keys |

### Glow System

All lit LEDs carry a `glowKey` that maps to a two-layer CSS `text-shadow` — a tight inner glow and a wider diffuse outer glow:

```javascript
const GLOW = {
    trunk_thick:  { blur1: 4,  blur2: 12, alpha2: 0.7 },
    trunk_medium: { blur1: 3,  blur2: 8,  alpha2: 0.5 },
    trunk_thin:   { blur1: 2,  blur2: 6,  alpha2: 0.4 },
    leaf_dense:   { blur1: 5,  blur2: 16, alpha2: 0.8 },
    leaf_medium:  { blur1: 4,  blur2: 10, alpha2: 0.6 },
    leaf_sparse:  { blur1: 2,  blur2: 6,  alpha2: 0.3 },
    fruit:        { blur1: 6,  blur2: 20, alpha2: 1.0 },
    pot:          { blur1: 3,  blur2: 8,  alpha2: 0.5 },
};
```

### Monochrome Palette

Unlike the standard palettes (which have fixed RGB ranges), the Monochrome palette is built dynamically from a user-chosen base color via `buildMonoPalette(hex)`. Brightness varies by element role:

| Element | Brightness |
|---|---|
| Trunk / branches | 35% |
| Sparse leaf edges | 40% |
| Medium foliage | 75% |
| Dense foliage | 100% |
| Tree base accent | 25% |

---

## Current Features (v0.10)

### Tree Generation
- **4 tree types:** Classic, Fibonacci, Offset Fibonacci, Random Fibonacci
- **Seeded RNG** (Mulberry32) for fully reproducible trees
- **Shareable URLs** encode all parameters

### Color Palettes (10 total)

| Key | Name | Character |
|---|---|---|
| `autumn` | Autumn | Green foliage, gray-brown trunk |
| `maple` | Autumn Maple | Red/orange/yellow foliage |
| `cherry` | Cherry Blossom | Pink tones + green, dark brown trunk |
| `willow` | Weeping Willow | Yellow-greens, tan trunk |
| `juniper` | Juniper | Silvery green needles, blue-green berries |
| `winter` | Winter Spruce | Dark green + snow white + ice blue |
| `birch` | Birch | White/cream trunk with dark marks, bright green leaves |
| `jacaranda` | Jacaranda | Lavender/purple blooms, gray-brown trunk |
| `wisteria` | Wisteria | Cascading purples + green, twisted gray trunk |
| `monochrome` | Monochrome | Single hue, brightness-gradient; 6 presets + custom picker |

### Pot System
- LED-dot rendered (all `●` characters, no box-drawing chars)
- **8 pot colors:** Terracotta, Slate, Jade, Ocean Blue, Oxblood, Charcoal, Cream, Copper
- Plus Random option
- Pot size slider (15–35)

### LED Effects (toggleable)
- **Vary glow by density** — trunk width and foliage distance map to glow intensity keys
- **Fruit / flowers (accent LEDs)** — palette-specific accent dots rendered with `fruit` glow key (brightest)
- **Show unlit panel dots** — toggle; currently renders as spaces (visually off, preserves grid spacing)

### Controls Panel (top to bottom)

1. **Seed** + 🎲 randomize seed button
2. **Start / Stop / 🎰 Randomize All / ↺ Defaults** button row
3. Tree Type selector
4. Color Palette selector (+ Monochrome sub-panel when selected)
5. **Structure** — Layers, Branch Length, Branch Angle, Leaf Length sliders
6. **Pot** — Pot Color, Pot Size
7. **LED Effects** — three checkboxes
8. **Animation** — Instant toggle, Speed slider
9. **Share** — URL field + 📋 copy button
10. **Export** — Export PNG… button
11. Credit

### Randomize All (🎰)
Randomizes tree type, palette (including random monochrome preset), and all four structure sliders within curated ranges. Does not touch pot, LED effects, or animation settings.

### Defaults (↺)
Restores fixed hardcoded defaults:

```javascript
const DEFAULTS = {
    treeType:  '0',       // Classic
    palette:   'autumn',
    monoColor: '#39ff14',
    layers:    8,
    startLen:  15,
    angle:     40,
    leafLen:   4,
};
```

### PNG Export
Opens a modal with three background options:
- **Panel** — dark gray `#131316` + radial dot grid (matches on-screen look)
- **Pure black** — `#000000`
- **Transparent** — no background fill

Renders directly to an offscreen `<canvas>` by parsing the tree's rendered HTML, extracting each dot's color and glow shadow, and redrawing character-by-character. No SVG/foreignObject; no cross-origin issues. Output is `ledbonsai-{seed}.png` at devicePixelRatio resolution.

### Animation
- Character-by-character draw animation
- Speed slider 1–100 (display normalized); internally maps to `Math.max(1, Math.round((101 - speed) / 24))`ms delay
- Instant mode toggle
- Stop button; generation ID system prevents race conditions on rapid regeneration

### Zoom
- +/− buttons (25% steps), reset, Ctrl/Cmd+scroll wheel
- Range: 25%–200%

### Sharing
- All parameters encoded in URL query string
- Parameters: `seed`, `type`, `layers`, `len`, `angle`, `leaf`, `speed`, `instant`, `palette`, `mono`, `potcolor`, `potsize`, `glow`, `fruit`, `unlit`

---

## URL Parameters

| Param | Description |
|---|---|
| `seed` | Integer RNG seed |
| `type` | Tree type (0–3) |
| `layers` | Branch layers (3–15) |
| `len` | Starting branch length (5–30) |
| `angle` | Branch angle in degrees (15–75) |
| `leaf` | Leaf length (1–10) |
| `speed` | Animation speed (1–100) |
| `instant` | Instant mode (0/1) |
| `palette` | Palette key (e.g. `autumn`, `monochrome`) |
| `mono` | Monochrome hex color (URL-encoded, e.g. `%2339ff14`) |
| `potcolor` | Pot color key or `random` |
| `potsize` | Pot width (15–35) |
| `glow` | Vary glow intensity (0/1) |
| `fruit` | Show accent LEDs (0/1) |
| `unlit` | Show unlit panel dots (0/1) |

---

## Version History

| Version | Changes |
|---|---|
| v0.1 | Initial fork from JSBonsai. All chars → `●`. Glow system. Unlit panel dots. LED-themed UI. Square-ish pixel spacing. Dark panel background with dot grid texture. |
| v0.2 | Monochrome palette (10 presets + custom picker). PNG export with background choice (panel / black / transparent). |
| v0.3 | Monochrome brightness range widened (trunk 35%, sparse 40%). Animation speed doubled (divisor /12 → /24). |
| v0.4 | Version rev only. |
| v0.5 | Unlit dots changed to spaces (invisible; removes visual clutter in PNG exports). |
| v0.6 | Juniper needles skewed greener (silvery green rather than silvery blue). |
| v0.7 | Favicon embedded as base64 PNG in `<head>` — glowing green LED dot. |
| v0.8 | Randomize All (🎲→🎰) and Defaults (↺) buttons added alongside Start/Stop. `DEFAULTS` constant. |
| v0.9 | Randomize All icon changed to 🎰. Share URL section moved to bottom of controls panel. |
| v0.10 | "Generate" button renamed "Start". |

---

## Future Enhancement Ideas

- **Falling leaves animation** — lit dots drifting downward and fading after generation completes
- **Scan-line / power-on animation** — panel sweeps a bright horizontal line as LEDs light up
- **Brightness/contrast slider** — global glow intensity multiplier
- **New LED-native palettes** — Neon (hot pink + cyan), Phosphor (green monochrome CRT), Amber (warm orange), Blueprint (icy blue)
- **Panel flicker effect** — occasional random LEDs briefly dim, simulating aging hardware
- **Panel size control** — fixed grid size with tree scaled to fit
- **Remove "Show unlit panel dots" checkbox** — currently a no-op since unlit dots are spaces; either restore the feature properly or remove the toggle
- **Pot style variety** — simple LED-native pot shapes (rectangular block, tapered trapezoid)
- **Soil/moss** — optional re-enabled feature at pot rim

---

## Credits

- Inspired by [PyBonsai](https://github.com/Ben-Edwards44/PyBonsai) by Ben Edwards
- Forked from JSBonsai
