# Halftone Dot Visualizer

An interactive visualizer for printed halftone screens. A split-view canvas shows the simulated halftone dots on one side and the resulting solid tone on the other, so you can see how dot coverage translates to apparent ink density on paper.

Two modes:

- **K** — single-channel black, one slider for dot coverage (0–100%).
- **CMYK** — four channels with independent coverage sliders, each rendered at its traditional screen angle and blended with a printing-industry color profile.

## Try it live

A live build is hosted at <https://rcongdo.github.io/halftonedotsim/> — open it in any modern browser to play with the visualizer without cloning.

## Run locally

It's a static site with no build step. Either:

```sh
npx serve .
```

…or just open `index.html` directly in a browser.

## How it works

A few things worth pointing out if you read the source:

- **Screen angles.** Each ink uses the traditional offset-lithography angle to avoid moiré: cyan 15°, magenta 75°, yellow 0°, black 45°. In single-K mode the screen is rotated to −16° for a less mechanical look.
- **Dot-area model.** Dot radius is computed from coverage as `√(coverage) · cell · 0.48`, which keeps the painted area roughly linear with the slider value. Near 100% the radius smoothly merges toward `cell · 0.69` so neighboring dots overlap into a solid before snapping to a fully filled fill at 100%.
- **CMYK color blending.** The solid-tone side uses the **Neugebauer equations** over a 16-point Neugebauer primary set sampled from the **GRACoL2013 CRPC6** characterization (paper, four primaries, six two-color overprints, four three-color overprints, and 4-color black). Each primary is weighted by the product of coverage / (1 − coverage) per channel, scaled against the simulated paper white, blended in linear light, then converted back to sRGB.
- **Rendering, with accurate high-coverage dots.** Each ink screen renders to its own offscreen canvas using normal source-over compositing — overlapping dots of the _same_ ink merge flat into one ink film rather than darkening each other. The four offscreen layers are then composited onto the main canvas with `globalCompositeOperation = "multiply"`, so overprinted _different_ inks (cyan over magenta, etc.) still darken realistically. This separation matters most above ~70% single-channel coverage, where adjacent dots heavily overlap: a naïve single-pass multiply renderer would produce a visible darker lattice between same-ink dots that doesn't correspond to anything in real printing.

## File layout

| File          | Purpose                                                        |
| ------------- | -------------------------------------------------------------- |
| `index.html`  | Markup: canvas + control strip (mode toggle, sliders, meter)   |
| `app.js`      | Canvas rendering, color math, slider/mode wiring               |
| `styles.css`  | Print-shop visual style, responsive layout                     |

## Deploying

This is a static site with no build step, so it deploys cleanly to any static host (GitHub Pages, Netlify, Cloudflare Pages, S3 + CloudFront, etc.). The current live build is served from the `gh-pages` branch at <https://rcongdo.github.io/halftonedotsim/>.
