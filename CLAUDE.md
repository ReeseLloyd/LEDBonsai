# LEDBonsai Project

## Project Location
`~/Library/CloudStorage/Dropbox/05-Projects/Programming/LEDBonsai/`

## What This Is
LEDBonsai is a single-file, self-contained HTML application that generates procedural bonsai trees rendered as glowing LED dot-matrix displays. Trees are deterministic — any tree can be exactly reproduced from its seed and settings. No server, no build step, no dependencies, no installation required.

It is designed to run on desktop browsers, mobile (iPad/iPhone), and Raspberry Pi (Chromium kiosk mode).

---

## File Naming Convention
Files follow the pattern `ledbonsai-vMAJOR_MINOR.html` and `ledbonsai-vMAJOR_MINOR.md`.
- The `.html` file is the working application code
- The `.md` file contains documentation and version history for that version
- The current latest version is the highest version number present (e.g., `v1_20`)

**Always read both the latest `.html` and `.md` before making any changes.**

---

## Versioning Rules
- **Every change — including minor bug fixes — gets a new version number**
- Increment the minor version for all changes (e.g., `v1_20` → `v1_21`)
- Increment the major version only for significant structural rewrites
- **Never overwrite a previous version file** — always create a new file
- Add an entry to the version history table in the new `.md` file describing what changed
- Version history entries should be concise (one line), e.g.:
  `| **1.21** | Fixed screensaver timer not resetting after manual generation |`

---

## Tech Stack Constraints
- **Vanilla HTML, CSS, and JavaScript only** — no frameworks, no libraries, no CDNs
- Everything must live in a single `.html` file — do not split into multiple files
- No npm, no build process, no transpilation
- No external dependencies of any kind (not even Google Fonts or icon libraries)
- Must work when opened directly as a local file (`file://`) with no server

---

## Architecture

### Tree System
- `Tree` is the base class. All tree types extend it.
- Tree types are registered in the `TREE_CLASSES` array — adding a new type only requires a new subclass and one entry in that array.
- Each subclass can optionally override `rootYOffset()` to request extra vertical canvas space (used by Cascade).
- The six current types: Classic, Fibonacci, Offset Fibonacci, Random Fibonacci, Literati, Cascade.

### Palette System
- Palettes are defined as entries in a `PALETTES` object (or similar structure).
- Each palette defines bark colors, leaf colors, fruit/blossom colors, and optionally a `bare` flag (no foliage).
- The Monochrome palette is a special case with a color picker and presets.

### Seeded Randomness
- All randomness uses a seeded PRNG so the same seed + settings always produce the same tree.
- **Never use `Math.random()` directly inside tree generation code** — always use the seeded RNG.
- Breaking determinism is a critical bug.

### URL Parameter System
- After generation, all current settings are encoded as URL query parameters for sharing.
- **Every new user-facing setting must also be added as a URL parameter** so shared links remain complete.
- Existing parameters: `seed`, `type`, `layers`, `len`, `angle`, `leaf`, `palette`, `mono`, `potcolor`, `potsize`, `glow`, `fruit`, `unlit`, `screensaver`, `ssinterval`, `focus`, `canbg`.

### Pin System
- Pins are saved to `localStorage` under the key `ledbonsai_pins`, capped at 50 entries.
- Pins store all settings needed to fully restore a tree.

### Memory Safety
- Before generating, the app estimates render cost and clamps `Layers` downward if it would exceed ~1 GB of allocated strings.
- Be cautious when adding features that increase per-cell output size.

---

## Design Principles
1. **Determinism is sacred** — same seed + settings must always produce the identical tree.
2. **Self-contained** — the file must work with no internet connection and no server.
3. **No regressions** — changes must not break existing seeds, share URLs, or saved pins.
4. **Cross-platform** — test mentally against desktop Chrome/Firefox, iPad Safari, and Raspberry Pi Chromium.
5. **Memory awareness** — Fibonacci trees at high layer counts with Unlit Panel are the stress case.
6. **Minimal UI chrome** — the tree is the focus; controls are secondary.

---

## How to Add Common Things

### New Tree Type
1. Write a new class that extends `Tree`
2. Implement `generate()` and any required helper methods
3. Override `rootYOffset()` if the type needs extra vertical space
4. Add one entry to the `TREE_CLASSES` array
5. Add it to the **🎰 Randomize All** rotation if appropriate
6. Document it in the `.md` under Tree Types

### New Color Palette
1. Add a new entry to the palettes definition with bark, leaf, and fruit colors
2. Add it to the palette `<select>` in the HTML
3. Include it in the **🎰 Randomize All** rotation
4. Document it in the `.md` under Color Palettes

### New User Setting
1. Add the HTML control in the sidebar
2. Wire up the JS handler
3. Add a URL parameter for it (read on load, write after generation)
4. Include it in the pin save/restore logic if it should persist with pins
5. Document it in the `.md` under Controls

---

## What NOT to Do
- Do not use `Math.random()` in tree generation — use the seeded RNG
- Do not introduce any external dependency, CDN link, or import
- Do not split the code into multiple files
- Do not overwrite or modify previous version files
- Do not add a new setting without also adding its URL parameter
- Do not remove or rename existing URL parameters (breaks shared links)
- Do not change the behavior of existing seeds without a clear reason — it breaks saved pins and shared URLs

---

## Testing
Open the `.html` file directly in a browser (`File > Open` or drag onto browser). No build step needed.

Key things to verify after any change:
- A tree generates on load without errors
- The browser console is clean (no JS errors)
- Share URL round-trips correctly (copy URL, open in new tab, same tree appears)
- Existing seeds still produce the same trees

---

## Git & GitHub
- Remote: https://github.com/ReeseLloyd/LEDBonsai.git
- Branch: `main`
- After completing changes, commit and push automatically:
  ```
  git add <new files>
  git commit -m "v1.XX: short description of what changed"
  git push
  ```
- Commit messages should describe the functional change, not the mechanism (e.g., "v1.21: Fix screensaver timer reset" not "v1.21: Set timerStart = null in resetTimer()")

### README Sync
- When a new `.md` documentation file is created at a milestone (as directed), copy it to `README.md` in the repo root so GitHub displays it.
- Include `README.md` in the same commit as the versioned `.md` file.
