# אלישלט · Elishelet

A 100% client-side "hold-a-sign" generator. Type text and it is warped in perspective
onto the blank paper held in the photo, rendered live on an HTML5 canvas — then download
it as a PNG. No backend, no build step, no data leaves the browser.

**Live:** https://elishelet.pages.dev

## Features

- **Real-time perspective warp** — a 4-point homography (solved in plain JS, no libraries)
  maps the text onto the paper quad via a triangle mesh, so it follows the page's tilt.
- **Realistic ink** — `multiply` blend lets the paper's shadows/grain show through the text
  (toggle to normal); adjustable ink opacity.
- **Hebrew-first / bidi** — RTL by default with an LTR toggle; five Hebrew Google Fonts
  (Assistant, Rubik, Heebo, Frank Ruhl Libre, Alef).
- **Auto-fit** text sizing (or manual), line-height, and horizontal + vertical alignment.
- **Shareable links** — the full editing state is encoded in the URL hash, so a shared link
  reproduces the exact result. "Copy link" button included.
- **Corner calibration** — a draggable overlay to re-fit the paper quad if the background
  photo is ever replaced.

## Run locally

Any static file server works, e.g.:

```bash
python3 -m http.server 8000
# open http://localhost:8000
```

## Deploy (Cloudflare Pages)

Pure static — no build. Deploy the three files (`index.html`, `empty.jpg`, `_headers`):

```bash
npx wrangler pages deploy dist --project-name=elishelet --branch=main
```

(`dist/` is a build artifact and is git-ignored; recreate it with
`cp index.html empty.jpg _headers dist/`.)

## Files

| File | Purpose |
|------|---------|
| `index.html` | The entire app — markup, CSS, and JS inlined. |
| `empty.jpg`  | Background photo (1568×2740). |
| `_headers`   | Cloudflare cache/security headers. |
| `spec.md`    | Original spec. |
| `PLAN.md`    | Spec review, measured paper coordinates, and build plan. |
