# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

A self-contained, single-file HTML pattern generator for **Dune Weaver** — a sand table that draws by interpolating polar (theta-rho) coordinates. The page generates spirograph-style patterns from a chosen base shape, transforms them across N iterations (rotation + scale + drift), converts to polar, and uploads the result as a `.thr` file to a Dune Weaver controller.

The repo is just `spirograph.html` plus `LICENSE`. There is no build, no package manager, no tests, and no server in this repo.

## Running / developing

Open `spirograph.html` directly in a browser, or serve it from any static server:

```bash
python3 -m http.server 8000
# then visit http://localhost:8000/spirograph.html
```

Note on the **Save** button: it `POST`s a multipart form with a `.thr` file to `${window.location.origin}/upload_theta_rho` — i.e. the Dune Weaver controller's API. To exercise Save end-to-end, the page must be served *from the Dune Weaver host* (or behind a reverse proxy that forwards `/upload_theta_rho`). When opened via `file://` or a plain static server, generation/preview/play work but Save will fail. Don't "fix" the relative URL or strip the upload — the same-origin convention is intentional.

## Architecture

Everything lives inside one IIFE in `spirograph.html`. Three layers worth knowing about before editing:

### 1. Shape registry (`SHAPES`)

Each entry is `{ l: label, c: category, g: (points) => Array<{x,y}> }`. Generators all return points roughly within the unit circle (the helper `parametric()` normalizes to max radius 1). Four primitive generators do the work:

- `regularPolygon(n, pts)` — n-gons with smooth edge interpolation
- `starShape(n, innerR, pts)` — alternating outer/inner radius
- `parametric(fn, pts, tMax)` — any `t → {x,y}` curve, auto-normalized
- `quad(corners, pts)` — closed polyline through arbitrary corners (used for non-regular polygons like cross/arrow despite the name)
- `wire3d(verts, edges, pts)` — projects 3D/4D wireframes (cube, tesseract) to 2D with fixed view angles

To add a shape: add one entry to `SHAPES` with a category `c` — the dropdown rebuilds optgroups from the categories automatically.

### 2. Transform pipeline (`generate()`)

For each of `iterations` copies of the base shape, it applies: scale `size * scale^i`, rotation `rotation * i`, then a polar drift (`thetaDrift * i`, `rhoDrift * i`). The rho drift uses a *triangle-wave fold* (`while (rho > 1) rho = 2 - rho; while (rho < 0) rho = -rho;`) so points reflect off the rim instead of clipping — keep this if you touch it. Between iterations it inserts 4 interpolated bridge points so the resulting `.thr` traces a continuous path.

### 3. Cartesian → polar conversion

The final loop converts each `(x, y)` to `(theta, rho)` while *accumulating* theta across `2π` boundaries (continuous unwrapping), so the saved file's theta increases/decreases monotonically across full revolutions rather than wrapping. This is what Dune Weaver expects — don't replace it with a naive `atan2`. Note `x, y` are negated before conversion (orientation convention for the table).

The output file is plain text, one `theta rho` pair per line, 3-decimal precision.

### Controls

Each parameter has a paired slider + number input that mirror each other via `input` listeners. Changing the shape resets all controls to defaults from the `controls` array. Live re-generation runs on every input event; this is fine because the math is cheap (`points * iterations` in the low thousands).

## Style conventions in this file

The code is intentionally compact: short variable names (`pts`, `cx`, `dpr`), inline math, no external dependencies, single IIFE. New code should match — don't introduce a build step, modules, or a framework. CSS uses HSL custom properties with a `prefers-color-scheme: light` override; keep that pattern if adding UI.
