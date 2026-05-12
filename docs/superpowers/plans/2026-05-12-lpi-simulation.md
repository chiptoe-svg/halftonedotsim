# LPI Simulation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a screen-frequency (LPI) slider that controls cell pitch of the halftone screens, letting the user see the same coverage at different print resolutions — from coarse dots, through the CMYK rosette pattern, to a fine screen indistinguishable from the GRACoL reference at high LPI.

**Architecture:** Replace the canvas-derived `cell` formula in `drawVisualizer()` with an LPI-driven one (`cell = max(4, 750 / lpi)`). Add a horizontal range slider in a new row between the visualizer and control strip, centered under the halftone (left) half of the canvas. Live redraw on `input`. No changes to the dot-rendering, screen-angle, or color math — the existing rendering pipeline naturally produces the rosette pattern at mid LPI and pixel-scale dots at high LPI.

**Tech Stack:** Vanilla HTML/CSS/JS, no build step, no test framework. Visual & behavioral verification via Playwright MCP and `npx serve`.

**Reference docs:**
- Spec: `docs/superpowers/specs/2026-05-12-lpi-simulation-design.html`

---

## Task 0: Set up the verification harness

This task starts the dev-server / Playwright loop used by every subsequent task. Skip if you already have `npx serve` running on port 4173 and Playwright open.

**Files:** No code changes — operational only.

- [ ] **Step 1: Start a local static server on port 4173**

Run (in background, from the repo root):
```sh
npx --yes serve . -l 4173 -n
```

- [ ] **Step 2: Wait until the server returns 200**

```sh
until curl -fsS http://localhost:4173 >/dev/null 2>&1; do sleep 0.3; done; echo "ready"
```
Expected output: `ready`

- [ ] **Step 3: Open the page in Playwright at a reasonable viewport**

Tools: `mcp__playwright__browser_resize` (1280×900), then `mcp__playwright__browser_navigate` to `http://localhost:4173/`.

Expected: page loads, title is "Halftone Dot Visualizer", only console error is the favicon 404 (ignorable).

---

## Task 1: Add the LPI row markup

**Files:**
- Modify: `index.html` (insert a new `<section>` between `.visualizer` and `.control-strip`)

**Why:** Establishes the static structure for the new control. No styling and no behavior yet — the slider renders unstyled and changing it has no rendering effect.

- [ ] **Step 1: Locate the closing `</section>` of `.visualizer` and the opening `<section class="control-strip"...>`**

The current structure (around lines 11-18):
```html
    <main class="app-shell">
      <section class="visualizer" aria-label="Halftone dot preview">
        <canvas id="halftoneCanvas"></canvas>
      </section>

      <section
        class="control-strip"
        id="controlStrip"
        data-mode="single"
        aria-label="Halftone controls"
      >
```

You will insert a new `<section class="lpi-row">` between the closing `</section>` of `.visualizer` and the opening `<section class="control-strip">`.

- [ ] **Step 2: Insert the LPI row markup**

Add exactly this markup as a sibling between the two existing sections:

```html
      <section class="lpi-row" aria-label="Halftone screen frequency">
        <div class="lpi-control">
          <label class="lpi-label" for="lpiSlider">LPI</label>
          <input
            id="lpiSlider"
            class="lpi-slider"
            type="range"
            min="25"
            max="200"
            step="1"
            value="25"
          />
          <span id="lpiValue" class="lpi-value">25</span>
        </div>
        <div class="lpi-row-right" aria-hidden="true"></div>
      </section>
```

The `lpi-control` div will be the left column (under the halftone-dot half of the canvas) and the empty `lpi-row-right` div is the right column (under the GRACoL reference half).

- [ ] **Step 3: Reload the page in Playwright and verify the row renders**

Navigate Playwright to `http://localhost:4173/`. Run `mcp__playwright__browser_evaluate`:

