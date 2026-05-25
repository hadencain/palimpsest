# PALIMPSEST.EXE v2 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add text visibility improvements, a toggleable full-control audio-reactive mode, and a loop mode to the existing `palimpsest.html`.

**Architecture:** Single file modification — all changes are in `palimpsest.html`. Three independent features applied in order: CSS text pass, loop mode (state + HTML + JS), AR mode (state + HTML + CSS + per-frame logic). No new files, no dependencies.

**Tech Stack:** HTML5, CSS, vanilla JS, Web Audio API (existing), HTMLVideoElement.playbackRate (new usage)

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `palimpsest.html` | Modify | All three features |

---

### Task 1: Text visibility — CSS

**Files:**
- Modify: `palimpsest.html` — `<style>` block only

- [ ] **Step 1: Update all color and font-size values in the CSS block**

Read `palimpsest.html`. Make the following CSS changes. Each is a discrete find-and-replace — do them all in one edit pass, reading the file first to confirm exact strings.

**Colors to change:**

```css
/* .status — was color: #444 */
.status { font-size: 12px; letter-spacing: 0.12em; color: #fff; }

/* .status span — was color: #888 */
.status span { color: #fff; }

/* .title — was font-size: 11px */
.title { font-size: 13px; letter-spacing: 0.22em; color: #fff; text-transform: uppercase; }

/* .canvas-hint — leave color: #1e1e1e unchanged (intentional ghost) */
/* .canvas-hint font-size: 10px → 12px */
.canvas-hint {
  position: absolute; inset: 0;
  display: flex; align-items: center; justify-content: center;
  font-size: 12px; letter-spacing: 0.18em; text-transform: uppercase; color: #1e1e1e;
  pointer-events: none;
}

/* .ltag — was font-size: 8px */
.ltag {
  font-size: 10px; letter-spacing: 0.14em; text-transform: uppercase;
  padding: 2px 6px; border: 1px solid;
}
/* layer tag graduated colors — lift all */
.ltag.p2 { color: #555; border-color: #444; }
.ltag.p1 { color: #777; border-color: #555; }
.ltag.now{ color: #fff; border-color: #aaa; }
.ltag.f1 { color: #777; border-color: #555; }
.ltag.f2 { color: #555; border-color: #444; }

/* .timecode — was font-size: 10px, color: #333 */
.timecode {
  position: absolute; bottom: 10px; left: 12px;
  font-size: 12px; letter-spacing: 0.12em; color: #aaa;
}

/* .bar — was background: #2a2a2a */
.bar { width: 100%; background: #444; transition: height 0.05s; }
/* .bar.now — was background: #777 */
.bar.now { background: #aaa; }

/* .vlabel — was font-size: 8px, color: #333 */
.vlabel { font-size: 10px; letter-spacing: 0.12em; text-transform: uppercase; color: #777; }
/* .vlabel.now — was color: #777 */
.vlabel.now { color: #fff; }

/* .ctrl-label — was font-size: 8px, color: #3a3a3a */
.ctrl-label { font-size: 10px; letter-spacing: 0.14em; text-transform: uppercase; color: #888; white-space: nowrap; }

/* .ctrl-value — was color: #bbb */
.ctrl-value { font-size: 14px; color: #fff; min-width: 38px; text-align: right; }

/* .ctrl-range::-webkit-slider-thumb — was background: #777 */
.ctrl-range::-webkit-slider-thumb {
  -webkit-appearance: none; width: 7px; height: 7px;
  background: #aaa; cursor: pointer; border-radius: 0;
}

/* .btn — was font-size: 9px, color: #444 */
.btn {
  background: none; border: 1px solid #1e1e1e; color: #aaa;
  font-family: inherit; font-size: 11px; letter-spacing: 0.14em;
  padding: 3px 10px; cursor: pointer; text-transform: uppercase;
}
.btn:hover { border-color: #666; color: #fff; }
/* .btn.active — was color: #ccc */
.btn.active { border-color: #666; color: #fff; background: #141414; }

/* .dlabel — was font-size: 8px, color: #2a2a2a */
.dlabel { font-size: 10px; letter-spacing: 0.14em; text-transform: uppercase; color: #777; }
/* .dlabel.now — was color: #666 */
.dlabel.now { color: #fff; }

/* .dval — was font-size: 9px, color: #444 */
.dval { font-size: 11px; letter-spacing: 0.08em; color: #888; }
/* .dval.now — was color: #777 */
.dval.now { color: #fff; }
```

