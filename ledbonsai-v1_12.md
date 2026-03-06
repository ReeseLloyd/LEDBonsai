# LEDBonsai — Documentation
**Version 1.12**

LEDBonsai is a single-file, self-contained HTML application that generates procedural bonsai trees rendered as glowing LED dot-matrix displays. Trees are deterministic — any tree can be exactly reproduced from its seed and settings. No server, no dependencies, no installation required.

---

## Getting Started

Open `ledbonsai-v1_12.html` in any modern web browser. A tree generates automatically on load. Click **Start** to make a new one, or **Randomize** (🎲) to shuffle the seed and generate, or **🎰** to randomize all settings (tree type, palette, structure sliders, and seed) before generating.

---

## Controls

### Tree Settings

| Control | Range | Default | Description |
|---|---|---|---|
| **Seed** | Any integer | Random | Determines the random sequence for the tree. The same seed always produces the same tree given identical settings. Leave blank to use a random seed each time. |
| **Tree Type** | — | Classic | Growth algorithm (see [Tree Types](#tree-types)). |
| **Color Palette** | — | Autumn | Visual color scheme for branches and foliage (see [Palettes](#color-palettes)). |
| **Layers** | 3 – 15 | 8 | Number of branching iterations. Higher values produce larger, denser trees. Very high values with certain settings are automatically clamped to protect memory (see [Memory Safety](#memory-safety)). |
| **Branch Length** | 5 – 30 | 15 | Starting length of the trunk and primary branches in grid cells. |
| **Branch Angle** | 15° – 75° | 40° | Mean angle of branch divergence from the parent. Higher values produce wider, more spreading trees. |
| **Leaf Length** | 1 – 10 | 4 | Length of terminal leaf clusters. |

### Appearance

| Control | Description |
|---|---|
| **Glow intensity** | Toggles the CSS text-shadow glow effect on all lit LEDs. |
| **Show fruit** | Toggles fruit / flower clusters (berries, blossoms, cones, catkins, or snow depending on palette). |
| **Unlit panel** | Shows dimly visible unlit LED dots in the background, simulating a real LED matrix panel. |

### Pot

| Control | Range | Default | Description |
|---|---|---|---|
| **Pot Color** | — | Random | Color of the decorative pot. Matte and gloss finish options available. |
| **Pot Size** | 15 – 35 | 25 | Width of the pot in grid cells. |

### Animation

| Control | Range | Default | Description |
|---|---|---|---|
| **Animation Speed** | 1 – 100 | 50 | Controls how quickly dots appear during tree generation. Lower is slower; higher is faster. |

### Auto-Advance (Screensaver)

When enabled, LEDBonsai automatically generates a new random tree at a set interval — useful for running as an ambient display.

| Control | Range | Default | Description |
|---|---|---|---|
| **Auto-advance** | On/Off | Off | Enables screensaver mode. |
| **Interval** | 5 – 300 s | 60 s | Time in seconds between each new tree. |

---

## Tree Types

Six tree types are available, each implementing a distinct growth algorithm.

| Type | Description |
|---|---|
| **Classic** | Symmetric binary branching with slight random variation. Produces well-balanced, traditional bonsai silhouettes. |
| **Fibonacci** | Branch counts follow the Fibonacci sequence at each layer. Produces asymmetric, naturally irregular trees that tend to lean. |
| **Offset Fibonacci** | Fibonacci branching with additional positional offset along parent branches, creating more organic-feeling growth. |
| **Random Fibonacci** | Fibonacci branching with fully randomized growth-point positions along each branch rather than strictly at the tips. Produces the most naturalistic and unpredictable results. |
| **Literati** | Inspired by the formal Japanese *bunjin* (literati) bonsai style. A thin trunk meanders dramatically with each layer. Branching is suppressed in the lower half of the tree and sparse near the apex, concentrating foliage into a few expressive clusters at the top. Looks painterly and minimal compared to the denser types. |
| **Cascade** | Inspired by the *kengai* (cascade) bonsai style. The pot is positioned near the top of the composition; the trunk and branches lean strongly to one side and arc downward, with foliage falling below the pot rim. A short upward "back branch" on the opposite side appears in most seeds, providing the compositional balance characteristic of authentic Kengai style. The cascade direction (left or right) and the height at which the trunk emerges from the pot are both randomized per seed. |

All six types are included in the **🎰 Randomize All** rotation.

### Architecture note

Tree types are implemented as subclasses of a common `Tree` base class. Each type can optionally declare a `rootYOffset()` to request additional vertical canvas space above the pot — used by Cascade to ensure room for the falling branches below. Adding new types only requires a new class and a one-line entry in the `TREE_CLASSES` array.

---

## Color Palettes

All 18 palettes are included in the **🎰 Randomize All** rotation.

| Palette | Character |
|---|---|
| **Autumn** | Warm brown bark, green foliage with red berries |
| **Autumn Maple** | Rich copper bark with flaking texture; vivid red, orange, and gold leaves |
| **Birch** | Pale white bark with dark horizontal marks; fresh green foliage |
| **Bioluminescent** | Near-black trunk, deep teal to electric cyan leaves; glows intensely |
| **Cherry Blossom** | Warm brown bark; pink-to-white blossoms in dense clusters |
| **Dusk** | Amber trunk; foliage sweeping from deep violet through coral to golden orange |
| **Ember** | Near-black trunk; leaves progressing from smoldering dark red through orange to bright yellow tips |
| **Fog** | Fully desaturated cool grays; minimal and contemplative |
| **Jacaranda** | Mid-brown bark; lavender to violet foliage with green accents |
| **Juniper** | Rough furrowed bark; deep forest green foliage with blue-gray berries |
| **Midnight** | Deep navy-indigo trunk; foliage spanning dark teal through blue to icy near-white tips |
| **Sakura Night** | Dark bark; saturated hot-pink blossoms |
| **Spring** | Warm tan bark; fresh lime-green to yellow-white new growth with white blossoms |
| **Weeping Willow** | Golden-tan bark; soft yellow-green foliage with catkins |
| **Winter (Bare)** | Dark bark, no foliage; heavy snow accumulation on branches |
| **Winter Spruce** | Dark bark; deep green needles tipped with snow-white highlights and cones |
| **Wisteria** | Warm brown bark; mixed green foliage and cascading lavender-purple blossoms |
| **Monochrome** | Single-color mode. Eight presets: Phosphor Green, Amber, Ice Blue, Cyan, Hot Pink, Red, White, Violet. Pick any custom color with the color picker. |

---

## Sharing & URLs

After each tree is generated, a shareable URL appears in the **Share** field. This URL encodes all current settings as query parameters. Anyone opening the URL will see the exact same tree.

Click the 📋 button to copy the URL to the clipboard.

**URL parameters:**

| Parameter | Description |
|---|---|
| `seed` | Tree seed integer |
| `type` | Tree type (0–5) |
| `layers` | Layer count |
| `len` | Branch length |
| `angle` | Branch angle |
| `leaf` | Leaf length |
| `palette` | Palette key (e.g. `dusk`, `ember`) |
| `mono` | Monochrome hex color (URL-encoded) |
| `potcolor` | Pot color key |
| `potsize` | Pot size |
| `glow` | Glow on/off (`1`/`0`) |
| `fruit` | Fruit on/off (`1`/`0`) |
| `unlit` | Unlit panel on/off (`1`/`0`) |
| `screensaver` | Auto-advance on/off (`1`/`0`) |
| `ssinterval` | Screensaver interval in seconds |
| `focus` | Focus mode on/off (`1`/`0`) |

---

## Pinned Trees

The pin system lets you save trees you want to return to, stored locally in the browser.

- **☆ Pin Tree** (sidebar) — pins the currently displayed tree.
- **📌 Pins** (sidebar) — opens the pins drawer.
- **☆** button (focus mode, top-right) — pins the current tree without leaving focus mode; flashes gold to confirm.

The **pins drawer** shows all saved trees with their palette swatches, label, seed number, and save date. From the drawer you can **Load** any pin (restores all settings and regenerates) or **🗑** delete it.

Pins are stored in `localStorage` under the key `ledbonsai_pins`, capped at 50 entries. They persist across page reloads and browser restarts. See [Storage Notes](#storage-notes) for platform-specific details.

The drawer can be closed by clicking the ✕ button, clicking the overlay behind it, or swiping right on touch devices.

---

## Focus Mode

Focus mode hides the control sidebar and header, filling the entire screen with the tree. The tree is automatically scaled to fit the viewport using a `translate + scale` transform pinned to the top-left corner, ensuring accurate centering at any tree size or aspect ratio.

- **Enter/exit**: click the **⛶** button (top-right corner) or press **Escape** to exit.
- **In focus mode**, four buttons appear in the top-right corner:
  - **⏭** — skip to next tree (randomize and generate immediately)
  - **☆** — pin the current tree
  - **⏸ / ▶** — pause/resume
  - **⊠** — exit focus mode

Focus mode also activates the **screensaver progress bar** — a thin glowing line along the bottom of the screen that depletes over the interval duration, showing time remaining until the next tree.

---

## Pause / Resume

The **⏸** button (visible in focus mode) pauses the current activity:

- **During animation** — freezes the dot-by-dot reveal mid-draw. Resume continues exactly where it left off.
- **During screensaver countdown** — freezes the countdown timer and the progress bar. Resume restores the remaining duration and picks up precisely where the bar stopped.

The button glows green while paused. **Spacebar** also toggles pause when in focus mode.

---

## Zoom

The **−**, **+**, and **⟲** buttons (top-right of the display area, outside focus mode) adjust the zoom level from 10% to 200%. **Ctrl/Cmd + scroll wheel** also zooms.

---

## Export

Click **Export…** to save the current tree as a file.

| Format | Notes |
|---|---|
| **PNG** | Raster image. Choose from three background options: Panel (dark gray dot grid), Pure black, or Transparent. |
| **SVG** | Scalable vector file. Crisp at any size; preserves glow effects. Background is transparent. |

---

## Keyboard Shortcuts

| Key | Action |
|---|---|
| **Space** | Generate new tree (normal mode) · Toggle pause (focus mode) |
| **Enter** | Generate new tree (when not in a text input) |
| **N** | Skip to next tree — randomize and generate immediately (focus mode only) |
| **Escape** | Exit focus mode |
| **Ctrl/Cmd + scroll** | Zoom in/out |

---

## Memory Safety

Very large trees can generate substantial HTML strings during animation. LEDBonsai automatically estimates the render cost of the requested parameters before generating. If the estimated memory usage would exceed approximately 1 GB of total allocated strings, the **Layers** value is silently clamped downward until the estimate is safe. The Layers slider updates to reflect the clamped value, and a warning is logged to the browser console.

This most commonly affects Fibonacci trees at high layer counts with the Unlit Panel option enabled.

---

## Storage Notes

Pins are stored in `localStorage`, which behaves differently across platforms:

| Platform | Persistence |
|---|---|
| **Raspberry Pi (Chromium)** | Essentially permanent until browser data is manually cleared. Ideal for kiosk use. |
| **Desktop (Chrome / Firefox)** | Permanent until manually cleared. Very reliable. |
| **iPad / iPhone (Safari)** | Safari may clear `localStorage` for sites not visited in 7 days. To prevent this, **add the page to your Home Screen** as a web app — this gives it its own persistent storage context not subject to the 7-day purge. |
| **Private / Incognito** | No persistence — pins are lost when the window closes. |

---

## Running as a Display (Raspberry Pi / iPad)

LEDBonsai is designed to work well as an always-on ambient display.

**Recommended setup:**
1. Open the file in the browser.
2. Enable **Auto-advance** and set your preferred interval.
3. Enter **Focus mode** (⛶).
4. On Raspberry Pi, launch Chromium in kiosk mode to hide the browser UI:
   ```
   chromium-browser --kiosk file:///path/to/ledbonsai-v1_12.html
   ```
5. On iPad, add to Home Screen for full-screen display and persistent pin storage.

The **⏸** button lets you freeze on a tree you like without leaving focus mode. **☆** pins it for later.

---

## Version History

| Version | Changes |
|---|---|
| **1.12** | Fixed focus mode fit-to-viewport: tree now uses `position: absolute` pinned to top-left with `translate + scale` transform, correctly centering at all sizes |
| **1.11** | Focus mode fit-to-viewport: switched to `transform-origin: top left` with explicit translate centering; use `window.innerHeight` instead of container height |
| **1.10** | Focus mode fit-to-viewport: use `window.innerWidth/Height` instead of `container.clientHeight` which was expanding to match tree size |
| **1.09** | Focus mode fit-to-viewport: subtract element padding from size measurement before computing scale |
| **1.08** | Monopodial tree type commented out pending further development; Literati renumbered to type 4, Cascade to type 5 |
| **1.07** | Monopodial: rewrote lateral placement as distributed whorls along trunk segments; lateral length anchored to `startLen` with floor |
| **1.06** | Cascade: direction-aware leaf clustering (`drawLeavesAtTip`) with 6 particles and gravity aligned to branch direction; Monopodial: increased `LATERAL_SCALE` and `MAX_LATERAL_DEPTH` |
| **1.05** | Cascade: draw vertical trunk segment between pot rim and cascade origin when `trunkRise > 0` |
| **1.04** | Cascade: randomized trunk emergence height (0–5 px above pot rim) per seed |
| **1.03** | Added Cascade (Kengai) tree type; `rootYOffset()` infrastructure on Tree base class for types that need vertical canvas space above the pot; refactored tree instantiation from switch statements to `TREE_CLASSES` array |
| **1.02** | Removed Weeping tree type |
| **1.01** | Added Monopodial, Weeping, and Literati tree types |
| **0.40** | Added Cyan and Hot Pink monochrome presets (8 total) |
| **0.39** | Added ⏭ skip/next button and **N** key shortcut in focus mode |
| **0.38** | Fixed palette name capitalization in selector display |
| **0.37** | Added 6 new palettes (Bioluminescent, Fog, Spring, Midnight, Dusk, Ember); alphabetized palette picker |
| **0.35** | Added pin system with persistent localStorage storage and slide-out drawer |
| **0.34** | Added pause/resume button for animation and screensaver countdown; Space key shortcut |
| **0.33** | Added version label to sidebar |
| **0.32** | Memory safety: automatic layer clamping to prevent OOM on 512 MB systems |