```js
() => {
  const row = document.querySelector('.lpi-row');
  const slider = document.getElementById('lpiSlider');
  const value = document.getElementById('lpiValue');
  return {
    rowExists: !!row,
    sliderMin: slider?.min,
    sliderMax: slider?.max,
    sliderStep: slider?.step,
    sliderValue: slider?.value,
    valueText: value?.textContent.trim(),
    rowIsAfterVisualizer: row?.previousElementSibling?.classList.contains('visualizer'),
    rowIsBeforeControlStrip: row?.nextElementSibling?.classList.contains('control-strip'),
  };
}
```

Expected output:
```json
{
  "rowExists": true,
  "sliderMin": "25",
  "sliderMax": "200",
  "sliderStep": "1",
  "sliderValue": "25",
  "valueText": "25",
  "rowIsAfterVisualizer": true,
  "rowIsBeforeControlStrip": true
}
```

- [ ] **Step 4: Commit**

```sh
git add index.html
git commit -m "Add LPI row markup with range slider 25-200

A new <section class=\"lpi-row\"> between .visualizer and
.control-strip. Two-column grid mirroring the canvas split: a label /
slider / value group in the left column (under the halftone-dot half
of the canvas), empty right column. Default value 25 matches today's
chunky-dot first-load look.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Style the LPI row

**Files:**
- Modify: `styles.css` (add new rules; also update `.app-shell` grid-template-rows to accommodate the new row)

**Why:** Make the slider look correct and align with the existing print-shop aesthetic. The 2-column grid mirrors the canvas split so the slider sits centered under the halftone half. After this task the page looks right but the slider still doesn't affect rendering.

- [ ] **Step 1: Update `.app-shell` to accommodate the new row**

Locate `.app-shell` in `styles.css` (around line 41-47):

```css
.app-shell {
  display: grid;
  height: 100vh;
  height: 100svh;
  min-height: 540px;
  grid-template-rows: minmax(0, 1fr) auto;
}
```

Change `grid-template-rows` to add a third auto-height track:

```css
.app-shell {
  display: grid;
  height: 100vh;
  height: 100svh;
  min-height: 540px;
  grid-template-rows: minmax(0, 1fr) auto auto;
}
```

Order in DOM: visualizer (1fr), lpi-row (auto), control-strip (auto). The grid order matches.

- [ ] **Step 2: Insert the LPI row rules**

Add these rules in `styles.css` just before the existing `.control-strip` rule (around line 67):

```css
.lpi-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  padding: 10px clamp(18px, 4vw, 48px);
  background: rgba(255, 250, 240, 0.88);
  border-top: 1px solid var(--line);
  backdrop-filter: blur(16px);
}

.lpi-control {
  display: flex;
  align-items: center;
  gap: 14px;
  min-width: 0;
}

.lpi-label {
  flex: 0 0 auto;
  font-size: 0.74rem;
  font-weight: 900;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: #51493d;
}

.lpi-slider {
  flex: 1;
  min-width: 0;
}

.lpi-value {
  flex: 0 0 auto;
  min-width: 36px;
  text-align: right;
  font-variant-numeric: tabular-nums;
  font-weight: 800;
  font-size: 0.9rem;
  color: var(--ink);
}
```

The right column (`.lpi-row-right`) needs no rules — it's an empty grid cell that holds the layout symmetry. The slider itself inherits the existing `input[type="range"]` styling that drives all the other range sliders in the app.

- [ ] **Step 3: Reload Playwright and visually verify**

Navigate to `http://localhost:4173/` and take a screenshot:

`mcp__playwright__browser_take_screenshot` with filename `task2-lpi-styled.png`. Read the screenshot back. Verify:

- The new LPI row appears between the canvas and the existing control strip.
- The slider sits in the left half of the row (under the halftone-dot half of the canvas).
- The right half of the row is empty.
- "LPI" label sits to the left of the slider; the value "25" sits to the right.
- The slider track is the same red color (--accent) as the other sliders.
- The slider thumb (filled circle) is at the LEFT end of the track (matches min=25).
- The row's background and border-top visually flow with the control strip below.

- [ ] **Step 4: Commit**

