# PALIMPSEST.EXE Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a single self-contained HTML file that plays one video through 5 time-offset layers simultaneously, with each layer's opacity driven live by a different frequency band of the audio.

**Architecture:** Single `palimpsest.html` with three logical JS sections — State (all mutable vars), Engine (audio + canvas loop), UI (DOM wiring). `layers[2].currentTime` is ground truth for the master clock. No build step, no dependencies.

**Tech Stack:** HTML5 Canvas 2D, Web Audio API (`AudioContext`, `AnalyserNode`, `MediaElementSourceNode`, `MediaStreamDestinationNode`), `HTMLVideoElement` (×5), `MediaRecorder`, `requestAnimationFrame`

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `palimpsest.html` | Create | Entire application — HTML, CSS, JS |

---

### Task 1: HTML skeleton and full-viewport CSS layout

**Files:**
- Create: `palimpsest.html`

- [ ] **Step 1: Create the file with full-viewport grid layout**

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>PALIMPSEST.EXE</title>
<style>
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

html, body {
  width: 100%; height: 100%;
  background: #080808;
  color: #d0d0d0;
  font-family: 'Courier New', Courier, monospace;
  overflow: hidden;
}

.app {
  width: 100vw; height: 100vh;
  display: grid;
  grid-template-rows: 32px 1fr 60px 44px 32px;
}

