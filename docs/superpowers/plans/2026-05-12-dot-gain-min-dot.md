# Dot Gain & Min Dot Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a press-physics model (dot gain + min dot) to the halftone visualizer so the left side shows what a file physically prints on the user's press, while the right side becomes a constant GRACoL coated soft-proof reference matching what designers see in Adobe.

**Architecture:** Three new user inputs (`Dot Gain`, `Min Dot`, `Min Dot Printed`) live behind a `<details>` disclosure in the existing control strip. A two-component gain function (`bell`-shaped midtone spread + decaying small-end bloom) maps the slider's *file* coverage to *press-effective* coverage, which drives both dot radius on the halftone view and the rendered ink area. The right side independently applies hardcoded G7/GRACoL coated TVI per channel before the existing Neugebauer mix, so it always represents the Adobe soft-proof reference and doesn't move when the user dials their press.

**Tech Stack:** Vanilla HTML/CSS/JS, no build step, no test framework. Visual & behavioral verification via Playwright MCP and `npx serve`.

**Reference docs:**
- Spec: `docs/superpowers/specs/2026-05-12-dot-gain-min-dot-design.html`
- Background discussion: `docs/superpowers/specs/2026-05-12-dot-gain-min-dot-background.html`

---

## Task 0: Set up the verification harness

This task creates the dev-server / Playwright loop used by every subsequent task. Skip if you already have `npx serve` running on port 4173 and Playwright open.

**Files:**
- No code changes — operational only.

- [ ] **Step 1: Start a local static server on port 4173**

Run (in background, from the repo root):
```sh
npx --yes serve . -l 4173 -n
```

- [ ] **Step 2: Wait until the server returns 200**

Run:
```sh
until curl -fsS http://localhost:4173 >/dev/null 2>&1; do sleep 0.3; done; echo "ready"
```
Expected output: `ready`

- [ ] **Step 3: Open the page in Playwright at a reasonable viewport**

Tools: `mcp__playwright__browser_resize` (1280×900), then `mcp__playwright__browser_navigate` to `http://localhost:4173/`.

Expected: page loads, title is "Halftone Dot Visualizer", only console error is the favicon 404 (ignorable).

---

## Task 1: Add HTML disclosure markup with three number inputs

**Files:**
- Modify: `index.html` (insert a new `<details>` block inside `.controls-panel`, after the `.cmyk-controls` div)

**Why:** Establishes the static structure so styling and JS wiring have something to attach to. No behavior yet — the inputs render but don't affect anything.

- [ ] **Step 1: Open `index.html` and locate the `</div>` that closes `.controls-panel`**

The current structure (around line 121):
```html
        <div class="controls-panel">
          <div class="single-controls" id="singleControls">
            ...
          </div>

          <div class="cmyk-controls" id="cmykControls" hidden>
            ...
          </div>
        </div>
```

You will insert a new `<details>` block as a third child of `.controls-panel`, immediately after `</div>` of `.cmyk-controls` and before the closing `</div>` of `.controls-panel`.

- [ ] **Step 2: Insert the disclosure block**

Add exactly this markup as a sibling of `#singleControls` and `#cmykControls`:

```html
          <details class="dotgain-controls" id="dotGainDetails">
            <summary>Dot Gain</summary>
            <div class="dotgain-inputs">
              <label class="dotgain-input" for="dotGainInput">
                <span>Dot Gain</span>
                <span class="input-row">
                  <input id="dotGainInput" type="number" min="0" max="40" step="1" value="18" />
                  <span class="suffix">%</span>
                </span>
              </label>
              <label class="dotgain-input" for="minDotInput">
                <span>Min Dot</span>
                <span class="input-row">
                  <input id="minDotInput" type="number" min="0" max="15" step="0.5" value="3" />
                  <span class="suffix">%</span>
                </span>
              </label>
              <label class="dotgain-input" for="minDotPrintedInput">
                <span>Min Dot Printed</span>
                <span class="input-row">
                  <input id="minDotPrintedInput" type="number" min="0" max="25" step="0.5" value="5" />
                  <span class="suffix">%</span>
                </span>
              </label>
            </div>
          </details>
```

- [ ] **Step 3: Reload the page in Playwright and verify the disclosure renders**

Navigate to `http://localhost:4173/`. Use `mcp__playwright__browser_evaluate` with:

```js
() => {
  const details = document.getElementById('dotGainDetails');
  const summary = details?.querySelector('summary');
  return {
    detailsExists: !!details,
    summaryText: summary?.textContent.trim(),
    inputs: {
      dotGain: document.getElementById('dotGainInput')?.value,
      minDot: document.getElementById('minDotInput')?.value,
      minDotPrinted: document.getElementById('minDotPrintedInput')?.value,
    }
  };
}
```

Expected output:
```json
{
  "detailsExists": true,
  "summaryText": "Dot Gain",
  "inputs": { "dotGain": "18", "minDot": "3", "minDotPrinted": "5" }
}
```

- [ ] **Step 4: Commit**

```sh
git add index.html
git commit -m "Add Dot Gain disclosure markup with three number inputs

Three new inputs (Dot Gain, Min Dot, Min Dot Printed) inside a closed
<details> disclosure in the control strip. Markup-only; styling and
behavior land in subsequent commits.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 2: Style the disclosure and number inputs

**Files:**
- Modify: `styles.css` (append new rules at the end of the file, before the `@media (max-width: 760px)` block to keep media queries grouped)

**Why:** Make the new controls match the existing print-shop aesthetic and behave responsively. After this task the page looks intentional even though the inputs still don't drive any rendering.

- [ ] **Step 1: Open `styles.css` and locate the `@media (max-width: 760px)` block**

Around line 286. You will insert the new dot-gain rules *before* this media query.

- [ ] **Step 2: Insert the dot-gain CSS rules**

Add exactly these rules just before the `@media (max-width: 760px)` block:

```css
.dotgain-controls {
  margin-top: 14px;
  padding-top: 14px;
  border-top: 1px dashed var(--line);
}

.dotgain-controls summary {
  cursor: pointer;
  list-style: none;
  display: inline-flex;
  align-items: center;
  gap: 8px;
  font-size: 0.74rem;
  font-weight: 900;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: #51493d;
  user-select: none;
}

.dotgain-controls summary::-webkit-details-marker { display: none; }

.dotgain-controls summary::before {
  content: "\25B8";
  display: inline-block;
  font-size: 0.7rem;
  color: var(--accent);
  transition: transform 120ms ease;
}

.dotgain-controls[open] summary::before {
  transform: rotate(90deg);
}

.dotgain-controls summary:hover { color: var(--ink); }

.dotgain-inputs {
  margin-top: 12px;
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 10px clamp(12px, 2vw, 22px);
}

.dotgain-input {
  display: grid;
  gap: 6px;
  min-width: 0;
  font-size: 0.74rem;
  font-weight: 900;
  letter-spacing: 0.06em;
  text-transform: uppercase;
  color: #51493d;
}

.dotgain-input .input-row {
  display: flex;
  align-items: center;
  gap: 6px;
  border: 1px solid var(--line);
  border-radius: 6px;
  background: var(--panel);
  padding: 6px 10px;
}

.dotgain-input input[type="number"] {
  flex: 1;
  border: 0;
  background: transparent;
  font: inherit;
  font-size: 1rem;
  font-weight: 700;
  font-variant-numeric: tabular-nums;
  color: var(--ink);
  width: 100%;
  outline: none;
  text-transform: none;
  -moz-appearance: textfield;
}

.dotgain-input .suffix {
  color: #51493d;
  font-weight: 700;
  font-size: 0.86rem;
}
```

- [ ] **Step 3: Update the existing `@media (max-width: 760px)` block to collapse the input grid**

Find the existing rule:

```css
@media (max-width: 760px) {
  .app-shell {
    min-height: 600px;
  }
  ...
}
```

Add this rule inside that same media query block (and inside the existing `@media (max-width: 430px)` block too):

For `@media (max-width: 760px)`:
```css
  .dotgain-inputs {
    grid-template-columns: 1fr 1fr;
  }
```

For `@media (max-width: 430px)`:
```css
  .dotgain-inputs {
    grid-template-columns: 1fr;
  }
```

- [ ] **Step 4: Reload and visually verify styling**

Navigate to `http://localhost:4173/`. Click the disclosure summary to expand. Take a screenshot.

```
mcp__playwright__browser_evaluate:
() => document.getElementById('dotGainDetails').open = true
```

Then `mcp__playwright__browser_take_screenshot` with filename `task2-styled.png`. Read the screenshot back and verify:
- The summary triangle rotates 90° when open
- Three inputs are in a row, each with a tan-bordered "card" appearance matching the existing channel-control cells
- Values `18`, `3`, `5` are visible with `%` suffix
- A dashed line separates the disclosure from the channel controls above