```sh
git add styles.css
git commit -m "Style LPI row: 2-column grid mirroring canvas split

Slider sits centered in the left column (under the halftone-dot half);
right column is intentionally empty (GRACoL reference doesn't have a
screen frequency). app-shell grid now has three rows: canvas (1fr),
lpi-row (auto), control-strip (auto). Background and border continue
the control-area visual treatment.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Wire the LPI slider to the rendering pipeline

**Files:**
- Modify: `app.js` — add DOM refs, sync function, replace the cell formula in `drawVisualizer`, wire the input listener

**Why:** This is the substantive behavior change. After this task, dragging the LPI slider changes the dot size live on the canvas.

- [ ] **Step 1: Add LPI DOM refs near the other DOM-ref lines**

Locate the block at the top of `app.js` (around lines 10-13) where the dot-gain inputs are declared. Immediately after that block, add:

```js
const lpiSlider = document.querySelector("#lpiSlider");
const lpiValue = document.querySelector("#lpiValue");
```

- [ ] **Step 2: Add a `syncLpiValue()` helper next to the existing sync functions**

Find `function syncCmykControls() { ... }` in `app.js` (around line 90). Immediately after that function, add:

```js
function syncLpiValue() {
  const lpi = Number(lpiSlider.value);
  const fillPercent = ((lpi - 25) / (200 - 25)) * 100;

  lpiSlider.style.setProperty("--track-fill", `${fillPercent}%`);
  lpiValue.textContent = String(lpi);
}
```

The `--track-fill` is the same CSS custom property the existing sliders use (see `input[type="range"]::-webkit-slider-runnable-track` rule). For LPI we normalize the 25-200 range to a 0-100% fill before writing it.

- [ ] **Step 3: Replace the cell-size formula in `drawVisualizer`**

Find `function drawVisualizer() {...}` in `app.js`. Look for the line:

```js
const cell = Math.max(15, Math.min(30, shorterSide / 18));
```

Replace that single line with:

```js
const lpi = Number(lpiSlider.value);
const cell = Math.max(4, 750 / lpi);
```

The `shorterSide` variable that the old formula used is no longer needed; the line above (`const shorterSide = Math.min(width, height);`) can also be deleted since nothing else references it.

Verify: search the rest of `drawVisualizer` for `shorterSide`. If no other reference exists, delete that line too. (Spot check by reading `app.js` around the relevant lines.)

- [ ] **Step 4: Wire the input listener and the initial sync**

At the bottom of `app.js`, near the other event listeners (around line 440-450, after the dot-gain trigger click handler), add:

```js
lpiSlider.addEventListener("input", () => {
  syncLpiValue();
  drawVisualizer();
});
```

Then find the end-of-file initialization block (around line 450-455) that looks like:

```js
syncSingleControl();
syncCmykControls();
setMode("single");
resizeCanvas();
```

Add `syncLpiValue();` between `syncCmykControls();` and `setMode("single");`:

```js
syncSingleControl();
syncCmykControls();
syncLpiValue();
setMode("single");
resizeCanvas();
```

- [ ] **Step 5: Verify behavior in Playwright**

Reload the page. Run:

```js
() => {
  const canvas = document.getElementById('halftoneCanvas');
  document.querySelector('.mode-button[data-mode="single"]').click();
  document.getElementById('coverageSlider').value = '50';
  document.getElementById('coverageSlider').dispatchEvent(new Event('input', {bubbles: true}));

  const at25 = canvas.toDataURL('image/png').length;

  // Dial up to 100 LPI
  const lpiSlider = document.getElementById('lpiSlider');
  lpiSlider.value = '100';
  lpiSlider.dispatchEvent(new Event('input', {bubbles: true}));
  const at100 = canvas.toDataURL('image/png').length;

  // Dial up to 200 LPI
  lpiSlider.value = '200';
  lpiSlider.dispatchEvent(new Event('input', {bubbles: true}));
  const at200 = canvas.toDataURL('image/png').length;

  // Reset to 25
  lpiSlider.value = '25';
  lpiSlider.dispatchEvent(new Event('input', {bubbles: true}));

  return {
    bytesAt25: at25,
    bytesAt100: at100,
    bytesAt200: at200,
    canvasChangesAcrossLPI: at25 !== at100 && at100 !== at200 && at25 !== at200,
    lpiValueDisplay: document.getElementById('lpiValue').textContent,
  };
}
```

Expected:
- `canvasChangesAcrossLPI`: `true` — three different byte sizes confirms the rendering responds to LPI
- `lpiValueDisplay`: `"25"` — the value display reset correctly

If `canvasChangesAcrossLPI` is `false`, the `drawVisualizer` change didn't take effect — re-read Step 3.

- [ ] **Step 6: Commit**

```sh
git add app.js
git commit -m "Wire LPI slider to cell-size formula in drawVisualizer