/* Header */
.header {
  display: flex; align-items: center; justify-content: space-between;
  padding: 0 16px;
  border-bottom: 1px solid #1c1c1c;
}
.title { font-size: 11px; letter-spacing: 0.22em; color: #fff; text-transform: uppercase; }
.status { font-size: 10px; letter-spacing: 0.12em; color: #444; }
.status span { color: #888; }

/* Canvas */
.canvas-wrap {
  position: relative;
  background: #0e0e0e;
  border-bottom: 1px solid #161616;
  overflow: hidden;
}
canvas { display: block; width: 100%; height: 100%; }
.canvas-hint {
  position: absolute; inset: 0;
  display: flex; align-items: center; justify-content: center;
  font-size: 10px; letter-spacing: 0.18em; text-transform: uppercase; color: #1e1e1e;
  pointer-events: none;
}
.layer-stack {
  position: absolute; left: 12px; top: 50%; transform: translateY(-50%);
  display: flex; flex-direction: column; gap: 4px;
}
.ltag {
  font-size: 8px; letter-spacing: 0.14em; text-transform: uppercase;
  padding: 2px 6px; border: 1px solid;
}
.ltag.p2 { color: #2a2a2a; border-color: #222; }
.ltag.p1 { color: #444;    border-color: #333; }
.ltag.now{ color: #bbb;    border-color: #777; }
.ltag.f1 { color: #444;    border-color: #333; }
.ltag.f2 { color: #2a2a2a; border-color: #222; }
.timecode {
  position: absolute; bottom: 10px; left: 12px;
  font-size: 10px; letter-spacing: 0.12em; color: #333;
}
.transport {
  position: absolute; bottom: 8px; right: 12px;
  display: flex; gap: 6px;
}

/* Visualizer */
.viz {
  display: flex;
  border-bottom: 1px solid #161616;
  background: #0a0a0a;
}
.viz-col {
  flex: 1; display: flex; flex-direction: column;
  align-items: center; justify-content: flex-end;
  padding: 4px 0 6px; gap: 4px;
  border-right: 1px solid #111;
  position: relative;
}
.viz-col:last-child { border-right: none; }
.bar-track { flex: 1; width: 50%; display: flex; align-items: flex-end; }
.bar { width: 100%; background: #2a2a2a; transition: height 0.05s; }
.bar.now { background: #777; }
.now-floor {
  position: absolute; left: 25%; right: 25%; height: 1px;
  background: rgba(255,255,255,0.35);
}
.vlabel { font-size: 8px; letter-spacing: 0.12em; text-transform: uppercase; color: #333; }
.vlabel.now { color: #777; }

/* Controls */
.controls {
  display: flex; align-items: center;
  border-bottom: 1px solid #161616;
  background: #090909;
}
.ctrl {
  flex: 1; display: flex; align-items: center; gap: 10px;
  padding: 0 14px; height: 100%;
  border-right: 1px solid #161616;
}
.ctrl:last-child { border-right: none; }
.ctrl-label { font-size: 8px; letter-spacing: 0.14em; text-transform: uppercase; color: #3a3a3a; white-space: nowrap; }
.ctrl-value { font-size: 14px; color: #bbb; min-width: 38px; text-align: right; }
.ctrl-range {
  flex: 1; height: 2px; -webkit-appearance: none;
  background: #222; outline: none; cursor: pointer;
}
.ctrl-range::-webkit-slider-thumb {
  -webkit-appearance: none; width: 7px; height: 7px;
  background: #777; cursor: pointer; border-radius: 0;
}

/* Footer */
.footer {
  display: flex; align-items: center; gap: 6px;
  padding: 0 12px; background: #070707;
}
.fsep { flex: 1; }

/* Buttons */
.btn {
  background: none; border: 1px solid #1e1e1e; color: #444;
  font-family: inherit; font-size: 9px; letter-spacing: 0.14em;
  padding: 3px 10px; cursor: pointer; text-transform: uppercase;
}
.btn:hover { border-color: #444; color: #aaa; }
.btn.active { border-color: #666; color: #ccc; background: #141414; }
.btn.record-active { border-color: #888; color: #fff; }
.btn.hidden { display: none; }

/* Debug panel */
.debug {
  position: fixed; bottom: 32px; left: 0; right: 0;
  background: rgba(8,8,8,0.94); border-top: 1px solid #1c1c1c;
  display: none; flex-direction: row;
  padding: 6px 16px; gap: 0;
}
.debug.visible { display: flex; }
.debug-col {
  flex: 1; display: flex; flex-direction: column; gap: 2px;
  border-right: 1px solid #151515; padding: 0 12px;
}
.debug-col:first-child { padding-left: 0; }
.debug-col:last-child { border-right: none; }
.dlabel { font-size: 8px; letter-spacing: 0.14em; text-transform: uppercase; color: #2a2a2a; }
.dlabel.now { color: #666; }
.dval { font-size: 9px; letter-spacing: 0.08em; color: #444; }
.dval.now { color: #777; }
</style>
</head>
<body>
<div class="app">

  <div class="header">
    <div class="title">PALIMPSEST.EXE</div>
    <div class="status" id="status">
      T = <span id="s-offset">1.0s</span> &nbsp;|&nbsp;
      LAYERS: <span id="s-layers">5</span> &nbsp;|&nbsp;
      <span id="s-state">READY</span>
    </div>
  </div>

  <div class="canvas-wrap" id="canvas-wrap">
    <canvas id="canvas"></canvas>
    <div class="canvas-hint" id="canvas-hint">DROP VIDEO FILE TO BEGIN</div>
    <div class="layer-stack">
      <div class="ltag f2">FUT-2</div>
      <div class="ltag f1">FUT-1</div>
      <div class="ltag now">NOW</div>
      <div class="ltag p1">PAST-1</div>
      <div class="ltag p2">PAST-2</div>
    </div>
    <div class="timecode" id="timecode">--:--.-</div>
    <div class="transport">
      <button class="btn" id="btn-play">&#9654; PLAY</button>
      <button class="btn" id="btn-pause">&#9646;&#9646; PAUSE</button>
    </div>
  </div>

  <div class="viz" id="viz">
    <div class="viz-col">
      <div class="bar-track"><div class="bar" id="bar-0" style="height:0%"></div></div>
      <div class="vlabel">PAST-2</div>
    </div>
    <div class="viz-col">
      <div class="bar-track"><div class="bar" id="bar-1" style="height:0%"></div></div>
      <div class="vlabel">PAST-1</div>
    </div>
    <div class="viz-col">
      <div class="bar-track"><div class="bar now" id="bar-2" style="height:30%"></div></div>
      <div class="now-floor" id="now-floor" style="bottom: calc(6px + 12px + 30%)"></div>
      <div class="vlabel now">NOW</div>
    </div>
    <div class="viz-col">
      <div class="bar-track"><div class="bar" id="bar-3" style="height:0%"></div></div>
      <div class="vlabel">FUT-1</div>
    </div>
    <div class="viz-col">
      <div class="bar-track"><div class="bar" id="bar-4" style="height:0%"></div></div>
      <div class="vlabel">FUT-2</div>
    </div>
  </div>

  <div class="controls">
    <div class="ctrl">
      <div class="ctrl-label">Temporal Offset</div>
      <div class="ctrl-value" id="val-offset">1.0s</div>
      <input type="range" class="ctrl-range" id="ctrl-offset" min="1" max="50" value="10">
    </div>
    <div class="ctrl">
      <div class="ctrl-label">Present Floor</div>
      <div class="ctrl-value" id="val-floor">0.30</div>
      <input type="range" class="ctrl-range" id="ctrl-floor" min="0" max="100" value="30">
    </div>
    <div class="ctrl">
      <div class="ctrl-label">FFT Smoothing</div>
      <div class="ctrl-value" id="val-smooth">0.85</div>
      <input type="range" class="ctrl-range" id="ctrl-smooth" min="0" max="95" value="85">
    </div>
    <div class="ctrl">
      <div class="ctrl-label">Layers</div>
      <div class="ctrl-value" id="val-layers">5</div>
      <input type="range" class="ctrl-range" id="ctrl-layers" min="0" max="1" step="1" value="1">
    </div>
  </div>

  <div class="footer">
    <button class="btn active" id="btn-load">LOAD FILE</button>
    <div class="fsep"></div>
    <button class="btn hidden" id="btn-record">&#9679; RECORD</button>
    <button class="btn hidden" id="btn-export">EXPORT</button>
    <button class="btn" id="btn-debug">DEBUG</button>
  </div>

</div>

<div class="debug" id="debug-panel">
  <div class="debug-col">
    <div class="dlabel">PAST-2</div>
    <div class="dval" id="dbg-t-0">t: —</div>
    <div class="dval" id="dbg-a-0">α: 0.00</div>
  </div>
  <div class="debug-col">
    <div class="dlabel">PAST-1</div>
    <div class="dval" id="dbg-t-1">t: —</div>
    <div class="dval" id="dbg-a-1">α: 0.00</div>
  </div>
  <div class="debug-col">
    <div class="dlabel now">NOW</div>
    <div class="dval now" id="dbg-t-2">t: —</div>
    <div class="dval now" id="dbg-a-2">α: 0.30</div>
  </div>
  <div class="debug-col">
    <div class="dlabel">FUT-1</div>
    <div class="dval" id="dbg-t-3">t: —</div>
    <div class="dval" id="dbg-a-3">α: 0.00</div>
  </div>
  <div class="debug-col">
    <div class="dlabel">FUT-2</div>
    <div class="dval" id="dbg-t-4">t: —</div>
    <div class="dval" id="dbg-a-4">α: 0.00</div>
  </div>
</div>

<script>
// ============================================================
// STATE
// ============================================================

// ============================================================
// ENGINE
// ============================================================

// ============================================================
// UI
// ============================================================
</script>
</body>
</html>
```

- [ ] **Step 2: Verify layout in browser**

Open `palimpsest.html` in a browser. Expected:
- Page fills entire viewport with no scrollbars
- Header with "PALIMPSEST.EXE" top-left, status top-right
- Large dark canvas area with "DROP VIDEO FILE TO BEGIN" centered
- 5-column visualizer strip below canvas
- 4-control strip below that
- Footer with LOAD FILE and DEBUG buttons
- No console errors

- [ ] **Step 3: Commit**

```bash
git add palimpsest.html
git commit -m "feat: palimpsest HTML skeleton and CSS layout"
```

---

### Task 2: State initialization and video element creation

**Files:**
- Modify: `palimpsest.html` — STATE section of `<script>`

- [ ] **Step 1: Add state variables and constants**

Replace `// ============================================================\n// STATE` section with:

```js
// ============================================================
// STATE
// ============================================================

// Params (live-updatable)
let temporalOffset = 1.0;   // seconds between adjacent layers
let presentFloor   = 0.30;  // minimum opacity for layer 2
let layerCount     = 5;     // 3 or 5

// Playback
let duration    = 0;
let frameCount  = 0;
let animFrameId = null;
let isPlaying   = false;

// Recording
let isRecording  = false;
let mediaRecorder = null;
let recordChunks  = [];

// Audio
let audioCtx   = null;
let analyser   = null;
let freqData   = null;
let recordDest = null;

// Layers — 5 HTMLVideoElement instances, created in setupLayers()
let layers = [];

// Frequency band definitions: [startBin, endBin] per layer index
const BANDS = [
  [0,   10],   // 0: PAST-2  bass
  [11,  40],   // 1: PAST-1  low-mid
  [41,  100],  // 2: NOW     mid
  [101, 160],  // 3: FUT-1   high-mid
  [161, 220],  // 4: FUT-2   treble
];

// DOM refs
const canvas     = document.getElementById('canvas');
const ctx        = canvas.getContext('2d');
const canvasWrap = document.getElementById('canvas-wrap');
```

- [ ] **Step 2: Add setupLayers() to ENGINE section**

```js
// ============================================================
// ENGINE
// ============================================================

function setupLayers(src) {
  // Tear down any previous layers
  layers.forEach(v => { v.pause(); v.src = ''; });

  layers = Array.from({ length: 5 }, (_, i) => {
    const v = document.createElement('video');
    v.src     = src;
    v.muted   = true;
    v.preload = 'auto';
    return v;
  });

  // Only the present layer (index 2) produces audio
  layers[2].muted = false;

  // Once metadata loads on layer 2, store duration
  layers[2].addEventListener('loadedmetadata', () => {
    duration = layers[2].duration;
    // Size canvas to video's natural dimensions
    canvas.width  = layers[2].videoWidth  || canvasWrap.clientWidth;
    canvas.height = layers[2].videoHeight || canvasWrap.clientHeight;
  }, { once: true });
}
```

- [ ] **Step 3: Verify in browser console**

Open browser DevTools console and paste:
```js
setupLayers('');
console.log(layers.length, layers[2].muted);
```
Expected output: `5 false`

- [ ] **Step 4: Commit**

```bash
git add palimpsest.html
git commit -m "feat: state initialization and video layer setup"
```

---

### Task 3: File loading — drag-and-drop and file picker

**Files:**
- Modify: `palimpsest.html` — UI section and ENGINE section

- [ ] **Step 1: Add loadFile() to ENGINE section**

```js
function loadFile(file) {
  if (!file || !file.type.startsWith('video/')) return;

  const src = URL.createObjectURL(file);
  setupLayers(src);

  document.getElementById('canvas-hint').style.display = 'none';
  document.getElementById('btn-export').classList.remove('hidden');
  setStatus('LOADED');
}
```

- [ ] **Step 2: Add file input and drag-and-drop wiring to UI section**

```js
// ============================================================
// UI
// ============================================================

// --- File picker ---
const fileInput = document.createElement('input');
fileInput.type   = 'file';
fileInput.accept = 'video/*';
fileInput.style.display = 'none';
document.body.appendChild(fileInput);

fileInput.addEventListener('change', () => {
  if (fileInput.files[0]) loadFile(fileInput.files[0]);
});

document.getElementById('btn-load').addEventListener('click', () => fileInput.click());

// --- Drag and drop ---
canvasWrap.addEventListener('dragover', e => {
  e.preventDefault();
  canvasWrap.style.outline = '1px solid #555';
});

canvasWrap.addEventListener('dragleave', () => {
  canvasWrap.style.outline = '';
});

canvasWrap.addEventListener('drop', e => {
  e.preventDefault();
  canvasWrap.style.outline = '';
  const file = e.dataTransfer.files[0];
  if (file) loadFile(file);
});
```

- [ ] **Step 3: Add setStatus() helper to ENGINE section**

```js
function setStatus(text) {
  document.getElementById('s-state').textContent = text;
}
```

- [ ] **Step 4: Verify in browser**

Open the file, drag a video onto the canvas. Expected:
- "DROP VIDEO FILE TO BEGIN" hint disappears
- EXPORT button appears in footer
- Status line shows "LOADED"
- No console errors

- [ ] **Step 5: Commit**

```bash
git add palimpsest.html
git commit -m "feat: file loading via drag-and-drop and file picker"
```

---

### Task 4: Audio graph setup

**Files:**
- Modify: `palimpsest.html` — ENGINE section

- [ ] **Step 1: Add setupAudio() to ENGINE section**

Called once on first play (requires user gesture before `AudioContext` can be created).

```js
function setupAudio() {
  if (audioCtx) return; // already set up

  audioCtx = new AudioContext();

  analyser = audioCtx.createAnalyser();
  analyser.fftSize = 512;
  analyser.smoothingTimeConstant = 0.85;
  freqData = new Uint8Array(analyser.frequencyBinCount); // 256 bins

  const src = audioCtx.createMediaElementSource(layers[2]);
  src.connect(analyser);
  analyser.connect(audioCtx.destination);

  // Secondary branch for recording
  recordDest = audioCtx.createMediaStreamDestination();
  analyser.connect(recordDest);
}
```

- [ ] **Step 2: Add getBandEnergy() to ENGINE section**

```js
function getBandEnergy(startBin, endBin) {
  let sum = 0;
  for (let b = startBin; b <= endBin; b++) sum += freqData[b];
  return sum / ((endBin - startBin + 1) * 255); // normalized 0..1
}
```

- [ ] **Step 3: Verify in browser console**

Load a video, then in the console:
```js
setupAudio();
console.log(analyser.fftSize, freqData.length);
```
Expected: `512 256`

- [ ] **Step 4: Commit**

```bash
git add palimpsest.html
git commit -m "feat: Web Audio graph setup with AnalyserNode"
```

---

### Task 5: Render loop — canvas compositing with opacity

**Files:**
- Modify: `palimpsest.html` — ENGINE section

- [ ] **Step 1: Add getActiveLayers() to ENGINE section**

Returns the layer indices and their time offsets for the current `layerCount`.

```js
function getActiveLayers() {
  if (layerCount === 5) {
    return [0, 1, 2, 3, 4].map(i => ({ i, offset: (i - 2) * temporalOffset }));
  }
  // layerCount === 3: use layers 0, 2, 4 mapped to offsets -T, 0, +T
  return [
    { i: 0, offset: -temporalOffset },
    { i: 2, offset: 0 },
    { i: 4, offset:  temporalOffset },
  ];
}
```

- [ ] **Step 2: Add syncLayers() to ENGINE section**

Writes computed seek positions to all non-present layers to correct drift.

```js
function syncLayers() {
  const masterTime = layers[2].currentTime;
  const active = getActiveLayers();
  active.forEach(({ i, offset }) => {
    if (i === 2) return; // present layer drives itself
    const target = Math.max(0, Math.min(masterTime + offset, duration));
    // Only write if meaningfully off — avoids thrashing
    if (Math.abs(layers[i].currentTime - target) > 0.1) {
      layers[i].currentTime = target;
    }
  });
}
```

- [ ] **Step 3: Add renderFrame() to ENGINE section**

```js
function renderFrame() {
  if (!isPlaying) return;

  frameCount++;
  const masterTime = layers[2].currentTime;

  // Resize canvas to container if needed
  const w = canvasWrap.clientWidth;
  const h = canvasWrap.clientHeight;
  if (canvas.width !== w || canvas.height !== h) {
    canvas.width  = w;
    canvas.height = h;
  }

  // Pull frequency data
  if (analyser) analyser.getByteFrequencyData(freqData);

  ctx.clearRect(0, 0, canvas.width, canvas.height);

  const active = getActiveLayers();
  const opacities = [0, 0, 0, 0, 0];

  active.forEach(({ i, offset }) => {
    const [s, e] = BANDS[i];
    let opacity = analyser ? getBandEnergy(s, e) : 0;
    if (i === 2) opacity = Math.max(opacity, presentFloor);
    opacities[i] = opacity;

    ctx.globalAlpha = opacity;
    ctx.drawImage(layers[i], 0, 0, canvas.width, canvas.height);
  });

  ctx.globalAlpha = 1.0;

  // Drift correction every 30 frames
  if (frameCount % 30 === 0) syncLayers();

  // Update UI
  updateVisualizer(opacities, masterTime);
  updateTimecode(masterTime);

  // End condition
  if (masterTime >= duration && duration > 0) {
    pausePlayback();
    setStatus('DONE');
    return;
  }

  animFrameId = requestAnimationFrame(renderFrame);
}
```

- [ ] **Step 4: Verify in browser**

Load a video. In console:
```js
setupAudio();
layers.forEach(v => v.play());
isPlaying = true;
renderFrame();
```
Expected: video frames composite on the canvas, present layer visible.

- [ ] **Step 5: Commit**

```bash
git add palimpsest.html
git commit -m "feat: rAF render loop with per-layer opacity from frequency bands"
```

---

### Task 6: Visualizer and debug panel updates

**Files:**
- Modify: `palimpsest.html` — ENGINE section

- [ ] **Step 1: Add updateVisualizer() to ENGINE section**

```js
function updateVisualizer(opacities, masterTime) {
  for (let i = 0; i < 5; i++) {
    const bar = document.getElementById(`bar-${i}`);
    if (bar) bar.style.height = `${(opacities[i] * 100).toFixed(1)}%`;

    const dt = document.getElementById(`dbg-t-${i}`);
    const da = document.getElementById(`dbg-a-${i}`);
    if (dt) {
      const t = (i === 2) ? masterTime : Math.max(0, Math.min(masterTime + (i - 2) * temporalOffset, duration));
      dt.textContent = `t: ${t.toFixed(2)}s`;
    }
    if (da) da.textContent = `α: ${opacities[i].toFixed(2)}`;
  }

  // Keep the NOW floor hairline aligned with presentFloor
  const floor = document.getElementById('now-floor');
  if (floor) {
    // labelHeight ≈ 12px + 4px gap, track is (60px - 22px) = 38px
    floor.style.bottom = `calc(22px + ${presentFloor * 100}%)`;
  }
}
```

- [ ] **Step 2: Add updateTimecode() to ENGINE section**

```js
function updateTimecode(t) {
  const fmt = s => {
    const m = Math.floor(s / 60);
    const sec = (s % 60).toFixed(1).padStart(4, '0');
    return `${String(m).padStart(2, '0')}:${sec}`;
  };
  document.getElementById('timecode').textContent =
    `${fmt(t)} / ${fmt(duration)}`;
  document.getElementById('s-offset').textContent = `${temporalOffset.toFixed(1)}s`;
  document.getElementById('s-layers').textContent = `${layerCount}`;
}
```

- [ ] **Step 3: Verify in browser**

Load and play a video. Expected:
- All 5 visualizer bars animate during playback
- NOW bar is brighter than others
- Floor hairline visible on NOW bar
- Debug panel (click DEBUG) shows live `t:` and `α:` values per layer
- Timecode advances

- [ ] **Step 4: Commit**

```bash
git add palimpsest.html
git commit -m "feat: visualizer bars and debug panel live updates"
```

---

### Task 7: Transport — play, pause, end condition

**Files:**
- Modify: `palimpsest.html` — ENGINE and UI sections

- [ ] **Step 1: Add startPlayback() and pausePlayback() to ENGINE section**

```js
function startPlayback() {
  if (!layers.length || isPlaying) return;

  setupAudio();
  if (audioCtx.state === 'suspended') audioCtx.resume();

  layers.forEach(v => v.play().catch(() => {}));
  isPlaying = true;
  setStatus('PLAYING');
  document.getElementById('btn-record').classList.remove('hidden');
  animFrameId = requestAnimationFrame(renderFrame);
}

function pausePlayback() {
  isPlaying = false;
  if (animFrameId) { cancelAnimationFrame(animFrameId); animFrameId = null; }
  layers.forEach(v => v.pause());
  setStatus('PAUSED');
  document.getElementById('btn-record').classList.add('hidden');
  if (isRecording) stopRecording();
}
```

- [ ] **Step 2: Wire transport buttons in UI section**

```js
document.getElementById('btn-play').addEventListener('click', startPlayback);
document.getElementById('btn-pause').addEventListener('click', pausePlayback);
```

- [ ] **Step 3: Wire debug toggle in UI section**

```js
document.getElementById('btn-debug').addEventListener('click', () => {
  const panel = document.getElementById('debug-panel');
  panel.classList.toggle('visible');
  document.getElementById('btn-debug').classList.toggle('active');
});
```

- [ ] **Step 4: Verify in browser**

Load a video. Click PLAY — video composites on canvas, visualizer animates. Click PAUSE — everything stops. RECORD button appears during playback, disappears on pause. Playback ends cleanly when `masterTime >= duration`.

- [ ] **Step 5: Commit**

```bash
git add palimpsest.html
git commit -m "feat: play/pause transport and debug panel toggle"
```

---

### Task 8: Controls — live-updating sliders

**Files:**
- Modify: `palimpsest.html` — UI section

- [ ] **Step 1: Wire all four control sliders in UI section**

```js
// Temporal Offset: slider range 1–50, value maps to 0.1–5.0s (divide by 10)
document.getElementById('ctrl-offset').addEventListener('input', function() {
  temporalOffset = this.value / 10;
  document.getElementById('val-offset').textContent = `${temporalOffset.toFixed(1)}s`;
  document.getElementById('s-offset').textContent   = `${temporalOffset.toFixed(1)}s`;
  // Trigger immediate re-sync on next tick
  if (isPlaying) syncLayers();
});

// Present Floor: slider range 0–100, value / 100
document.getElementById('ctrl-floor').addEventListener('input', function() {
  presentFloor = this.value / 100;
  document.getElementById('val-floor').textContent = presentFloor.toFixed(2);
});

// FFT Smoothing: slider range 0–95, value / 100
document.getElementById('ctrl-smooth').addEventListener('input', function() {
  const v = this.value / 100;
  document.getElementById('val-smooth').textContent = v.toFixed(2);
  if (analyser) analyser.smoothingTimeConstant = v;
});

// Layer Count: slider 0=3layers, 1=5layers
document.getElementById('ctrl-layers').addEventListener('input', function() {
  layerCount = this.value === '1' ? 5 : 3;
  document.getElementById('val-layers').textContent = `${layerCount}`;
  document.getElementById('s-layers').textContent   = `${layerCount}`;
});
```

- [ ] **Step 2: Verify in browser**

Load and play a video. While playing:
- Drag Temporal Offset slider — ghosting between layers visibly increases/decreases
- Drag Present Floor slider — NOW bar minimum height changes
- Drag FFT Smoothing slider — bar animations become more/less reactive
- Toggle Layer Count — number of layers composited changes

- [ ] **Step 3: Commit**

```bash
git add palimpsest.html
git commit -m "feat: live-updating controls for all four parameters"
```

---

### Task 9: Recording — MediaRecorder capture and download

**Files:**
- Modify: `palimpsest.html` — ENGINE and UI sections

- [ ] **Step 1: Add startRecording() and stopRecording() to ENGINE section**

```js
function startRecording() {
  if (isRecording || !isPlaying) return;

  // Mix canvas video stream with audio
  const videoStream = canvas.captureStream(30);
  const audioStream = recordDest.stream;
  const combined = new MediaStream([
    ...videoStream.getVideoTracks(),
    ...audioStream.getAudioTracks(),
  ]);

  recordChunks = [];
  mediaRecorder = new MediaRecorder(combined, { mimeType: 'video/webm' });

  mediaRecorder.ondataavailable = e => {
    if (e.data.size > 0) recordChunks.push(e.data);
  };

  mediaRecorder.onstop = () => {
    const blob = new Blob(recordChunks, { type: 'video/webm' });
    const url  = URL.createObjectURL(blob);
    const a    = document.createElement('a');
    a.href     = url;
    a.download = `palimpsest-${Date.now()}.webm`;
    a.click();
    URL.revokeObjectURL(url);
  };

  mediaRecorder.start();
  isRecording = true;

  const btn = document.getElementById('btn-record');
  btn.textContent = '&#9646;&#9646; STOP REC';
  btn.classList.add('record-active');
}

function stopRecording() {
  if (!isRecording) return;
  mediaRecorder.stop();
  isRecording = false;

  const btn = document.getElementById('btn-record');
  btn.innerHTML = '&#9679; RECORD';
  btn.classList.remove('record-active');
}
```

- [ ] **Step 2: Wire RECORD button in UI section**

```js
document.getElementById('btn-record').addEventListener('click', () => {
  if (isRecording) stopRecording();
  else startRecording();
});
```

- [ ] **Step 3: Verify in browser**

Load a video, click PLAY, click RECORD. Record ~10 seconds. Click STOP REC. Expected:
- Browser auto-downloads a `.webm` file
- Playback of downloaded file shows the composited canvas with audio
- RECORD button returns to normal state

- [ ] **Step 4: Commit**

```bash
git add palimpsest.html
git commit -m "feat: MediaRecorder capture of composited canvas with audio download"
```

---

### Task 10: Final integration — canvas resize, edge cases, done criteria check

**Files:**
- Modify: `palimpsest.html` — ENGINE section

- [ ] **Step 1: Add window resize handler to UI section**

Canvas must stay matched to its container when the window resizes.

```js
window.addEventListener('resize', () => {
  canvas.width  = canvasWrap.clientWidth;
  canvas.height = canvasWrap.clientHeight;
});
```

- [ ] **Step 2: Add initial canvas size to UI section**

Set canvas dimensions on load before any video is present.

```js
canvas.width  = canvasWrap.clientWidth;
canvas.height = canvasWrap.clientHeight;
```

- [ ] **Step 3: Verify all done criteria manually**

Work through each item in the spec's Done Criteria:

```
[ ] Load video via drag-and-drop — drop a file, hint disappears, EXPORT appears
[ ] Load video via LOAD FILE button — click, pick file, same result
[ ] All 5 layers visible during playback — check on a video with varied audio
[ ] Layer opacities respond to frequency bands — bass hit → PAST-2 bar surges
[ ] NOW layer never disappears — set floor to 0.3, confirm bar never drops below
[ ] Temporal offset control changes ghosting — drag slider while playing
[ ] Layer drift < 0.1s after 2 min playback — check debug t: values stay close to expected
[ ] Only NOW layer produces audio — confirm no echo or doubling in output
[ ] 5-bar visualizer labels correct — PAST-2/PAST-1/NOW/FUT-1/FUT-2
[ ] Floor hairline visible on NOW bar — check at various presentFloor values
[ ] Layer count toggle 3↔5 — toggle during playback, density changes
[ ] RECORD captures canvas+audio — record 10s, download, play in VLC/browser
[ ] Debug panel shows live t: and α: per layer
[ ] All controls update live during playback
```

- [ ] **Step 4: Final commit**

```bash
git add palimpsest.html
git commit -m "feat: final integration — resize handling and edge case verification"
```
