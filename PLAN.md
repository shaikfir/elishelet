# Elishelet — Project Plan & Spec Review

A 100% client-side "hold-a-sign" meme generator. The user types text; it is warped in
perspective onto the blank paper held in `empty.png` and rendered live, with a PNG download.
Deploys to **Cloudflare Pages** as a pure static site (no server, no build required).

---

## 1. Spec review — findings

### 🔴 Blockers (must fix before building)

1. **Wrong image dimensions.** Spec says `empty.png` is **1080×1920**. The actual file is
   **1568×2740**. Every corner coordinate in §2.1 is therefore off. All geometry must use the
   real resolution.

2. **Wrong corner coordinates.** I re-measured the paper corners against the real image and
   verified them with an overlay (near-perfect fit):

   | Corner | Spec (1080×1920, wrong) | **Measured (1568×2740, correct)** |
   |--------|-------------------------|-----------------------------------|
   | P1 Top-Left      | (174, 1378) | **(272, 1958)** |
   | P2 Top-Right     | (678, 1391) | **(1016, 1930)** |
   | P3 Bottom-Right  | (676, 1756) | **(1020, 2442)** |
   | P4 Bottom-Left   | (183, 1742) | **(198, 2474)** |

   Note the real tilt is the *opposite* of the spec's description: the top edge rises slightly
   to the **right** and the bottom-left dips lowest. Paper is ~745px wide at top, ~820px at
   bottom, ~510px tall.

### 🟠 Design gaps in the spec

3. **Option A (CSS `matrix3d`) and Option B (Canvas) conflict.** CSS transforms can't be
   rasterized into the downloadable PNG, so you'd need two separate rendering paths that must
   be kept pixel-identical — a maintenance trap. **Recommendation: one canvas as the single
   source of truth** for both live preview and download.

4. **Canvas 2D can't do a true homography.** `ctx.transform` is affine only (no perspective).
   The spec hand-waves this. The pragmatic fix: compute the 3×3 homography, then **subdivide
   the text rectangle into a grid and draw each cell with a per-triangle affine transform**
   (standard mesh-warp technique). The paper's perspective is mild, so a ~12×12 mesh looks
   perfect, keeps the native `multiply` blend, and works on every browser incl. mobile.
   (WebGL is a possible later upgrade but unnecessary.)

5. **Fonts must be loaded before first render.** Canvas measures the *fallback* font if the web
   font isn't ready, producing wrong wrapping/sizing. Must `await document.fonts.ready` (and
   per-font `document.fonts.load(...)`) before rendering.

6. **No font-size strategy.** Spec never says how big the text is. Need **auto-shrink-to-fit**
   inside the safe zone + a manual size override.

7. **RTL is under-specified.** The name (Elishelet/אלישלט) and all five fonts are Hebrew fonts —
   this is clearly a Hebrew-first tool. Needs real bidi handling, RTL default, and correct
   per-line alignment on the canvas (`ctx.direction`, careful `textAlign`).

### 🟢 Good in the spec (keep)

- `multiply` blend for realistic ink-over-shadow — correct and cheap. (Add a "normal" fallback
  so light inks don't vanish, plus an opacity control to mimic marker soak.)
- 5% safe-padding inset inside the quad.
- Google Fonts choice (all Hebrew-capable).
- Real-time render target.

---

## 2. Rendering pipeline (revised)

Single `<canvas>` at full **1568×2740**, downscaled via CSS for on-screen preview (HiDPI-aware).
On every input change, throttled to one render per animation frame:

1. Draw `empty.png` to the master canvas.
2. Render wrapped, aligned text to an **offscreen** canvas sized to the safe zone (with 5%
   inset), using chosen font/color/size/line-height/direction.
3. Compute homography `H` mapping the offscreen rect → the four paper corners.
4. **Mesh-warp**: subdivide the offscreen rect into an N×N grid; for each cell map its 4 corners
   through `H` and blit via per-triangle affine transform + clip.
5. Composite the warped layer with `globalCompositeOperation = 'multiply'` (toggle: normal).

Download = `canvas.toBlob('image/png')` → `<a download>`. Preview and download come from the
**same pixels**, so what you see is exactly what you get.

---

## 3. My proposed improvements (opt-in)

1. **Draggable corner calibration overlay** (toggle). Lets anyone re-fit the quad by eye and
   copy out the coordinates — future-proofs against a new background image and removes guesswork.
   The measured corners above are the defaults.
2. **Auto-shrink-to-fit** font sizing + manual slider + line-height control.
3. **Font preloading** gate before first render (correctness).
4. **HiDPI/retina** display canvas; master always full-res for a crisp download.
5. **rAF-throttled render** to hit the <16ms real-time target without redundant work.
6. **Vertical alignment** (top/middle/bottom) in addition to horizontal.
7. **Blend + ink opacity controls** for marker realism; optional subtle jitter for ink bleed.
8. **Mobile-first responsive layout** + Web Share API ("share" on phones) alongside download.
9. **Zero-dependency deploy**: everything self-contained; fonts self-hosted (or `preconnect`)
   so the site works without runtime calls to Google.

---

## 4. Proposed structure

```
elishelet/
├─ index.html            # markup + UI controls
├─ css/style.css
├─ js/
│  ├─ homography.js      # 4-point DLT solve + point mapping (no deps)
│  ├─ warp.js            # mesh triangle-warp onto canvas
│  ├─ text.js            # wrap / align / auto-fit into safe zone
│  └─ app.js             # state, controls, rAF render loop, download
├─ assets/empty.png      # (moved from repo root)
├─ fonts/                # self-hosted woff2 (optional; else preconnect Google)
├─ _headers              # Cloudflare cache headers
└─ config.js             # paper corners, safe-zone %, palette, font list
```

Single-file `index.html` (inlined CSS/JS) is also viable if you prefer one artifact — say the
word and I'll build it that way.

---

## 5. Cloudflare Pages deployment

- Pure static — **no build step**. Framework preset: *None*. Output dir: project root (or
  `dist/`). Connect the repo or drag-and-drop the folder.
- `_headers`: long-cache `empty.png`/fonts/hashed assets, short-cache `index.html`.
- Self-host fonts (recommended) or add `<link rel=preconnect>` to Google Fonts.
- Everything runs in the browser → nothing sensitive server-side; the uploaded photo never
  leaves the device.

---

## 6. Build order

1. Scaffold structure + move `empty.png` → `assets/`; put corners/palette in `config.js`.
2. `homography.js` + `warp.js`; validate with a checkerboard test into the quad.
3. `text.js` (wrap, align, auto-fit, RTL) rendered to offscreen canvas.
4. `app.js`: controls (textarea, LTR/RTL, color swatches, font, size, blend), rAF loop, download.
5. Calibration overlay + polish; mobile layout; multiply/opacity realism pass.
6. `_headers` + deploy to Cloudflare Pages; verify the downloaded PNG at full resolution.