Replaces the canvas-derived cell formula
(Math.max(15, Math.min(30, shorterSide / 18))) with an LPI-driven one
(Math.max(4, 750 / lpi)). Adds DOM refs for the slider and value
display, a syncLpiValue() helper to keep the value text and track-fill
CSS in sync with the slider state, and an input-event listener that
redraws live. Initial sync runs at startup.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: Visual tuning — verify the four-phase progression

**Files:**
- Modify: `app.js` lines around the cell formula (the constants `4` and `750`), if and only if the visual checkpoints fail.
- Possibly modify: `app.js` — add a post-blur step if the smooth-tone acceptance criterion can't be met by formula tuning alone.

**Why:** The cell formula `cell = max(4, 750 / lpi)` is a first guess. The spec's acceptance criterion is that LPI 200 with default dot-gain produces a left half **indistinguishable from the right (GRACoL reference) half** at normal viewing distance. The intermediate phases (rosette at ~85, noise at ~130) also need to be visible. This task walks the slider through five canonical values and either confirms each phase or tunes the constants until they pass.

Approach: keep checking, only adjust when a checkpoint genuinely fails. **Do not** speculatively change constants. **Do** make small adjustments and re-screenshot.

### Step 1: Establish CMYK reference state at default dot-gain

In Playwright, navigate to `http://localhost:4173/` and set state:

```js
() => {
  document.querySelector('.mode-button[data-mode="cmyk"]').click();
  for (const [ch, v] of [['cyan','50'],['magenta','50'],['yellow','50'],['black','50']]) {
    const s = document.getElementById(ch+'Slider');
    s.value = v;
    s.dispatchEvent(new Event('input', {bubbles: true}));
  }
  // Default dot-gain values are already 18 / 1 / 3 — confirm
  return {
    dotGain: document.getElementById('dotGainInput').value,
    minDot: document.getElementById('minDotInput').value,
    minDotPrinted: document.getElementById('minDotPrintedInput').value,
  };
}
```

Expected: `{ dotGain: "18", minDot: "1", minDotPrinted: "3" }`.

CMYK at 50/50/50/50 produces a neutral gray that exercises all four channels' rotated screens — ideal for spotting rosette emergence vs. noise vs. smooth tone.

- [ ] **Step 2: Checkpoint LPI 25 — "Discrete dots"**

```js
() => {
  const s = document.getElementById('lpiSlider');
  s.value = '25';
  s.dispatchEvent(new Event('input', {bubbles: true}));
  return 'set 25';
}
```

Screenshot `tune-lpi-25.png`. Read it back. Verify:

- Individual dots are clearly visible across the canvas.
- The four screen angles (C 15°, M 75°, Y 0°, K 45°) are visually distinguishable per channel — you can see directional structure.
- Cell size appears similar to the visualizer's pre-LPI default look.

PASS criterion: looks like "what the visualizer used to render before this feature."

- [ ] **Step 3: Checkpoint LPI 85 — "Newsprint / late dots"**

```js
() => {
  const s = document.getElementById('lpiSlider');
  s.value = '85';
  s.dispatchEvent(new Event('input', {bubbles: true}));
  return 'set 85';
}
```

Screenshot `tune-lpi-85.png`. Verify:

- Dots are smaller; you can still see individual dots up close but the screen is denser.
- Rosette-like structure is starting to be visible if you squint.