- [ ] **Step 2: Verify in browser**

Open `palimpsest.html`. Expected:
- Header title and status text clearly legible white
- Layer stack tags visible with NOW brightest, past/future at readable gray
- Visualizer labels visible, NOW label white
- Control labels, values, and buttons clearly readable
- Debug panel (click DEBUG) shows white/light text
- Canvas drop hint "DROP VIDEO FILE TO BEGIN" still nearly invisible (correct)

- [ ] **Step 3: Commit**

```bash
git -C "C:\Users\haden\Documents\Ship\src\audioProjects\palimpsest" add palimpsest.html
git -C "C:\Users\haden\Documents\Ship\src\audioProjects\palimpsest" commit -m "feat: lift text visibility — white unless functionally graduated"
```

---

### Task 2: Loop mode — state, HTML, JS

**Files:**
- Modify: `palimpsest.html` — STATE section, footer HTML, UI section, `renderFrame()`

- [ ] **Step 1: Add `loopMode` to STATE section**

Find this line in the STATE section:
```js
let isPlaying   = false;
```

Add after it:
```js
let loopMode    = false;
```

- [ ] **Step 2: Add LOOP button to footer HTML**

Find:
```html
    <button class="btn active" id="btn-load">LOAD FILE</button>
    <div class="fsep"></div>
```

Replace with:
```html
    <button class="btn active" id="btn-load">LOAD FILE</button>
    <button class="btn" id="btn-loop">LOOP</button>
    <div class="fsep"></div>
```

- [ ] **Step 3: Modify `renderFrame()` end condition**

Find this block in `renderFrame()`:
```js
  // End condition
  if (masterTime >= duration && duration > 0) {
    pausePlayback();
    setStatus('DONE');
    return;
  }
```

Replace with:
```js
  // End condition
  if (masterTime >= duration && duration > 0) {
    if (loopMode) {
      const active = getActiveLayers();
      active.forEach(({ i, offset }) => {
        if (i === 2) return;
        layers[i].currentTime = Math.max(0, Math.min(offset, duration));
      });
      layers[2].currentTime = 0;
    } else {
      pausePlayback();
      setStatus('DONE');
      return;
    }
  }
```

- [ ] **Step 4: Wire LOOP button in UI section**

Append after the existing `btn-record` event listener in the UI section:
```js
// --- Loop toggle ---
document.getElementById('btn-loop').addEventListener('click', () => {
  loopMode = !loopMode;
  document.getElementById('btn-loop').classList.toggle('active', loopMode);
});
```

- [ ] **Step 5: Verify in browser**

Load a short video. Enable LOOP (button shows active). Play to near the end. Expected:
- Video loops back to beginning seamlessly
- Status stays PLAYING through the loop point
- LOOP button stays active
- Disable LOOP mid-playback — next end-of-video stops normally

- [ ] **Step 6: Commit**

```bash
git -C "C:\Users\haden\Documents\Ship\src\audioProjects\palimpsest" add palimpsest.html
git -C "C:\Users\haden\Documents\Ship\src\audioProjects\palimpsest" commit -m "feat: loop mode — seamless video loop with layer position reset"
```

---

### Task 3: AR mode — state, HTML, CSS, button wiring

**Files:**
- Modify: `palimpsest.html` — STATE section, `<style>` block, footer HTML, UI section

- [ ] **Step 1: Add `arMode` to STATE section**

Find:
```js
let loopMode    = false;
```

Add after it:
```js
let arMode      = false;
```

- [ ] **Step 2: Add `.ctrl.disabled` CSS rule**

Find this rule in the `<style>` block:
```css
.ctrl:last-child { border-right: none; }
```

Add after it:
```css
.ctrl.disabled { opacity: 0.35; pointer-events: none; }
```

- [ ] **Step 3: Add AR button to footer HTML**

Find:
```html
    <button class="btn" id="btn-debug">DEBUG</button>
```

