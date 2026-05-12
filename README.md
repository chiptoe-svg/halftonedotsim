# Halftone Dot Visualizer

An interactive visualizer for printed halftone screens. A split-view canvas shows the simulated halftone dots on one side and the resulting solid tone on the other, so you can see how dot coverage translates to apparent ink density on paper.

Two modes:

- **K** — single-channel black, one slider for dot coverage (0–100%).
- **CMYK** — four channels with independent coverage sliders, each rendered at its traditional screen angle and blended with a printing-industry color profile.

## Live demo

<https://chiptoe-svg.github.io/halftonedotsim/>

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
- **Rendering.** The halftone side draws each channel with `globalCompositeOperation = "multiply"` so overprinted dots darken realistically rather than just stacking opaquely.

## File layout

| File          | Purpose                                                        |
| ------------- | -------------------------------------------------------------- |
| `index.html`  | Markup: canvas + control strip (mode toggle, sliders, meter)   |
| `app.js`      | Canvas rendering, color math, slider/mode wiring               |
| `styles.css`  | Print-shop visual style, responsive layout                     |

## Deploying

This repo is configured for **GitHub Pages**, served from the `main` branch at the repo root. Any push to `main` updates the live site within a minute or two — no build step, no workflow file. To enable from scratch: in the repo's Settings → Pages, set _Source: Deploy from a branch_ and pick `main` / `/ (root)`.