PASS criterion: looks finer than 25 but not yet a clean rosette texture.

- [ ] **Step 4: Checkpoint LPI 130 — "Rosette"**

```js
() => {
  const s = document.getElementById('lpiSlider');
  s.value = '130';
  s.dispatchEvent(new Event('input', {bubbles: true}));
  return 'set 130';
}
```

Screenshot `tune-lpi-130.png`. Verify:

- The classic CMYK rosette / flower texture is visible — small dot clusters arranged in a regular flower pattern.
- This is the most recognizably "printed" look. If a print designer saw this they'd say "rosette pattern" without hesitation.

PASS criterion: clear rosette texture visible.

If FAIL (no rosette, looks like uniform noise or solid tone):
- The formula may be giving cells too small. Try changing `750 / lpi` to `900 / lpi` in `drawVisualizer` (larger cells at every LPI). Re-take checkpoints 2-4. Don't over-tune — small steps.
- Conversely, if checkpoint 25 looks too small at the new formula, lower the constant back toward 750. The goal is checkpoint 25 ≈ current-default-cell AND checkpoint 130 shows clear rosette. Both have to be satisfied.

- [ ] **Step 5: Checkpoint LPI 175 — "Noisy texture"**

```js
() => {
  const s = document.getElementById('lpiSlider');
  s.value = '175';
  s.dispatchEvent(new Event('input', {bubbles: true}));
  return 'set 175';
}
```

Screenshot `tune-lpi-175.png`. Verify:

- The rosette has broken down — what was a structured flower pattern is now a fine grain texture.
- You can still see texture variation across the canvas if you look closely, but it doesn't read as "dots" anymore.

PASS criterion: fine grain / noise; not solid; not rosette.

- [ ] **Step 6: Checkpoint LPI 200 — "Smooth tone" (acceptance criterion)**

```js
() => {
  const s = document.getElementById('lpiSlider');
  s.value = '200';
  s.dispatchEvent(new Event('input', {bubbles: true}));
  return 'set 200';
}
```

Screenshot `tune-lpi-200.png`. **This is the acceptance test.**

Verify by comparing the left half of the canvas to the right half:

- The two halves should be **indistinguishable** at normal screen viewing distance (about 18-24 inches from the monitor).
- The left side should NOT have visible discrete dots, rosettes, or grain.
- The color tone of the left should match the right (both are showing 50/50/50/50 CMYK at default GRACoL TVI for the right and default press-gain for the left — math says they should land at similar colors).

PASS criterion: at screen viewing distance, no visible difference between halves.

If FAIL (left side still shows visible dots / texture):
1. **First**, try lowering the pixel floor from `4` to `2`. In `app.js`, change `Math.max(4, 750 / lpi)` to `Math.max(2, 750 / lpi)`. Re-screenshot LPI 25, 130, 200. The smaller floor lets cells go to ~3.75 px at LPI 200 with the existing formula constant. Check 200 — if still dotty, continue.
2. **Second**, try a smaller formula constant. Change `750 / lpi` to `600 / lpi` (cells 25% smaller everywhere). Re-screenshot 25, 130, 200. Check that 25 doesn't shrink so much it stops looking like the current default look (cells should still be ~24+ px at LPI 25).
3. **Third (if needed)**, add a post-blur step. In `compositeInkScreen`, before drawing the offscreen onto the main ctx, optionally apply a CSS filter. Implementation:

   Find `function compositeInkScreen(offscreen, width, height) { ... }` in `app.js`. Modify it to:

   ```js
   function compositeInkScreen(offscreen, width, height, cell) {
     if (!offscreen) {
       return;
     }

     ctx.save();
     ctx.globalCompositeOperation = "multiply";
     ctx.filter = cell < 5 ? `blur(${Math.max(0.5, (5 - cell) * 0.6)}px)` : "none";
     ctx.drawImage(offscreen, 0, 0, width, height);
     ctx.restore();
   }

   function drawInkScreen(screen, clip, width, height, cell) {
     compositeInkScreen(renderInkScreen(screen, clip, width, height, cell), width, height, cell);
   }
   ```

   The blur is gated to only apply when `cell < 5 px`. At LPI 200 with the default constant, cell is around 3.75 px → blur radius ~0.75 px, which is enough to dissolve dot edges visually without affecting LPI 25-150 at all.

   Re-screenshot all five checkpoints after adding the blur. Verify: 25/85/130/175 unchanged (cell ≥ 5), 200 now passes the indistinguishable criterion.

