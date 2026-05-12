# Halftone Dot Visualizer

An interactive visualizer for printed halftone screens. A split-view canvas shows the simulated halftone dots on one side and the resulting solid tone on the other, so you can see how dot coverage translates to apparent ink density on paper.

Two modes:

- **K** — single-channel black, one slider for dot coverage (0–100%).
- **CMYK** — four channels with independent coverage sliders, each rendered at its traditional screen angle and blended with a printing-industry color profile.

A **Dot Gain** disclosure in the control strip exposes three press-physics knobs (Dot Gain, Min Dot, Min Dot Printed). The left side reflects what files actually print like on your press; the right side stays a constant GRACoL coated soft-proof reference matching what designers see in Adobe.

An **LPI** slider centered below the halftone half lets you sweep through screen frequencies from coarse (25 LPI, dots clearly visible) to fine (200 LPI, dots dissolve into solid tone at viewing distance), passing through the recognizable CMYK rosette pattern in the middle of the range.

## Try it live

A live build of this work (from a fork) is hosted at <https://chiptoe-svg.github.io/halftonedotsim/> — open it in any modern browser to play with the visualizer without cloning.

## Run locally

It's a static site with no build step. Either:

```sh
npx serve .
```

…or just open `index.html` directly in a browser.

## How it works

A few things worth pointing out if you read the source:

- **Screen angles.** Each ink uses the traditional offset-lithography angle to avoid moiré: cyan 15°, magenta 75°, yellow 0°, black 45°. Single-K mode uses the same 45° black angle.
- **Dot-area model.** Dot radius is computed from coverage as `√(coverage) · cell · 0.48`, which keeps the painted area roughly linear with the slider value. Near 100% the radius smoothly merges toward `cell · 0.69` so neighboring dots overlap into a solid before snapping to a fully filled fill at 100%.
- **CMYK color blending.** The solid-tone side uses the **Neugebauer equations** over a 16-point Neugebauer primary set sampled from the **GRACoL2013 CRPC6** characterization (paper, four primaries, six two-color overprints, four three-color overprints, and 4-color black). Each primary is weighted by the product of coverage / (1 − coverage) per channel, scaled against the simulated paper white, blended in linear light, then converted back to sRGB.
- **Rendering, with accurate high-coverage dots.** Each ink screen renders to its own offscreen canvas using normal source-over compositing — overlapping dots of the _same_ ink merge flat into one ink film rather than darkening each other. The four offscreen layers are then composited onto the main canvas with `globalCompositeOperation = "multiply"`, so overprinted _different_ inks (cyan over magenta, etc.) still darken realistically. This separation matters most above ~70% single-channel coverage, where adjacent dots heavily overlap: a naïve single-pass multiply renderer would produce a visible darker lattice between same-ink dots that doesn't correspond to anything in real printing.
- **Press-physics gain model.** The Dot Gain disclosure runs each channel's slider value through a hard min-dot floor and a two-component additive gain (`bell`-shape midtone spread + `decay`-shape small-end bloom) anchored by the three user inputs. The right-side reference independently applies hardcoded G7/GRACoL coated TVI per channel (C+12, M+14, Y+13, K+18 at 50%) before the Neugebauer mix, so it always represents the Adobe soft-proof equivalent.
- **Screen frequency (LPI).** The LPI slider drives the cell pitch of all four ink screens via `cell = max(2, 600 / lpi)`. Higher LPI means smaller cells which means smaller dots. The classic CMYK rosette emerges automatically at mid LPI because the four screen angles (C 15°, M 75°, Y 0°, K 45°) interfere visibly once dots are small enough. At the high end of the slider, the geometric dot grid can't physically dissolve at pixel scale, so a smooth crossfade ramps the halftone half toward the predicted reference color as `cell` drops below 4 px — by LPI 200 the two halves read as a single seamless tone.
- **Performance: cached dot patterns.** At each rendering frame, every ink screen builds (or reuses) a tiny cached `cell × cell` dot tile (supersampled 2×) and fills its entire offscreen with one `fillRect` using `createPattern` + `setTransform`. The browser's native pattern tiler replaces hundreds of thousands of per-dot `arc()` calls with a single fill, keeping every LPI value in the sub-millisecond range during slider drag. `requestAnimationFrame` coalesces rapid slider input into one draw per frame; an offscreen-canvas pool reuses per-channel backing canvases across draws.

## File layout

| File          | Purpose                                                        |
| ------------- | -------------------------------------------------------------- |
| `index.html`  | Markup: canvas + control strip (mode toggle, sliders, meter)   |
| `app.js`      | Canvas rendering, color math, slider/mode wiring               |
| `styles.css`  | Print-shop visual style, responsive layout                     |

## Deploying

This is a static site with no build step, so it deploys cleanly to any static host (GitHub Pages, Netlify, Cloudflare Pages, S3 + CloudFront, etc.). For GitHub Pages: in Settings → Pages, set _Source: Deploy from a branch_ and pick `main` / `/ (root)`. Once enabled, every push to `main` triggers a Pages rebuild and the live site updates within a minute or two — no GitHub Actions workflow needed.

`index.html` references `app.js` and `styles.css` with a `?v=YYYY-MM-DD-N` query string so browsers refetch the assets after each deploy instead of serving stale cached copies. Bump the version suffix when shipping non-trivial changes.
