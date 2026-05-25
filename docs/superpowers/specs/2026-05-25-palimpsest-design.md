# PALIMPSEST.EXE — Design Spec

## Concept

One video plays through 5 simultaneous `HTMLVideoElement` instances, each offset in time from the present layer. The opacity of each layer is driven live by a different frequency band of the present layer's audio — bass controls the deepest past, treble controls the furthest future. The result is a video that exists in multiple temporal states at once, past and future bleeding through the present like a palimpsest.

---

## Deliverable

Single self-contained HTML file. No build step, no dependencies, no external assets. Drop in browser, load video, play.

---

## Architecture

Three logical sections inside one `<script>` block:

- **State** — all mutable vars: layers array, offsets, analyser, presentFloor, frameCount, isRecording, etc.
- **Engine** — audio setup, canvas render loop, sync logic, record logic
- **UI** — DOM wiring, event listeners, control updates

No classes. Flat functions with clear naming.

---

## File Loading

Drag-and-drop onto canvas or LOAD FILE button → `URL.createObjectURL` → assigned to all 5 video `src` attributes. EXPORT and RECORD controls hidden until a file is loaded.

---

## Audio Graph

Created on first play (user-gesture gate). `AudioContext` → `MediaElementSourceNode` from layer 2 only → `AnalyserNode` (FFT size 512) → `destination`. All other layers muted via `video.muted = true`. The same `AnalyserNode` is also connected to a `MediaStreamDestinationNode` for recording.

---

## Master Clock

`layers[2].currentTime` is ground truth. Each frame:

1. `masterTime = layers[2].currentTime`
2. Compute target seek per layer: `clamp(masterTime + offset[i], 0, duration)`
3. Every 30 frames, write computed positions to `layers[i].currentTime` for the 4 non-present layers (drift correction)

---

## Render Loop (`requestAnimationFrame`)

Each frame:

1. Read `masterTime` from layer 2
2. `analyser.getByteFrequencyData(frequencyData)`
3. Compute normalized band energy for each layer (see Band Assignments)
4. `ctx.clearRect`
5. For each layer (past-2 → future-2):
   - `opacity = energy[i]`
   - layer 2: `opacity = Math.max(energy[2], presentFloor)`
   - `ctx.globalAlpha = opacity`
   - `ctx.drawImage(layers[i], 0, 0, w, h)`
6. `ctx.globalAlpha = 1.0`
7. Every 30 frames: re-sync non-present layer seek positions
8. Update visualizer bar heights and debug values
9. Stop loop when `masterTime >= duration`

---

## Band Assignments

| Layer | Label  | Freq Band    | FFT bins  |
|-------|--------|--------------|-----------|
| 0     | PAST-2 | Bass         | 0–10      |
| 1     | PAST-1 | Low-mid      | 11–40     |
| 2     | NOW    | Mid          | 41–100    |
| 3     | FUT-1  | High-mid     | 101–160   |
| 4     | FUT-2  | Treble       | 161–220   |

Energy normalized: `sum(bins) / (count * 255)` → 0.0–1.0

---

## Controls

| Parameter | Range | Default | Behavior |
|-----------|-------|---------|----------|
| Temporal Offset (T) | 0.1–5.0s | 1.0s | Updates offsets array immediately; triggers re-sync next frame |
| Present Floor | 0.0–1.0 | 0.30 | Updates `presentFloor`; takes effect next draw |
| FFT Smoothing | 0.0–0.95 | 0.85 | Writes directly to `analyser.smoothingTimeConstant` |
| Layer Count | 3 or 5 | 5 | 5: all layers; 3: indices [0,2,4] remapped to `[-T, 0, +T]` |

All controls update live during playback.

---

## Recording / Export

`MediaRecorder` captures `canvas.captureStream(30)` mixed with `MediaStreamDestinationNode` audio. RECORD button appears during playback — red border when active. On stop: `Blob` collected from recorded chunks → auto-download as `.webm`.

---

## UI Layout

Full-viewport, no max-width. Grid: `header (32px) / canvas (1fr) / visualizer (60px) / controls (44px) / footer (32px)`.

- **Header**: title left, status line right (`T = Xs | LAYERS: N | STATE`)
- **Canvas**: video composite output; layer stack tags on left edge; timecode bottom-left; transport (PLAY/PAUSE) bottom-right
- **Visualizer**: 5 equal columns, bar height = current opacity. NOW bar is brighter; white hairline marks the opacity floor
- **Controls**: 4 inline sliders with live value readout
- **Footer**: LOAD FILE, EXPORT, DEBUG toggle; RECORD appears during playback
- **Debug panel**: overlays above footer when active — 5 columns showing `t:` (seek position) and `α:` (opacity) per layer

Aesthetic: black and white terminal emulator. Courier New, monospace. No color except functional red on RECORD active state.

---

## Edge Cases

| Case | Handling |
|------|----------|
| Future layer seeks past `duration` | Clamp to `duration`; layer freezes on last frame |
| Past layer seeks before `0` | Clamp to `0`; layer freezes on first frame |
| Layer drift | 30-frame re-sync writes computed positions back |
| Silence (no audio) | Band energies → 0; non-present opacities collapse to ~0; NOW stays at floor |
| Layer count = 3 | Use layers [0, 2, 4], remap offsets to `[-T, 0, +T]` |
| Tab backgrounded | `masterTime` stays anchored to layer 2 `currentTime`; no drift |

---

## Done Criteria

- [ ] Single video file loads via drag-and-drop or file picker
- [ ] All 5 layers visible during playback (even at low energy)
- [ ] Layer opacities visibly respond to frequency bands
- [ ] Present layer (NOW) never disappears — opacity floor respected
- [ ] Temporal offset control produces visibly more/less ghosting
- [ ] Layer drift does not accumulate over 2-minute playback
- [ ] Only the present layer produces audio
- [ ] 5-bar visualizer labeled correctly and responsive; floor hairline visible on NOW
- [ ] Layer count toggle switches between 3 and 5 cleanly
- [ ] RECORD captures composited canvas + audio; downloads as `.webm`
- [ ] Debug panel shows live seek position and opacity per layer
- [ ] All controls update live during playback