If after all three adjustments LPI 200 still fails the acceptance criterion, STOP and report it. The spec's acceptance is concrete; if the formula model can't deliver it, we need to revisit the design (e.g., expand the LPI range to 300, or change the rendering model).

- [ ] **Step 7: Final commit (whether or not tuning was needed)**

If you made adjustments during steps 4-6, commit them now. Use a descriptive message:

```sh
git add app.js
git commit -m "Tune LPI cell formula for smooth tone at LPI 200

<one or two sentences about what changed and why — for example:
'Lowered pixel floor from 4 to 2 and added a sub-pixel post-blur in
compositeInkScreen when cell < 5 px, so the halftone half becomes
visually indistinguishable from the GRACoL reference at LPI 200.'>

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

If no adjustments were needed, skip this step — the formula from Task 3 is good as-is.

---

## Task 5: Update README to document the LPI control

**Files:**
- Modify: `README.md` — add a short sentence about the LPI slider; add a bullet to "How it works".

**Why:** Keeps documentation honest about what the visualizer can show.

- [ ] **Step 1: Update the "Two modes" section**

Find this paragraph in `README.md`:

```md
A **Dot Gain** disclosure in the control strip exposes three press-physics knobs (Dot Gain, Min Dot, Min Dot Printed). The left side reflects what files actually print like on your press; the right side stays a constant GRACoL coated soft-proof reference matching what designers see in Adobe.
```

Add a sentence about LPI directly after it:

```md
An **LPI** slider centered below the halftone half lets you sweep through screen frequencies from coarse (25 LPI, dots clearly visible) to fine (200 LPI, dots dissolve into solid tone at viewing distance), passing through the recognizable CMYK rosette pattern in the middle of the range.
```

- [ ] **Step 2: Add a bullet to "How it works"**

In the "How it works" section, find the "Press-physics gain model" bullet (the last one added). Insert a new bullet after it:

```md
- **Screen frequency (LPI).** The LPI slider drives the cell pitch of all four ink screens via `cell = max(4, 750 / lpi)`. Higher LPI means smaller cells which means smaller dots. The classic CMYK rosette emerges automatically at mid LPI because the four screen angles (C 15°, M 75°, Y 0°, K 45°) interfere visibly once dots are small enough. At the high end of the slider, dots are small enough that the eye blends them into solid tone at typical screen viewing distance.
```

- [ ] **Step 3: Commit**

```sh
git add README.md
git commit -m "Document LPI slider in README

Adds a short description of the LPI slider to the 'Two modes' section
and a 'Screen frequency (LPI)' bullet to 'How it works' explaining the
cell formula and why rosettes emerge naturally at mid LPI.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: End-to-end behavioral verification + cleanup

**Files:** No code changes — verification + cleanup only.

- [ ] **Step 1: Verify K mode also responds to LPI**

```js
() => {
  document.querySelector('.mode-button[data-mode="single"]').click();
  document.getElementById('coverageSlider').value = '50';
  document.getElementById('coverageSlider').dispatchEvent(new Event('input', {bubbles: true}));

  const canvas = document.getElementById('halftoneCanvas');
  const lpiSlider = document.getElementById('lpiSlider');
  lpiSlider.value = '25';
  lpiSlider.dispatchEvent(new Event('input', {bubbles: true}));
  const k25 = canvas.toDataURL('image/png').length;
  lpiSlider.value = '200';
  lpiSlider.dispatchEvent(new Event('input', {bubbles: true}));
  const k200 = canvas.toDataURL('image/png').length;
  lpiSlider.value = '25';
  lpiSlider.dispatchEvent(new Event('input', {bubbles: true}));

  return {
    k25_bytes: k25,
    k200_bytes: k200,
    kModeRespondsToLpi: k25 !== k200,
  };
}
```