Replace with:
```html
    <button class="btn" id="btn-ar">AR</button>
    <button class="btn" id="btn-debug">DEBUG</button>
```

- [ ] **Step 4: Wire AR button in UI section**

Append after the loop toggle listener:
```js
// --- AR mode toggle ---
document.getElementById('btn-ar').addEventListener('click', () => {
  arMode = !arMode;
  document.getElementById('btn-ar').classList.toggle('active', arMode);
  document.querySelectorAll('.ctrl').forEach(el => {
    el.classList.toggle('disabled', arMode);
  });
  if (!arMode) {
    // Restore all layer playback rates to 1.0 when leaving AR mode
    layers.forEach(v => { if (v) v.playbackRate = 1.0; });
  }
});
```

- [ ] **Step 5: Verify in browser**

Open the file. Expected:
- AR button appears in footer between RECORD/EXPORT and DEBUG
- Clicking AR highlights the button (active state)
- All four sliders dim to 35% opacity and stop responding to drag
- Clicking AR again restores sliders to full opacity and interactive

- [ ] **Step 6: Commit**

```bash
git -C "C:\Users\haden\Documents\Ship\src\audioProjects\palimpsest" add palimpsest.html
git -C "C:\Users\haden\Documents\Ship\src\audioProjects\palimpsest" commit -m "feat: AR mode UI — button, disabled slider state, playback rate reset"
```

---

### Task 4: AR mode — per-frame logic

**Files:**
- Modify: `palimpsest.html` — `renderFrame()` function in ENGINE section

- [ ] **Step 1: Add AR parameter-driving block inside `renderFrame()`**

Find this block in `renderFrame()`:
```js
  // Pull frequency data
  if (analyser) analyser.getByteFrequencyData(freqData);

  ctx.clearRect(0, 0, canvas.width, canvas.height);
```

Replace with:
```js
  // Pull frequency data
  if (analyser) analyser.getByteFrequencyData(freqData);

  // AR mode — audio drives parameters each frame
  if (arMode && analyser) {
    temporalOffset = 0.5 + getBandEnergy(0, 10) * 4.5;
    presentFloor   = getBandEnergy(11, 40);
    const futRate  = 0.5 + getBandEnergy(101, 160) * 1.5;
    const pastRate = 0.5 + getBandEnergy(161, 220) * 1.5;
    if (layers[3]) layers[3].playbackRate = futRate;
    if (layers[4]) layers[4].playbackRate = futRate;
    if (layers[0]) layers[0].playbackRate = pastRate;
    if (layers[1]) layers[1].playbackRate = pastRate;
    // Update slider thumbs and value displays to show audio-driven values
    const offsetSlider = document.getElementById('ctrl-offset');
    if (offsetSlider) offsetSlider.value = temporalOffset * 10;
    const floorSlider = document.getElementById('ctrl-floor');
    if (floorSlider) floorSlider.value = presentFloor * 100;
    document.getElementById('val-offset').textContent = `${temporalOffset.toFixed(1)}s`;
    document.getElementById('val-floor').textContent  = presentFloor.toFixed(2);
  }

  ctx.clearRect(0, 0, canvas.width, canvas.height);
```

- [ ] **Step 2: Verify in browser**

Load a video with varied audio. Enable AR mode. Play. Expected:
- Temporal offset visibly pumps in/out with bass hits — more bass = more ghosting between layers
- Present floor rises and falls with low-mid content — loud low-mids = NOW layer more opaque
- Past layers (PAST-1, PAST-2) speed up/slow down with treble content
- Future layers (FUT-1, FUT-2) speed up/slow down with high-mid content
- Temporal Offset and Present Floor sliders visually track the audio-driven values (thumbs move)
- Toggling AR off: layer playback rates snap back to 1.0×, sliders become interactive at their current tracked position
- Existing opacity behavior (all 5 bars in visualizer) still functions normally in AR mode
- Debug panel shows live values that reflect AR-driven state

- [ ] **Step 3: Commit**

```bash
git -C "C:\Users\haden\Documents\Ship\src\audioProjects\palimpsest" add palimpsest.html
git -C "C:\Users\haden\Documents\Ship\src\audioProjects\palimpsest" commit -m "feat: AR mode per-frame logic — audio drives offset, floor, and layer playback rates"
```
