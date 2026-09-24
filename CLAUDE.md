# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Tank Life is a browser aquarium ecosystem simulation — a single self-contained HTML file (`tank-life.html`), canvas-rendered, no build step, no external JS libraries, no framework. Everything (HTML, CSS, JS) lives in that one file.

It is also published as a Claude Artifact (the live demo linked from `README.md`); `tank-life.html` in this repo is the canonical source, and the artifact is a mirror published from it.

## Running it

No build step, no dependencies to install. Either open `tank-life.html` directly in a browser, or serve the directory with any static file server, e.g.:

```bash
python -m http.server 8000
```

Then visit `http://localhost:8000/tank-life.html`. There is no test suite, linter, or build/deploy tooling in this repo.

## Architecture

The entire simulation lives inside one IIFE in the `<script>` tag at the bottom of `tank-life.html`, structured as a few internal modules in this order:

- **`Sound`** — a small synthesized audio engine (Web Audio oscillators only, no audio files). Exposes event methods (`eat`, `predation`, `split`, `hatch`, `gameOver`) called from the simulation step. Mute state persists via `localStorage` (`tankLifeSound`).
- **`History`** — persists outcomes across tank resets via `localStorage` (`tankLifeHistory`), keyed by stable **behavior role** (`breeder`/`balanced`/`apex`/`evader`), not by the generated strain name — names/colors reshuffle every run but roles don't.
- **`SPECIES`** — the core data model: 4 fixed behavior roles (fast breeder, all-rounder, apex predator, evader), each with a `ranges` object of min/max bounds for every stat (speed, metabolism, vision, repro cost, etc). `rollSpecies()` randomizes each strain's actual stats within its role's ranges every tank reset; `assignFlavor()` independently randomizes name/color/visual silhouette. This decoupling (stable role vs. re-rolled flavor and stats) is the key design idea — no two tanks look or play the same, but role identity is the one constant worth tracking in `History`.
- **Simulation step (`stepLogic`)** — runs at a fixed 30 Hz tick via an accumulator in the `requestAnimationFrame` loop (`loop()`), independent of display framerate; the speed buttons (1x/2x/4x) scale the accumulator, not the tick rate. Per creature per tick: energy drains, then a priority chain decides behavior — flee a threat > hunt prey (if big/hungry enough) > forage for plankton > wander — followed by movement, wall collision, and a reproduction (mitosis split) check. This is an **O(n²) sim**: every creature scans every other creature/food item each tick for nearest-food/nearest-prey/nearest-threat, which is why `MAX_CREATURES` (currently 400) exists purely as a performance ceiling, not a design choice — respect it if population limits ever come up.
- **Rendering (`draw`)** — canvas-only, no images. Creature bodies are procedural wobbling blobs (`blobPoints`) built from a per-creature `membrane` (a few sine-wave harmonics whose shape depends on the strain's `visual` trait: `tail`/`bulge`/`spiked`/`streak`), plus mode-based overlay rings (hunting/fleeing/zoomie/split-flash/selected). Colors read CSS custom properties (`--glow`, `--food`, etc.) at draw time so the canvas art follows the light/dark theme automatically.
- **UI wiring** — legend cards, the creature inspector panel, and the game-over overlay are all built by direct `innerHTML`/DOM calls (`buildLegend`, `updateLegend`, `selectCreature`/`updateInspector`, `triggerGameOver`) rather than any templating system.

### Theming

Colors are CSS custom properties on `:root`, with a `prefers-color-scheme: dark` media query and a `[data-theme="dark"]`/`[data-theme="light"]` attribute override layer for explicit theme switching (used when embedded as an Artifact). When editing colors, add both the light (`:root`) and dark (`@media` block + `[data-theme="dark"]`) values — the canvas rendering code reads these live via `getComputedStyle`, so it doesn't need separate updates.

### Editing workflow

Since this also feeds a published Claude Artifact, verify changes by serving the file locally and testing in a browser before treating an edit as done — there's no automated test coverage to catch regressions in the simulation balance or rendering.