Expected: `kModeRespondsToLpi: true`. K mode uses the same `renderInkScreen` → `getDotRadius` → cell pipeline as CMYK, so it should respond.

- [ ] **Step 2: Verify LPI control persists across mode toggles**

```js
() => {
  const lpiSlider = document.getElementById('lpiSlider');
  lpiSlider.value = '120';
  lpiSlider.dispatchEvent(new Event('input', {bubbles: true}));

  document.querySelector('.mode-button[data-mode="cmyk"]').click();
  const afterCmyk = lpiSlider.value;
  document.querySelector('.mode-button[data-mode="single"]').click();
  const afterSingle = lpiSlider.value;

  return {
    lpiAfterCmykToggle: afterCmyk,
    lpiAfterSingleToggle: afterSingle,
    lpiValueDisplay: document.getElementById('lpiValue').textContent,
  };
}
```

Expected: `lpiAfterCmykToggle: "120"`, `lpiAfterSingleToggle: "120"`. The LPI slider's value should not reset when toggling modes.

- [ ] **Step 3: Verify hard reload defaults**

Hard reload the page via `mcp__playwright__browser_navigate` to `http://localhost:4173/`. Run:

```js
() => ({
  lpiSliderValue: document.getElementById('lpiSlider').value,
  lpiValueDisplay: document.getElementById('lpiValue').textContent,
  visualizerHeight: document.querySelector('.visualizer').offsetHeight,
});
```

Expected: `lpiSliderValue: "25"`, `lpiValueDisplay: "25"`, and the visualizer height is reasonable (positive, well under the full viewport). The fresh-load state should match the current pre-feature first-load look.

- [ ] **Step 4: Verify LPI control doesn't break the Dot Gain popover**

```js
() => {
  const trigger = document.getElementById('dotGainTrigger');
  trigger.click();
  const popoverHiddenAfterOpen = document.getElementById('dotGainPopover').hidden;

  // Drag LPI slider while popover is open
  const lpiSlider = document.getElementById('lpiSlider');
  lpiSlider.value = '100';
  lpiSlider.dispatchEvent(new Event('input', {bubbles: true}));
  const popoverStillOpen = !document.getElementById('dotGainPopover').hidden;

  trigger.click(); // close
  lpiSlider.value = '25';
  lpiSlider.dispatchEvent(new Event('input', {bubbles: true}));

  return {
    popoverHiddenAfterOpen,
    popoverStillOpen,
  };
}
```

Expected: `popoverHiddenAfterOpen: false`, `popoverStillOpen: true`. The LPI slider and the Dot Gain popover are independent controls.

- [ ] **Step 5: Stop the dev server and clean up**

Stop the background `npx serve` process. Delete any screenshot files left in the repo root or `.playwright-mcp/` directory. The `.playwright-mcp/` entry in `.gitignore` already covers it, but `rm -rf .playwright-mcp/ *.png` from the repo root is the explicit cleanup.

---

## Notes on running this plan

- **Order matters.** Tasks 1–3 are strictly sequential. Task 4 (tuning) iterates until acceptance. Task 5 (docs) can be done in parallel with Task 4 if you want, but it's small enough to just run last. Task 6 is the final verification + cleanup.
- **Don't speculatively tune.** Run all five LPI checkpoints first. Only adjust constants if a specific checkpoint fails.
- **Each commit should leave the repo in a working state.** If a task feels too big mid-execution, find an internal seam and split it.
- **Visual verification is your test suite.** Use Playwright screenshots at each rendering change; the user's acceptance criteria are visual, not numerical.
- **Don't add a test framework.** The user explicitly chose minimal/zero-build. Behavioral verification via Playwright is the substitute.

When all tasks are complete, the feature is shippable from `main`. Verify nothing is uncommitted, push with `git push origin main`, and GitHub Pages auto-deploys within a minute.
