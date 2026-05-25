# PALIMPSEST.EXE v2 — Design Spec

## Scope

Three additions to the existing `palimpsest.html`:

1. **Text visibility** — lift all UI text to white unless dimming carries functional meaning
2. **Audio-reactive mode (AR)** — video audio takes full control of four parameters when toggled on
3. **Loop mode** — seamless video loop instead of stopping at end

No new files. All changes are in `palimpsest.html`.

---

## 1. Text Visibility

### Rule
Everything white unless the color encodes information. Functional dim = intentional hierarchy. Decorative dim = remove it.

### What keeps graduated color
- **Layer stack tags (ltag)** — past/future fade encodes temporal distance. Lift values but keep hierarchy:
  - `.ltag.p2`, `.ltag.f2` → `#555` / border `#444`
  - `.ltag.p1`, `.ltag.f1` → `#777` / border `#555`
  - `.ltag.now` → `#fff` / border `#aaa`
- **Canvas drop hint** (`.canvas-hint`) — ghost watermark, stays near-invisible (`#1e1e1e`)
- **Visualizer labels** — NOW → `#fff`; others → `#777`

### What goes white
| Element | Current | New |
|---------|---------|-----|
| `.status` (labels) | `#444` | `#fff` |
| `.status span` (values) | `#888` | `#fff` |
| `.timecode` | `#333` | `#aaa` |
| `.ctrl-label` | `#3a3a3a` | `#888` |
| `.ctrl-value` | `#bbb` | `#fff` |
| `.btn` default | `#444` | `#aaa` |
| `.btn.active` | `#ccc` | `#fff` |
| `.dlabel` | `#2a2a2a` | `#777` |
| `.dlabel.now` | `#666` | `#fff` |
| `.dval` | `#444` | `#888` |
| `.dval.now` | `#777` | `#fff` |
| `.bar` (viz bars) | `#2a2a2a` | `#444` |
| `.bar.now` | `#777` | `#aaa` |

### Font size bumps
All sizes step up one level. `ctrl-value` (14px) stays.

| Current | New |
|---------|-----|
| 8px | 10px |
| 9px | 11px |
| 10px | 12px |
| 11px | 13px |

---

## 2. Audio-Reactive Mode (AR)

### Toggle
Button `AR` added to footer. Toggles `arMode` boolean. When active: button gets `.active` class, sliders get `.disabled` class (opacity 0.3, pointer-events none). When inactive: sliders restore to normal, parameters resume from their last manual slider values.

### State
```js
let arMode = false;
```

### Band-to-Parameter Mapping

Fixed. Runs every frame inside `renderFrame()` when `arMode` is true. Uses the existing `freqData` array (already populated each frame by `analyser.getByteFrequencyData`).

| Band | Bins | Parameter | Formula |
|------|------|-----------|---------|
| Bass | 0–10 | `temporalOffset` | `0.5 + energy * 4.5` → 0.5–5.0s |
| Low-mid | 11–40 | `presentFloor` | `energy` → 0.0–1.0 |
| High-mid | 101–160 | `layers[3].playbackRate`, `layers[4].playbackRate` | `0.5 + energy * 1.5` → 0.5–2.0× |
| Treble | 161–220 | `layers[0].playbackRate`, `layers[1].playbackRate` | `0.5 + energy * 1.5` → 0.5–2.0× |

Mid (41–100) continues to drive opacity via the existing `getBandEnergy` path — unchanged.

### Per-frame logic (inside `renderFrame`, before compositing)

```
if (arMode && analyser) {
  temporalOffset = 0.5 + getBandEnergy(0, 10) * 4.5
  presentFloor   = getBandEnergy(11, 40)
  const futRate  = 0.5 + getBandEnergy(101, 160) * 1.5
  const pastRate = 0.5 + getBandEnergy(161, 220) * 1.5
  layers[3].playbackRate = futRate
  layers[4].playbackRate = futRate
  layers[0].playbackRate = pastRate
  layers[1].playbackRate = pastRate
}
```

Layer 2 (`playbackRate`) is left at 1.0 — master clock must run at normal speed.

### Slider state when AR is on
Slider thumbs and value displays both update each frame to reflect audio-driven values — `ctrl-offset.value`, `ctrl-floor.value`, `val-offset`, `val-floor`, `s-offset` all track the live audio output. This way the user can see what the audio is doing in real time. When AR is toggled off, the sliders resume from their current (audio-driven) positions, not their pre-AR positions.

### UI additions
- Footer: `<button class="btn" id="btn-ar">AR</button>` between RECORD and DEBUG
- CSS: `.ctrl.disabled { opacity: 0.35; pointer-events: none; }` — applied to all four `.ctrl` elements when AR is active

---

## 3. Loop Mode

### Toggle
Button `LOOP` added to footer between LOAD FILE and the spacer. Toggles `loopMode` boolean. Active state shown via `.active` class.

### State
```js
let loopMode = false;
```

### End-condition change in `renderFrame()`

Current:
```js
if (masterTime >= duration && duration > 0) {
  pausePlayback();
  setStatus('DONE');
  return;
}
```

New:
```js
if (masterTime >= duration && duration > 0) {
  if (loopMode) {
    const active = getActiveLayers();
    active.forEach(({ i, offset }) => {
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

Layer 2 always seeks to 0. Other layers seek to `clamp(offset, 0, duration)` — same clamp logic as `syncLayers()`, anchored at `masterTime = 0`.

### Behavior notes
- RECORD continues uninterrupted through a loop point
- If `loopMode` is toggled off mid-playback, next end-condition hit stops normally
- Status stays `PLAYING` through loop points

---

## Done Criteria

- [ ] All UI text is white or functionally graduated — no decorative dim
- [ ] Font sizes stepped up across the board
- [ ] AR button toggles mode; sliders dim and stop responding when active
- [ ] In AR mode, bass drives temporal offset, low-mid drives present floor, high-mid drives future rates, treble drives past rates
- [ ] Status bar and control value displays reflect audio-driven values in AR mode
- [ ] AR off restores slider control with last manual values
- [ ] LOOP button appears in footer, toggles loop behavior
- [ ] Video loops seamlessly — all layers seek to correct offset positions at loop point
- [ ] RECORD captures through loop points without interruption
- [ ] No regressions on existing file loading, transport, recording, debug panel
