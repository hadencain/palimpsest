# PALIMPSEST.EXE

A browser-based audio-reactive video compositor. The same video plays five times simultaneously, each instance offset in time. Each layer's opacity is driven live by a different frequency band of the video's own audio — bass controls the deepest-past layer, treble controls the furthest-future layer. The result is a video that exists in multiple temporal states at once: past and future bleeding through the present like overexposed film.

No installation. No dependencies. Open `palimpsest.html` in a browser.

---

## Quick Start

1. Open `palimpsest.html` in Chrome or Firefox
2. Drop a video file onto the canvas, or click **LOAD FILE**
3. Click **▶ PLAY** or press **EXPORT** to begin

---

## How It Works

Five `<video>` elements share the same source file. They are staggered in time at offsets `[-2T, -T, 0, +T, +2T]` from a master clock, where `T` is the temporal offset parameter. Each frame, all five are composited onto a canvas in order (deepest past first) using `globalAlpha` tied to the energy in their assigned frequency band.

| Layer | Position | Frequency Band |
|-------|----------|----------------|
| PAST-2 | −2T from now | Bass (0–938 Hz) |
| PAST-1 | −T from now | Low-mid (1.0–3.7 kHz) |
| NOW | 0 (present) | Mid (3.8–9.4 kHz) |
| FUT-1 | +T from now | High-mid (9.5–15 kHz) |
| FUT-2 | +2T from now | Treble (15–20.6 kHz) |

Audio plays only from the NOW layer. All other instances are muted.

---

## Controls

### Transport

Located on the canvas overlay (bottom-right corner).

| Button | Action |
|--------|--------|
| ▶ PLAY | Start playback. Initializes audio context and begins compositing. |
| ❙❙ PAUSE | Pause playback. Stops all layers and the animation loop. If recording, stops the recording and triggers download. |

### Parameter Sliders

Located in the control strip below the visualizer.

| Parameter | Range | Default | Effect |
|-----------|-------|---------|--------|
| **Temporal Offset** | 0.1s – 5.0s | 1.0s | Time gap between adjacent layers. Larger = more ghosting, more temporal spread between past/future. |
| **Present Floor** | 0.00 – 1.00 | 0.30 | Minimum opacity for the NOW layer. Prevents the present from disappearing entirely even during silence. |
| **FFT Smoothing** | 0.00 – 0.95 | 0.85 | Web Audio `smoothingTimeConstant`. Higher = slower, more gradual opacity transitions. Lower = snappier, more reactive. |
| **Layers** | 3 or 5 | 5 | Toggle between 5-layer mode (PAST-2, PAST-1, NOW, FUT-1, FUT-2) and 3-layer mode (PAST-1, NOW, FUT-1). Fewer layers = cleaner but less temporal depth. |

### Footer Buttons

| Button | Behavior |
|--------|----------|
| **LOAD FILE** | Opens a file picker. Accepts any video format the browser supports. |
| **LOOP** | Toggle seamless loop. When enabled, all layers reset to their offset positions at end-of-video and playback continues without stopping. |
| **RECORD** | Appears when playback is active. Starts a real-time capture of the composited canvas with the NOW layer's audio. Click again to stop — triggers automatic download as `.webm`. |
| **EXPORT** | Appears after a file is loaded. One-click shortcut: starts playback and begins recording immediately. Use this to capture the full video from the start without manually clicking PLAY then RECORD. |
| **AR** | Enables AR (Audio-Reactive) mode. Overrides all four parameter sliders with live audio-driven values. See below. |
| **DEBUG** | Toggles the debug panel, which shows per-layer seek position and opacity values in real time. |

---

## AR Mode

When active, AR mode drives all temporal and opacity parameters directly from frequency analysis each frame. The sliders dim and become non-interactive, but their thumbs continue to move, showing the audio-driven values.

| Parameter | Driver | Range |
|-----------|--------|-------|
| Temporal Offset | Bass energy | 0.5s – 5.0s |
| Present Floor | Low-mid energy | 0.00 – 1.00 |
| FUT-1 / FUT-2 playback rate | High-mid energy | 0.5× – 2.0× |
| PAST-1 / PAST-2 playback rate | Treble energy | 0.5× – 2.0× |

Bass hits expand the temporal spread — layers pull further apart in time. Low-mid energy raises the opacity floor of the present. High-mid and treble speed up or slow down the future and past layers respectively, making them drift out of sync in a way that accumulates and recovers with the music.

**Toggling AR off** restores all layer playback rates to 1.0× and re-enables the sliders at their last audio-driven positions.

**Loading a new file while AR is active** automatically exits AR mode.

---

## Recording & Export

Both **RECORD** and **EXPORT** capture the composited canvas stream (what you see) combined with the audio from the NOW layer.

- Output format: `.webm` (VP8/Opus)
- Output filename: `palimpsest-<timestamp>.webm`
- Download triggers automatically when recording stops

**RECORD** — manual start/stop during playback. Use this to capture a specific moment.

**EXPORT** — starts playback and recording together from the current loaded file. Use this to capture a full take from the beginning.

Pausing playback while recording is active stops the recording and triggers the download.

---

## Visualizer

The five-bar strip below the canvas shows the current opacity/energy value for each layer in real time.

- Bar height = current opacity (0–100%)
- The NOW bar is brighter than the others
- The horizontal hairline on the NOW column shows the current Present Floor value

---

## Debug Panel

Click **DEBUG** to toggle a panel showing live per-layer state:

| Field | Description |
|-------|-------------|
| `t: Xs` | Current seek position for that layer |
| `α: 0.XX` | Current opacity (energy) for that layer |

NOW layer values are highlighted white.

---

## File Support

Accepts any video format the browser supports: `.mp4`, `.mov`, `.webm`, `.mkv`. H.264 and VP9 sources work in all modern browsers. ProRes and other high-bitrate codecs may work depending on browser/OS.

Audio in the source file is required for reactive behavior. Silent video produces near-zero opacities for all non-NOW layers — the app still runs but will look static.

---

## Browser Compatibility

| Browser | Status |
|---------|--------|
| Chrome 90+ | Full support |
| Firefox 88+ | Full support |
| Safari 15+ | Partial — `video/webm` export may not be supported |
| Edge (Chromium) | Full support |

Requires: Web Audio API, Canvas 2D, MediaRecorder, `<video>` with multiple instances of the same source.

---

## Architecture Notes

See `design.md` for full algorithm and technical spec.

- Single file: `palimpsest.html` — no build step, no dependencies
- Master clock: `layers[2].currentTime` is ground truth
- Drift correction: layer seek positions re-sync to master clock every 30 frames (every frame in AR mode)
- Audio graph: `MediaElementSource(layers[2]) → AnalyserNode → destination + MediaStreamDestination`
- The `MediaStreamDestination` provides the audio track for recording