- [ ] **Step 5: Commit**

```sh
git add styles.css
git commit -m "Style Dot Gain disclosure to match print-shop aesthetic

Adds rules for the disclosure triangle, three-column input grid, and
input-row card styling. Mobile breakpoints collapse to two- then
single-column. No behavior change yet.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 3: Wire the inputs into JS state (read-only, no rendering effect)

**Files:**
- Modify: `app.js` (add input element references near the top, add a sync/redraw listener)

**Why:** Establishes the JS bindings and confirms that changes to the inputs trigger redraws — but the actual gain math comes in later tasks. After this task the inputs cause `drawVisualizer()` to be called on each `change` event, and you can read their current values from JS.

- [ ] **Step 1: Add element references near the other DOM-ref lines**

Locate the block at the top of `app.js` (lines 1-10) that references DOM elements:

```js
const canvas = document.querySelector("#halftoneCanvas");
const controlStrip = document.querySelector("#controlStrip");
const modeTitle = document.querySelector("#modeTitle");
const modeButtons = document.querySelectorAll(".mode-button");
const singleMeter = document.querySelector("#singleMeter");
const singleControls = document.querySelector("#singleControls");
const cmykControls = document.querySelector("#cmykControls");
const singleSlider = document.querySelector("#coverageSlider");
const singleValue = document.querySelector("#coverageValue");
const ctx = canvas.getContext("2d");
```

Immediately after this block, add:

```js
const dotGainInput = document.querySelector("#dotGainInput");
const minDotInput = document.querySelector("#minDotInput");
const minDotPrintedInput = document.querySelector("#minDotPrintedInput");
```

- [ ] **Step 2: Add a helper that reads the three values as fractions in [0, 1]**

After the existing helpers near the top of `app.js` (after `let singleCoverage = ...`), add:

```js
function getDotGainParams() {
  return {
    dotGain: Math.max(0, Number(dotGainInput.value) / 100) || 0,
    minDot: Math.max(0, Number(minDotInput.value) / 100) || 0,
    minDotPrinted: Math.max(0, Number(minDotPrintedInput.value) / 100) || 0,
  };
}
```

Note: the `|| 0` guards against `NaN` from a temporarily empty input.

- [ ] **Step 3: Wire `change` event listeners to trigger a redraw**

At the bottom of `app.js`, near the other event listeners (after the `inkScreens.forEach` slider listener block, before `window.addEventListener("resize", ...)`), add:

```js
[dotGainInput, minDotInput, minDotPrintedInput].forEach((input) => {
  input.addEventListener("change", () => {
    drawVisualizer();
  });
});
```

We use `change` (fires on blur/Enter), not `input` (which would fire on every keystroke and cause jank while typing multi-digit values).

- [ ] **Step 4: Verify the state and listeners are wired**

In Playwright, navigate to `http://localhost:4173/` and run:

```js
() => {
  // change-event listeners only fire on commit, so we set value + dispatch change
  document.getElementById('dotGainInput').value = '25';
  document.getElementById('dotGainInput').dispatchEvent(new Event('change', {bubbles: true}));
  return {
    dotGainInputValue: document.getElementById('dotGainInput').value,
    // The function below should exist now:
    paramsAvailable: typeof getDotGainParams === 'undefined' ? 'no' : 'yes',
  };
}
```

Note: `getDotGainParams` won't be visible in the global scope from outside the script (it's module-scoped to the script tag). Instead verify by checking that the `change` event didn't throw — the page should still render without console errors. Check via `mcp__playwright__browser_console_messages` level `error`.

Expected: no new console errors. Page still renders normally.

- [ ] **Step 5: Commit**

```sh
git add app.js
git commit -m "Wire Dot Gain inputs to JS state + redraw on change

Adds module-level references to the three new inputs, a
getDotGainParams() reader, and change-event listeners that call
drawVisualizer() on commit (blur/Enter). The gain math itself
arrives in the next task; this commit just establishes the binding.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 4: Implement the gain-model helper functions

**Files:**
- Modify: `app.js` (add four pure functions: `bell`, `bloomDecay`, `pressEffectiveCoverage`, `pressEffectiveAmount`)

**Why:** These are the math primitives used by Tasks 5 and 6. They're pure (no side effects) and can be verified independently before integration. Adding them in their own commit makes the diff easy to read.

- [ ] **Step 1: Locate the existing math helpers in `app.js`**

After `function smoothstep(start, end, current) { ... }` (around line 118-122), you'll add the new helpers.

- [ ] **Step 2: Add the four gain-model functions**

Immediately after `smoothstep`, add:

```js
function bell(c) {
  return 4 * c * (1 - c);
}

