# PALIMPSEST.EXE — Design Document

## Concept

The same video plays 5 times simultaneously, each instance offset in time from the others. The five layers are composited on a single canvas, with each layer's opacity driven live by a different frequency band of the video's own audio. When bass hits, the deepest-past layer surges forward. When treble peaks, the furthest-future layer bleeds through. The present layer always has a minimum opacity floor — it never fully disappears.

The result is a video that exists in multiple temporal states at once. Past and future are always visible beneath the surface, bleeding through like overexposed film. The experience is of a memory: the present moment contaminated by what came before and what's about to happen.

The name is deliberate — a palimpsest is a manuscript where older text is still visible under newer writing.

---

## Core Mechanic

Five `HTMLVideoElement` instances all share the same source. They are assigned seek positions at staggered offsets from a master clock: `[-2T, -T, 0, +T, +2T]` where `T` is user-configurable. Each frame, all five are drawn to a canvas in order (deepest past first), with `globalAlpha` set per layer based on the normalized energy in that layer's assigned frequency band. Every 30 frames, each layer's `currentTime` is re-synced to the master clock plus its offset to prevent drift.

Audio comes only from layer 2 (the "present" layer). All others are muted.

---

## Algorithm

```js
// --- SETUP ---
const NUM_LAYERS = 5;
const layers = Array.from({ length: NUM_LAYERS }, () => {
  const v = document.createElement('video');
  v.src = videoSrc;
  v.muted = true; // all muted
  return v;
});
layers[2].muted = false; // present layer plays audio

// Temporal offsets relative to master clock
// T = temporalOffset (user-controlled, default 1.0s)
// offsets: [-2T, -T, 0, +T, +2T]
const getOffsets = () => [-2, -1, 0, 1, 2].map(m => m * temporalOffset);

// Frequency band assignments per layer (index into getBandEnergy call)
// layer 0 (past-2)  → bass
// layer 1 (past-1)  → low-mid
// layer 2 (present) → mid
// layer 3 (future-1)→ high-mid
// layer 4 (future-2)→ treble
const BANDS = [
  [0, 10],    // bass
  [11, 40],   // low-mid
  [41, 100],  // mid
  [101, 160], // high-mid
  [161, 220]  // treble
];

const MIN_PRESENT_OPACITY = 0.3;

// --- PLAYBACK SYNC ---
// Called on play and every 30 frames to prevent drift
function syncLayers() {
  const offsets = getOffsets();
  for (let i = 0; i < NUM_LAYERS; i++) {
    const t = Math.max(0, Math.min(masterTime + offsets[i], duration));
    layers[i].currentTime = t;
  }
}

// --- PER FRAME ---
analyser.getByteFrequencyData(frequencyData);

ctx.clearRect(0, 0, w, h);

for (let i = 0; i < NUM_LAYERS; i++) {
  const [start, end] = BANDS[i];
  let energy = 0;
  for (let b = start; b <= end; b++) energy += frequencyData[b];
  energy = energy / ((end - start + 1) * 255); // normalize 0..1

  let opacity = energy;
  if (i === 2) opacity = Math.max(opacity, MIN_PRESENT_OPACITY); // floor for present

  ctx.globalAlpha = opacity;
  ctx.drawImage(layers[i], 0, 0, w, h);
}
ctx.globalAlpha = 1.0;

// Advance master clock
masterTime += 1 / frameRate;
masterTime = Math.min(masterTime, duration);

// Periodic re-sync to prevent drift
if (frameCount % 30 === 0) syncLayers();

// End condition
if (masterTime >= duration) {
  pause();
}
```

---

## Controls

| Parameter | Range | Default | Effect |
|-----------|-------|---------|--------|
| Temporal Offset (T) | 0.1s–5.0s | 1.0s | Gap between each adjacent layer. Larger = more temporal spread, more ghosting |
| Present Opacity Floor | 0.0–1.0 | 0.3 | Minimum opacity for the present layer (layer 2). Prevents it from disappearing entirely |
| FFT Smoothing | 0.0–0.95 | 0.85 | `analyser.smoothingTimeConstant`. Higher = more gradual opacity transitions |
| Layer Count | 3 or 5 | 5 | Toggle between 3 layers ([-T, 0, +T]) and 5. Fewer layers = cleaner but less temporal depth |

---

## Browser APIs

- **Web Audio API**: `AudioContext`, `AnalyserNode`, `MediaElementSourceNode` (wired to layer 2 only)
- **Canvas 2D**: compositing with `globalAlpha` per layer
- **HTMLVideoElement**: 5 instances (3 if layer count = 3)
- **MediaRecorder**: for export

---

## Performance Notes

- 5 simultaneous video decodes is the main cost. Modern browsers handle this, but mobile will struggle
- Canvas compositing with `globalAlpha` is GPU-accelerated — not a bottleneck
- `video.currentTime` re-sync every 30 frames is cheap and prevents slow drift accumulation
- `requestVideoFrameCallback()` is preferred over `requestAnimationFrame()` where available — it signals when a new frame is actually ready, reducing redundant draws
- Do not use `video.playbackRate` to simulate seek position — use `currentTime` directly
- If FPS drops: reduce to 3 layers, increase re-sync interval to 60 frames

---

## Edge Cases

| Case | Handling |
|------|----------|
| Future layer seeks past `duration` | Clamp to `duration`, layer freezes on last frame |
| Past layer seeks before `0` | Clamp to `0`, layer freezes on first frame |
| Layer drift over time | Periodic 30-frame re-sync corrects this |
| Audio echo from multiple layers | All layers muted except layer 2 (present) |
| Layer count = 3 | Use indices [0, 2, 4] of the 5-layer setup (or re-index to [-T, 0, +T]) — same logic |
| Video with no audio | FFT still runs (silence = zero energy), opacities collapse to floor values. Layer 2 = present opacity floor, others = near-zero. Works but is visually static — expected |

---

## UI / Aesthetic

Same terminal aesthetic as GLITCH.EXE.

**Palimpsest-specific additions:**
- Single canvas output — no split view
- Below the canvas: a 5-bar frequency/opacity visualizer labeled `PAST-2 / PAST-1 / NOW / FUT-1 / FUT-2`
  - Each bar height = current opacity value for that layer (0.0–1.0)
  - The NOW bar has a visible floor line at the minimum opacity value
- Status line shows `T = Xs` (current temporal offset) during playback
- Debug panel shows per-layer current seek position and opacity value

---

## Done Criteria

- [ ] Single video file loads into 5 simultaneous instances
- [ ] All layers visible during playback (even at low energy)
- [ ] Opacity of each layer visibly responds to audio frequency bands
- [ ] Present layer (NOW) never fully disappears — opacity floor respected
- [ ] Temporal offset control produces visibly more or less ghosting
- [ ] Layer drift does not accumulate over a 2-minute playback
- [ ] Muting behavior correct: only the present layer produces audio
- [ ] 5-bar opacity visualizer labeled correctly and responsive
- [ ] Export works: output captures the composited canvas