function bloomDecay(c, minDotFrac) {
  const range = 0.5 - minDotFrac;

  if (range <= 0) {
    return 0;
  }

  const t = (c - minDotFrac) / range;

  return Math.max(0, Math.min(1, 1 - t));
}

function pressEffectiveCoverage(cFrac, params) {
  if (cFrac <= 0) {
    return 0;
  }

  if (cFrac < params.minDot) {
    return 0;
  }

  if (cFrac >= 1) {
    return 1;
  }

  const bloomAmplitude = Math.max(0, params.minDotPrinted - params.minDot);
  const bloomTerm = bloomAmplitude * bloomDecay(cFrac, params.minDot);
  const spreadTerm = params.dotGain * bell(cFrac);

  return Math.min(1, cFrac + bloomTerm + spreadTerm);
}

function pressEffectiveAmount(rawAmount) {
  const cFrac = rawAmount / 100;
  const effective = pressEffectiveCoverage(cFrac, getDotGainParams());

  return effective * 100;
}
```

`pressEffectiveCoverage` is the pure mathematical core (takes a fraction, returns a fraction). `pressEffectiveAmount` is the convenience wrapper that matches the existing `amount` convention (0-100) and reads from the live inputs.

- [ ] **Step 3: Verify the math at known anchor points**

The functions are declared at the top level of a non-module `<script>`, so they're on `window` and directly callable from `page.evaluate`. No debug exposure needed.

In Playwright at `http://localhost:4173/`, run:

```js
() => {
  // Set inputs to canonical defaults: 18/3/5 (no change event needed — the
  // functions read .value directly via getDotGainParams())
  document.getElementById('dotGainInput').value = '18';
  document.getElementById('minDotInput').value = '3';
  document.getElementById('minDotPrintedInput').value = '5';
  return {
    belowMin:   pressEffectiveAmount(2),       // < minDot → 0
    atMin:      pressEffectiveAmount(3),       // minDot anchor + small spread term
    midtone:    pressEffectiveAmount(50),      // 50 + 18 = 68
    solid:      pressEffectiveAmount(100),     // 100 (no gain at top)
    bellAtHalf:  bell(0.5),                    // 1
    decayAtMin:  bloomDecay(0.03, 0.03),       // 1
    decayAtHalf: bloomDecay(0.5, 0.03),        // 0
  };
}
```

Expected output (exact values for the closed-form math):

```json
{
  "belowMin": 0,
  "atMin": 7.095...,    // 3 + (5-3)*1 + 18*bell(0.03) ≈ 7.095
  "midtone": 68,        // 50 + 18*bell(0.5) = 50 + 18
  "solid": 100,
  "bellAtHalf": 1,
  "decayAtMin": 1,
  "decayAtHalf": 0
}
```

Note: `atMin` is not exactly 5 because at c=minDot=0.03 the spread term still contributes `18 * bell(0.03) ≈ 2.1` on top of the bloom term `(5-3)*1 = 2`. Acceptable range: between 5 and 9 inclusive. The anchor `effective(minDot) ≈ minDotPrinted` is approximate, not exact — and that's the design (the spec calls this out).

If any value is off by more than rounding, STOP and inspect the math you wrote in Step 2.

- [ ] **Step 4: Commit**

```sh
git add app.js
git commit -m "Add gain-model helper functions (bell, bloomDecay, pressEffective)

Pure math primitives that compute press-effective coverage from raw
file coverage and the three user knobs. Hard step at min dot; bloom
term decays from min-dot to midtone; spread term peaks at midtone.
No rendering integration yet.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 5: Apply press-effective coverage to the left-side rendering

**Files:**
- Modify: `app.js` — change `renderInkScreen` to use `pressEffectiveAmount` for both the `amount <= 0` and `amount >= 100` early-returns and for `getDotRadius`. Also update the meter labels behavior (no change to displayed value; meter still shows raw slider).

**Why:** This is the user-visible behavior change for the halftone view. The dot radius math is unchanged — we just feed it press-effective coverage instead of the raw slider value.

- [ ] **Step 1: Locate `renderInkScreen` in `app.js`**

Around lines 199-241 (after the offscreen-canvas refactor commit). The current first lines:

```js
function renderInkScreen(screen, clip, width, height, cell) {
  const amount = Number(screen.slider?.value ?? screen.amount);

  if (amount <= 0) {
    return null;
  }
  ...
```

- [ ] **Step 2: Apply press-effective coverage at the top of `renderInkScreen`**

Change the opening of `renderInkScreen` to:

```js
function renderInkScreen(screen, clip, width, height, cell) {
  const rawAmount = Number(screen.slider?.value ?? screen.amount);
  const amount = pressEffectiveAmount(rawAmount);

  if (amount <= 0) {
    return null;
  }
```

The rest of the function (offscreen-canvas creation, clip, fill, dot loop) is unchanged. The variable `amount` now carries press-effective values; the existing `getDotRadius(cell, amount)` call automatically benefits.

- [ ] **Step 3: Verify at defaults — left side now shows realistic gain**

In Playwright, ensure inputs are at defaults (18 / 3 / 5), switch to CMYK mode with C=50% and other channels 0, take a screenshot.

Set state:
```js
() => {
  document.getElementById('dotGainInput').value = '18';
  document.getElementById('minDotInput').value = '3';
  document.getElementById('minDotPrintedInput').value = '5';
  document.querySelector('.mode-button[data-mode="cmyk"]').click();
  for (const [ch, v] of [['cyan','50'],['magenta','0'],['yellow','0'],['black','0']]) {
    const s = document.getElementById(ch+'Slider');
    s.value = v;
    s.dispatchEvent(new Event('input', {bubbles: true}));
  }
  return 'set';
}
```

Take screenshot `task5-c50-defaults.png`. Read it back. Verify:
- The left side shows cyan dots that visually correspond to roughly 68% (50% + 18% gain) coverage — noticeably larger than 50% nominal.
- The right side has NOT yet changed (still raw Murray-Davies, will be fixed in Task 6).

Quick numerical check:
```js
() => window.__dotGainAPI ? null : 'no debug api — that is expected after Task 4 step 4'
```

(If you want a numerical readback during this task only, you can temporarily re-add the debug API line.)

- [ ] **Step 4: Verify min-dot hard step kicks in below threshold**

Set state:
```js
() => {
  document.getElementById('minDotInput').value = '5';
  document.getElementById('minDotInput').dispatchEvent(new Event('change', {bubbles: true}));
  document.getElementById('cyanSlider').value = '3';
  document.getElementById('cyanSlider').dispatchEvent(new Event('input', {bubbles: true}));
  return 'cyan slider = 3, min dot = 5';
}
```

Take screenshot `task5-mindot-hardstep.png`. Read it back. Verify:
- Left side shows NO cyan dots (3% < 5% min, dropped to 0).
- The other channels still render normally.

- [ ] **Step 5: Reset to defaults and screenshot for sanity**

```js
() => {
  document.getElementById('dotGainInput').value = '18';
  document.getElementById('dotGainInput').dispatchEvent(new Event('change'));
  document.getElementById('minDotInput').value = '3';
  document.getElementById('minDotInput').dispatchEvent(new Event('change'));
  document.getElementById('minDotPrintedInput').value = '5';
  document.getElementById('minDotPrintedInput').dispatchEvent(new Event('change'));
  document.getElementById('cyanSlider').value = '50';
  document.getElementById('cyanSlider').dispatchEvent(new Event('input', {bubbles: true}));
  return 'reset';
}
```

Take `task5-reset-c50.png`. Confirms first-load defaults render coherently.

- [ ] **Step 6: Commit**

```sh
git add app.js
git commit -m "Apply press-effective coverage to halftone rendering

renderInkScreen now feeds pressEffectiveAmount(rawAmount) through to
getDotRadius and the early-return checks. The hard min-dot floor zeroes
file values below the threshold; bloom + spread enlarge dots above it.
Right side still uses raw Murray-Davies — fixed in next commit.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 6: Apply GRACoL coated TVI to the right-side reference

**Files:**
- Modify: `app.js` — add GRACoL TVI constants, `applyGracolTvi` function, `getGracolReferenceCoverages` helper; change `getProfiledCmykToneColor` to use the new helper; change `getSingleToneColor` to apply K's TVI.

**Why:** The right side should always show what designers see in Adobe soft-proof against GRACoL2013 CRPC6 coated. Without this change the right side is unrealistically clean. After this change, at default settings, left and right halves visually agree (both reflect ~coated gain); the two halves diverge only as the user dials toward flexo/newsprint on the left.

- [ ] **Step 1: Add GRACoL TVI constants near the top of `app.js`**

After the `gracolNeugebauerRgb` table (lines 12-30) and before the `inkScreens` array, add:

```js
// Channel-order index for inkScreens: 0=C, 1=M, 2=Y, 3=K
const gracolChannelOrder = ["c", "m", "y", "k"];

// G7/GRACoL coated #1 target TVI: extra coverage at 50% file value, per channel.
const gracolCoatedTviAt50 = {
  c: 0.12,
  m: 0.14,
  y: 0.13,
  k: 0.18,
};
```

- [ ] **Step 2: Add `applyGracolTvi` near the other gain helpers**

Just after the `pressEffectiveAmount` function from Task 4, add:

```js
function applyGracolTvi(cFrac, channel) {
  if (cFrac <= 0) {
    return 0;
  }

  if (cFrac >= 1) {
    return 1;
  }

  const tviAt50 = gracolCoatedTviAt50[channel] ?? 0;

  return Math.min(1, cFrac + tviAt50 * bell(cFrac));
}
```

- [ ] **Step 3: Add `getGracolReferenceCoverages` adjacent to `getScreenCoverages`**

Locate `function getScreenCoverages()` (around line 171). Immediately after it, add:

```js
function getGracolReferenceCoverages() {
  return inkScreens.map((screen, index) => {
    const raw = Number(screen.slider.value) / 100;

    return applyGracolTvi(raw, gracolChannelOrder[index]);
  });
}
```

- [ ] **Step 4: Switch `getProfiledCmykToneColor` to the GRACoL reference**

Find the line in `getProfiledCmykToneColor`:

```js
const coverages = getScreenCoverages();
```

Change it to:

```js
const coverages = getGracolReferenceCoverages();
```

Everything else in that function is unchanged. The 16 Neugebauer primaries already represent GRACoL2013 CRPC6 sampled colors — we're just feeding TVI-adjusted coverages into the existing math.

- [ ] **Step 5: Apply K's TVI to the single-K right side**

Find `function getSingleToneColor()` (around line 161):

```js
function getSingleToneColor() {
  const amount = singleCoverage / 100;
  const red = blendChannel(255, 16, amount);
  const green = blendChannel(250, 16, amount);
  const blue = blendChannel(240, 16, amount);

  return `rgb(${red}, ${green}, ${blue})`;
}
```

Change the body to apply GRACoL K TVI:

```js
function getSingleToneColor() {
  const amount = applyGracolTvi(singleCoverage / 100, "k");
  const red = blendChannel(255, 16, amount);
  const green = blendChannel(250, 16, amount);
  const blue = blendChannel(240, 16, amount);

  return `rgb(${red}, ${green}, ${blue})`;
}
```

- [ ] **Step 6: Verify the right side now shows GRACoL-gain output**

In Playwright at defaults (18/3/5), CMYK mode, C=50% only:

Set state:
```js
() => {
  document.getElementById('dotGainInput').value = '18';
  document.getElementById('dotGainInput').dispatchEvent(new Event('change'));
  document.getElementById('minDotInput').value = '3';
  document.getElementById('minDotInput').dispatchEvent(new Event('change'));
  document.getElementById('minDotPrintedInput').value = '5';
  document.getElementById('minDotPrintedInput').dispatchEvent(new Event('change'));
  document.querySelector('.mode-button[data-mode="cmyk"]').click();
  for (const [ch, v] of [['cyan','50'],['magenta','0'],['yellow','0'],['black','0']]) {
    const s = document.getElementById(ch+'Slider');
    s.value = v;
    s.dispatchEvent(new Event('input', {bubbles: true}));
  }
  return 'set';
}
```

Take screenshot `task6-c50-with-gracol.png`. Read it back. Verify:
- Right-side cyan tone is now noticeably more saturated/darker than before this task (12% TVI applied to C at 50%).
- Left-side and right-side cyans are now visually close to each other — both reflect realistic gain.

- [ ] **Step 7: Switch to single-K mode and verify K TVI**

Set state:
```js
() => {
  document.querySelector('.mode-button[data-mode="single"]').click();
  const s = document.getElementById('coverageSlider');
  s.value = '50';
  s.dispatchEvent(new Event('input', {bubbles: true}));
  return 'K=50';
}
```

Take screenshot `task6-k50-with-gracol.png`. Read it back. Verify:
- Right side is darker than before this task (K gets +18% at midtone).
- Left and right look visually close.

- [ ] **Step 8: Commit**

```sh
git add app.js
git commit -m "Apply GRACoL coated TVI to right-side reference

Per-channel TVI (C+12, M+14, Y+13, K+18 at 50%) modulates raw coverage
before the existing Murray-Davies/Neugebauer mix. Same TVI applies to
the single-K right side. The right half now matches what Adobe soft-
proof shows against GRACoL2013 CRPC6 coated — a corrective fix to a
prior inaccuracy as well as the intended pedagogical reference.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

---

## Task 7: End-to-end behavioral verification

**Files:**
- No code changes — verification only. May discover defects; if so, fix in this task with additional commits.

**Why:** Confirm the feature works as a system across the modes, default values, edge cases, and the "dial toward flexo, watch left diverge from right" pedagogical scenario.

- [ ] **Step 1: Default-state coherence at midtone**

In Playwright, reset everything to defaults, CMYK mode, all four channels at 50%:

```js
() => {
  document.getElementById('dotGainInput').value = '18';
  document.getElementById('dotGainInput').dispatchEvent(new Event('change'));
  document.getElementById('minDotInput').value = '3';
  document.getElementById('minDotInput').dispatchEvent(new Event('change'));
  document.getElementById('minDotPrintedInput').value = '5';
  document.getElementById('minDotPrintedInput').dispatchEvent(new Event('change'));
  document.querySelector('.mode-button[data-mode="cmyk"]').click();
  for (const ch of ['cyan','magenta','yellow','black']) {
    const s = document.getElementById(ch+'Slider');
    s.value = '50';
    s.dispatchEvent(new Event('input', {bubbles: true}));
  }
  return 'ready';
}
```

Take `task7-defaults-cmyk50.png`. Verify left and right halves are visually coherent (similar grayscale-ish dark color); divider is clearly visible.

- [ ] **Step 2: Flexo-like settings — left should diverge from right**

```js
() => {
  document.getElementById('dotGainInput').value = '28';
  document.getElementById('dotGainInput').dispatchEvent(new Event('change'));
  document.getElementById('minDotInput').value = '5';
  document.getElementById('minDotInput').dispatchEvent(new Event('change'));
  document.getElementById('minDotPrintedInput').value = '18';
  document.getElementById('minDotPrintedInput').dispatchEvent(new Event('change'));
  return 'flexo';
}
```

Take `task7-flexo-cmyk50.png`. Verify:
- Left side is visibly darker / more saturated than right (the press is amplifying coverage; the GRACoL coated reference on the right is unchanged).
- Both halves still look like sensible CMYK process colors (no artifacts).

- [ ] **Step 3: Set channel below minDot to confirm hard step persists**

```js
() => {
  document.getElementById('cyanSlider').value = '4';
  document.getElementById('cyanSlider').dispatchEvent(new Event('input', {bubbles: true}));
  return 'C=4, M=Y=K=50';
}
```

Take `task7-flexo-c-below-min.png`. Verify:
- Left side shows NO cyan dots in the cyan screen at all (4% < 5% min).
- Right side still has cyan in the mix (right side is GRACoL-coated, doesn't know about user's flexo min dot).
- This is exactly the pedagogical contract — "your file says 4% cyan; on flexo with 5% min dot, that drops out entirely; on the reference GRACoL coated press you'd see it."

- [ ] **Step 4: Switch to single-K mode and verify the disclosure still works**

```js
() => {
  document.querySelector('.mode-button[data-mode="single"]').click();
  document.getElementById('coverageSlider').value = '70';
  document.getElementById('coverageSlider').dispatchEvent(new Event('input', {bubbles: true}));
  return 'K mode, 70';
}
```

Take `task7-flexo-k70.png`. Verify:
- Single-K mode renders without errors.
- Dot Gain disclosure remains visible and the inputs retain their flexo values (28/5/18).
- Left side shows clearly larger/more-overlapping dots than nominal 70% would suggest.
- Right side shows a corresponding dark tone (K TVI applied).

- [ ] **Step 5: Verify the closed-disclosure default first-load behavior**

Reload the page (hard refresh — clear caches in Playwright):

```js
mcp__playwright__browser_navigate to http://localhost:4173/
```

Verify:
- The Dot Gain disclosure is CLOSED on first load (the user has to click "Dot Gain" to expand).
- All three inputs default to 18 / 3 / 5.
- The rest of the visualizer renders normally — no console errors.

- [ ] **Step 6: Open the disclosure and confirm typing into an input doesn't redraw mid-keystroke**

```js
mcp__playwright__browser_evaluate:
() => {
  document.getElementById('dotGainDetails').open = true;
  const inp = document.getElementById('dotGainInput');
  inp.focus();
  inp.value = '';
  // Synthesize three keystrokes via input events (which we DON'T listen to)
  inp.dispatchEvent(new Event('input', {bubbles: true}));
  inp.value = '2';
  inp.dispatchEvent(new Event('input', {bubbles: true}));
  inp.value = '25';
  inp.dispatchEvent(new Event('input', {bubbles: true}));
  return 'typed; no change event yet';
}
```

This should NOT trigger a redraw. Confirm by checking the canvas pixel data is unchanged from before the typing. Then commit:

```js
() => {
  document.getElementById('dotGainInput').dispatchEvent(new Event('change', {bubbles: true}));
  return 'committed';
}
```

Take `task7-after-commit.png`. Now the redraw should have happened, and the dot gain reflects 25.

- [ ] **Step 7: Update the README to advertise the new controls**

Modify `README.md`. Find the "Two modes" section:

```md
Two modes:

- **K** — single-channel black, one slider for dot coverage (0–100%).
- **CMYK** — four channels with independent coverage sliders, each rendered at its traditional screen angle and blended with a printing-industry color profile.
```

Replace with:

```md
Two modes:

- **K** — single-channel black, one slider for dot coverage (0–100%).
- **CMYK** — four channels with independent coverage sliders, each rendered at its traditional screen angle and blended with a printing-industry color profile.

A **Dot Gain** disclosure in the control strip exposes three press-physics knobs (Dot Gain, Min Dot, Min Dot Printed). The left side reflects what files actually print like on your press; the right side stays a constant GRACoL coated soft-proof reference matching what designers see in Adobe.
```

Then find the "How it works" bullet list and add a fifth bullet after the "Rendering, with accurate high-coverage dots" bullet:

```md
- **Press-physics gain model.** The Dot Gain disclosure runs each channel's slider value through a hard min-dot floor and a two-component additive gain (`bell`-shape midtone spread + `decay`-shape small-end bloom) anchored by the three user inputs. The right-side reference independently applies hardcoded G7/GRACoL coated TVI per channel (C+12, M+14, Y+13, K+18 at 50%) before the Neugebauer mix, so it always represents the Adobe soft-proof equivalent.
```

- [ ] **Step 8: Final commit**

```sh
git add README.md
git commit -m "Document Dot Gain disclosure in README

Adds a short description of the three knobs and the two-side
pedagogical contract to the existing 'Two modes' and 'How it works'
sections.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"
```

- [ ] **Step 9: Tear down the verification harness**

Stop the background `npx serve` process, close the Playwright tab, delete any screenshot files left in the repo root or `.playwright-mcp/` directory. The `.playwright-mcp/` entry in `.gitignore` already covers this, but `rm -rf .playwright-mcp/ *.png` in the repo root is the explicit cleanup.

---

## Notes on running this plan

- **Order matters.** Tasks 1–6 are strictly sequential. Each builds on state from the previous. Task 7 verifies the whole.
- **Each commit should leave the repo in a working state.** If a task feels too big mid-execution, find an internal seam and split it; don't leave the working tree half-broken.
- **Run the Playwright verification on every rendering task.** Visual regressions are easy to miss in this codebase because there's no test suite — your eyes are the test suite.
- **Don't add a test framework.** The user explicitly chose minimal/zero-build. Behavioral verification via Playwright is the substitute.
- **Don't push to the upstream PR branch (`rendering-fix-and-docs`).** This feature belongs on `main` only. Verify with `git branch --show-current` before committing.

When all tasks are complete, the feature is shippable from `main`. If you want it on GitHub Pages, just push — Pages auto-deploys from `main`/root.
