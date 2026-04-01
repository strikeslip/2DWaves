# TREMORS — Implementation Plan
## 2D Seismic Wave Propagation + Granular Sonification Instrument
### SOS/SeisClaw Suite | sos.allshookup.org | seisclaw.com

---

## Overview

TREMORS is a new standalone browser instrument in the SOS (Sounds of Seismic) suite at [sos.allshookup.org](http://sos.allshookup.org), alongside `ONE.html`, `flow.html`, `seis.html`, and `ShadowZone.html`. It is a 2D acoustic finite difference wave propagation simulator whose synthetic seismograms feed a GranularEngine for real-time sonification.

The physics engine is derived from the same SW4/CIG lineage ([github.com/geodynamics/sw4](https://github.com/geodynamics/sw4)) that underlies USGS seismic hazard calculations, reduced to a 2D scalar acoustic stencil that runs in vanilla JS in a browser tab. The reference implementation that confirmed browser viability is [jtiscione/webassembly-wave](https://github.com/jtiscione/webassembly-wave) ([live demo](https://jtiscione.github.io/webassembly-wave/index.html)).

**Final deliverable:** `tremors.html` — single HTML file, all JS inline, zero dependencies, deployable via SFTP to allshookup.org exactly as all other SOS instruments.

---

## Architecture summary

```
[ Velocity model (2D Vp array) ]
          |
[ FDTD stencil loop ]  ←  [ Ricker wavelet source ]
          |
[ Pressure field arrays: P_cur, P_prv, P_nxt ]
          |
[ Canvas render (ImageData) ]    [ Virtual seismograph ]
                                          |
                                 [ GranularEngine ]
                                          |
                                 [ Web Audio output ]
```

All components vanilla JS. No external dependencies. No build tools. No WebAssembly required — plain JS handles 400×300 grid at real-time speeds on modern hardware. Reference: [jtiscione benchmark](https://github.com/jtiscione/webassembly-wave) confirms JS is sufficient at this scale.

---

## Build phases

```
Phase 1 — Grid + velocity model + pressure arrays
Phase 2 — CFL condition + stencil loop (headless)
Phase 3 — Canvas render + requestAnimationFrame loop
Phase 4 — Ricker wavelet refinement + geological presets
Phase 5 — Virtual seismograph display
Phase 6 — GranularEngine bridge (batch mode)
Phase 7 — Polish, identity, deployment
Phase 8 — Streaming bridge (real-time, deferred)
```

Each phase = one Opus 4.6 session. Each session produces a **complete, runnable, standalone HTML file**. Never a fragment.

---

## Session discipline — critical

**Each session prompt must contain the complete output of the previous session verbatim** in the `[PASTE COMPLETE tremors_pN.html CONTENT HERE]` slot.

Opus 4.6 starts every conversation completely blank. It has no memory of previous sessions, no knowledge of variable names chosen, no awareness of architectural decisions made. The only way to give it that context is to paste the entire file built in the previous phase directly into the prompt. The HTML file IS the memory. Paste the whole thing, every time, without summarising or paraphrasing it.

This applies equally to every Anthropic model — Opus, Sonnet, Haiku. New conversation = blank slate. The file carries the state forward, not the model.

---

## Reference URLs

| Resource | URL |
|---|---|
| SOS instrument suite | http://sos.allshookup.org |
| allshookup.org | http://www.allshookup.org |
| SeisClaw / SEISCLAW | http://seisclaw.com |
| CIG / geodynamics.org | https://geodynamics.org |
| CIG GitHub org | https://github.com/geodynamics |
| SW4 repo | https://github.com/geodynamics/sw4 |
| SW4 theory (LLNL) | https://computing.llnl.gov/projects/serpentine-wave-propagation/theory |
| SW4 software (LLNL) | https://computing.llnl.gov/projects/serpentine-wave-propagation/software |
| seismic_cpml repo | https://github.com/geodynamics/seismic_cpml |
| jtiscione wave demo | https://jtiscione.github.io/webassembly-wave/index.html |
| jtiscione repo | https://github.com/jtiscione/webassembly-wave |
| OpenSWPC | https://github.com/OpenSWPC/OpenSWPC |
| FD_ACOUSTIC reference | https://github.com/florianwittkamp/FD_ACOUSTIC |
| EarthScope FDSN | https://service.earthscope.org |
| Louie lecture (UNR) | https://youtu.be/-ZrIK3cfjFk?si=SxVMSyPqxEG7zaRh |
| Nevada shake zoning | https://sites.google.com/view/nevada-shake-zoning |

---

## Phase 1 — Grid, velocity model, pressure arrays

**Goal:** Valid data structure foundation. No physics. Browser opens, console logs confirm arrays correctly sized and populated.

**Deliverable:** `tremors_p1.html`

**Physics basis:** The velocity model encodes Vp (P-wave velocity in m/s) at every grid cell — the same physical property discussed in Louie's UNR lecture ([youtu.be/-ZrIK3cfjFk](https://youtu.be/-ZrIK3cfjFk?si=SxVMSyPqxEG7zaRh)) and the foundation of all SW4 simulations ([github.com/geodynamics/sw4](https://github.com/geodynamics/sw4)). Basin cells (Vp ~1500 m/s) vs bedrock cells (Vp ~4000 m/s) is the geological contrast that produces wave trapping and amplification.

```
You are building TREMORS — a new standalone browser instrument in
the SOS (Sounds of Seismic) suite at allshookup.org. TREMORS is a
2D acoustic finite difference wave propagation simulator that
produces synthetic seismograms, rendered visually on a Canvas and
sonified via a GranularEngine (to be integrated in a later phase).

STRICT CONSTRAINTS — non-negotiable:
- Vanilla JS only. Zero external dependencies. No imports, no npm,
  no CDN links.
- Single HTML file. All JS inline in <script> tags.
- No ES6 modules. Use plain functions and var/let/const.
- KISS at every decision. Simplest correct implementation wins.
- Do not write the GranularEngine yet. Do not write Canvas
  rendering yet. Data structures only this phase.

YOUR TASK — Phase 1: Define all data structures.

Build the following inside a single DOMContentLoaded handler:

1. GRID PARAMETERS
   - WIDTH = 400 (cells)
   - HEIGHT = 300 (cells)
   - DX = 50.0 (metres per cell — physical spacing)
   - DT = null (to be calculated in Phase 2)

2. VELOCITY MODEL — a flat Float32Array of size WIDTH*HEIGHT.
   Populate with a geologically plausible 2D basin cross-section:
   - Background (bedrock): Vp = 4000.0 m/s everywhere
   - Basin shape: a half-ellipse centred horizontally, top at
     row 20, bottom at row 200, width spanning columns 100 to 300.
     Inside the basin: Vp = 1500.0 m/s (soft sediment)
   - Helper function: getIdx(x, y) returns y*WIDTH + x (flat index)

3. PRESSURE ARRAYS — three Float32Array of size WIDTH*HEIGHT:
   - P_cur (current timestep)
   - P_prv (previous timestep)
   - P_nxt (next timestep — computed each step, then rotated)
   All initialised to 0.0.

4. SPONGE MASK — a Float32Array of size WIDTH*HEIGHT.
   Border zone = 30 cells wide on all four edges.
   Inside border zone: values taper from 1.0 (at inner edge)
   down to 0.88 (at outer edge) using linear interpolation.
   Outside border zone: 1.0.
   This will be multiplied against P_nxt each step to damp
   outgoing waves at boundaries — a simple absorbing boundary
   condition replacing the PML used in SW4
   (ref: https://computing.llnl.gov/projects/serpentine-wave-propagation/theory).

5. SEISMOGRAM BUFFER — a Float32Array of size 8000 (pre-allocated,
   enough for ~80 seconds at 100 samples/sec). Initialised to 0.0.
   Let seismoIdx = 0 (current write position).

6. STATION — define stationX = 280, stationY = 80 (virtual
   seismograph position, in grid cells).

After defining everything, log to console:
- Grid dimensions
- Vp at the basin centre cell (should be 1500)
- Vp at a bedrock cell (should be 4000)
- Sponge value at corner cell [0,0] (should be ~0.88)
- Sponge value at centre cell (should be 1.0)
- All three pressure array lengths (should all be 120000)

Output a complete, runnable tremors_p1.html. The page body can be
blank — all verification is via console.log.
```

---

## Phase 2 — CFL condition + stencil loop (headless)

**Goal:** Physics engine running. No rendering. Console logs confirm waves propagating — seismogram buffer written with non-zero values after N steps.

**Deliverable:** `tremors_p2.html`

**Physics basis:** The CFL (Courant-Friedrichs-Lewy) stability condition is the fundamental constraint linking timestep, grid spacing, and wave speed in all finite difference seismic codes including SW4 ([github.com/geodynamics/sw4](https://github.com/geodynamics/sw4)) and seismic_cpml ([github.com/geodynamics/seismic_cpml](https://github.com/geodynamics/seismic_cpml)). The Ricker wavelet is the standard seismic source function used across the entire FDTD literature. The stencil itself — four-neighbour 2nd order Laplacian — is the Madariaga/Levander lineage documented in OpenSWPC ([github.com/OpenSWPC/OpenSWPC](https://github.com/OpenSWPC/OpenSWPC)).

```
You are building TREMORS — a 2D acoustic finite difference wave
propagation simulator (SOS instrument, allshookup.org).

STRICT CONSTRAINTS:
- Vanilla JS only. Zero external dependencies. Single HTML file.
- KISS. No rendering yet. Physics engine only this phase.

CARRY FORWARD from Phase 1 (include all of this verbatim in your
output — do not reference it, reproduce it):
[PASTE COMPLETE tremors_p1.html CONTENT HERE]

YOUR TASK — Phase 2: CFL condition + finite difference stencil.

1. CFL STABILITY CONDITION
   After the data structures from Phase 1, calculate:

   VP_MAX = maximum value in the velocity model array (4000.0)
   DT = (DX / VP_MAX) * 0.45

   The 0.45 factor gives 45% of the theoretical maximum timestep —
   a safe margin below the CFL limit of 1/sqrt(2) ≈ 0.707 for 2D.
   Log DT to console (expect ~0.0056 seconds).

2. RICKER WAVELET SOURCE
   Define source position: srcX = 200, srcY = 10 (near surface,
   centre of grid).
   Define dominant frequency: F0 = 8.0 Hz.
   Define source duration: SRC_STEPS = Math.ceil(2.0 / (F0 * DT))
   steps (two full cycles).

   Function rickerValue(stepN):
     t = stepN * DT
     t0 = 1.0 / F0
     u = Math.PI * F0 * (t - t0)
     return (1 - 2*u*u) * Math.exp(-u*u)
   Standard Ricker wavelet centred at t=t0.

3. FINITE DIFFERENCE STENCIL
   Function stepOnce(stepN):

   a) For every interior cell (x from 1 to WIDTH-2,
      y from 1 to HEIGHT-2):

      idx = getIdx(x, y)

      Laplacian = P_cur[getIdx(x+1,y)] + P_cur[getIdx(x-1,y)]
                + P_cur[getIdx(x,y+1)] + P_cur[getIdx(x,y-1)]
                - 4.0 * P_cur[idx]

      vp = velocity[idx]
      factor = (vp * DT / DX) * (vp * DT / DX)

      P_nxt[idx] = 2.0*P_cur[idx] - P_prv[idx] + factor * Laplacian

   b) Inject source (if stepN < SRC_STEPS):
      P_nxt[getIdx(srcX, srcY)] += rickerValue(stepN) * 1e9
      (Large amplitude ensures the wave is visible above
      numerical noise.)

   c) Apply sponge mask:
      For every cell: P_nxt[idx] *= sponge[idx]

   d) Rotate arrays:
      temp = P_prv
      P_prv = P_cur
      P_cur = P_nxt
      P_nxt = temp

   e) Record seismogram:
      If seismoIdx < seismogram.length:
        seismogram[seismoIdx++] = P_cur[getIdx(stationX, stationY)]

4. HEADLESS TEST RUN
   Run stepOnce() 500 times in a loop.
   After the loop, log:
   - seismogram[0] through seismogram[9] (first 10 samples)
   - Max absolute value in seismogram[0..499]
   - The step number at which max occurs

   Non-zero values confirm the wave engine is working. A clear
   arrival peak confirms correct propagation.

CRITICAL INDEXING NOTE: The Laplacian for cell (x,y) uses
four neighbours: (x+1,y), (x-1,y), (x,y+1), (x,y-1).
Be precise — a common bug is mixing x and y in getIdx calls.
getIdx(x, y) = y*WIDTH + x. Double-check every neighbour call.

Output complete runnable tremors_p2.html.
```

---

## Phase 3 — Canvas render + requestAnimationFrame loop

**Goal:** Visible wavefield animation. Waves expand from source, slow in basin, reflect off boundaries. The visual is directly analogous to the Louie basin animations (https://sites.google.com/view/nevada-shake-zoning) and the jtiscione wave demo (https://jtiscione.github.io/webassembly-wave/index.html).

**Deliverable:** `tremors_p3.html`

```
You are building TREMORS — a 2D acoustic finite difference wave
propagation simulator (SOS instrument, allshookup.org).

STRICT CONSTRAINTS:
- Vanilla JS only. Zero external dependencies. Single HTML file.
- KISS. Canvas 2D API only — no WebGL.
- Page background: #000000. Canvas border: none.

CARRY FORWARD from Phase 2:
[PASTE COMPLETE tremors_p2.html CONTENT HERE]

YOUR TASK — Phase 3: Canvas render + animation loop.

Remove the headless 500-step loop from Phase 2. Replace with:

1. CANVAS SETUP
   Add to HTML body:
   <canvas id="tremors" width="512" height="384"></canvas>

   const canvas = document.getElementById('tremors')
   const ctx = canvas.getContext('2d')
   const imageData = ctx.createImageData(512, 384)
   const pixels = imageData.data  // Uint8ClampedArray, RGBA

2. RENDER FUNCTION
   Function renderFrame():

   For every canvas pixel (px from 0 to 511, py from 0 to 383):
     Map to grid cell:
       gx = Math.floor(px * WIDTH / 512)
       gy = Math.floor(py * HEIGHT / 384)

     val = P_cur[getIdx(gx, gy)]
     scaled = val / 1e8
     clamped = Math.max(-1, Math.min(1, scaled))

     Colour mapping:
       clamped > 0: blue = Math.floor(clamped * 255), r=0, g=0
       clamped < 0: red = Math.floor(-clamped * 255), b=0, g=0
       zero: all channels 0
     Alpha = 255 always.

     Write to pixels[(py*512 + px)*4 + 0] = red
     Write to pixels[(py*512 + px)*4 + 1] = green
     Write to pixels[(py*512 + px)*4 + 2] = blue
     Write to pixels[(py*512 + px)*4 + 3] = 255

   ctx.putImageData(imageData, 0, 0)

   Colour convention: blue = compression (positive pressure),
   red = rarefaction (negative pressure), black = zero.
   This matches the standard seismic wavefield visualisation
   palette used in SW4 output and the jtiscione demo
   (https://github.com/jtiscione/webassembly-wave).

3. VELOCITY MODEL OVERLAY (draw once at start)
   After canvas setup, before animation loop starts:
   ctx.strokeStyle = 'rgba(255,255,255,0.15)'
   ctx.lineWidth = 1
   ctx.beginPath()
   ctx.ellipse(256, 110*384/300, 100*512/400, 90*384/300,
               0, 0, Math.PI)
   ctx.stroke()
   This draws a dim outline of the basin shape so geology
   is visible under the wavefield.

4. ANIMATION LOOP
   let stepCount = 0
   let running = true
   const STEPS_PER_FRAME = 4
   (4 simulation steps per animation frame — balances visual
   smoothness vs simulation speed.)

   Function animate():
     if (!running) return
     for (let i = 0; i < STEPS_PER_FRAME; i++) {
       stepOnce(stepCount++)
     }
     renderFrame()
     requestAnimationFrame(animate)

   animate()

5. BASIC CONTROLS
   Space bar: toggle running (pause/resume)
   'r' key: reset — zero all pressure arrays, reset
            stepCount=0, seismoIdx=0, resume

   Log stepCount to console every 100 frames to confirm
   the loop is advancing.

Output complete runnable tremors_p3.html.
```

---

## Phase 4 — Geological presets + UI controls

**Goal:** Four geologically meaningful velocity model presets selectable at runtime. Source frequency selectable. HUD showing simulation state. Source and station markers on canvas.

**Deliverable:** `tremors_p4.html`

**Geological basis:** The four presets represent real geological scenarios described in Louie's UNR lecture (https://youtu.be/-ZrIK3cfjFk?si=SxVMSyPqxEG7zaRh) and in SW4's basin hazard applications (https://sites.google.com/view/nevada-shake-zoning). Deep basin = Reno/Sparks. Shallow basin = smaller valley. Layered = horizontal stratigraphy. Homogeneous = calibration reference.

```
You are building TREMORS — a 2D acoustic finite difference wave
propagation simulator (SOS instrument, allshookup.org).

STRICT CONSTRAINTS:
- Vanilla JS only. Zero dependencies. Single HTML file.
- UI: minimal, monospace font, dark theme (#000 bg, #0f0 or #fff
  text). Controls sit below the canvas, not overlaid on it.

CARRY FORWARD from Phase 3:
[PASTE COMPLETE tremors_p3.html CONTENT HERE]

YOUR TASK — Phase 4: Geological presets + UI controls.

1. PRESET VELOCITY MODELS
   Refactor velocity model population into function
   buildVelocityModel(presetName) called on init and reset.

   'deep_basin':
     Background Vp = 4000.0 m/s.
     Half-ellipse: centre col 200, top row 20, bottom row 220,
     half-width 100 cols.
     Basin Vp = 1500.0 m/s.
     (Models Reno/Sparks basin ~1km deep, ref:
     https://sites.google.com/view/nevada-shake-zoning)

   'shallow_basin':
     Background Vp = 3500.0 m/s.
     Half-ellipse: centre col 200, top row 20, bottom row 100,
     half-width 120 cols.
     Basin Vp = 1800.0 m/s.
     (Faster sediment, shallower — smaller valley scenario.)

   'layered':
     Rows 0-50:    Vp = 1200.0 m/s (very soft near-surface)
     Rows 51-120:  Vp = 2200.0 m/s (consolidated sediment)
     Rows 121-300: Vp = 4500.0 m/s (bedrock)
     No basin shape — flat horizontal layers only.
     (Classic exploration seismology velocity model.)

   'homogeneous':
     Vp = 3000.0 m/s everywhere. No geology.
     For reference and calibration — circular wavefronts only.

2. SOURCE PARAMETERS UI
   Below canvas, add controls (monospace, dark theme):

   Preset selector: [deep_basin] [shallow_basin] [layered]
                    [homogeneous]
   Frequency selector: [4Hz] [8Hz] [12Hz] [20Hz]
   [RESET] button

   Changing preset or frequency calls reset() which:
   - Calls buildVelocityModel(newPreset)
   - Recalculates VP_MAX from new velocity array
   - Recalculates DT = (DX / VP_MAX) * 0.45
   - Recalculates SRC_STEPS = Math.ceil(2.0 / (F0 * DT))
   - Zeros P_cur, P_prv, P_nxt, seismogram
   - Resets stepCount = 0, seismoIdx = 0
   - Resumes animation if paused

3. STATION MARKER
   At start of each renderFrame(), draw small white cross
   (3×3 pixels) at station position scaled to canvas coords:
   canvasX = Math.floor(stationX * 512 / WIDTH)
   canvasY = Math.floor(stationY * 384 / HEIGHT)

4. SOURCE MARKER
   At start of each renderFrame(), draw small red dot
   (3×3 pixels) at source position scaled to canvas coords.
   Red = #ff0000.

5. HUD
   Top-left corner of canvas, white monospace 11px, after
   putImageData (so it draws over the wavefield):
   Line 1: 'TREMORS | step: ' + stepCount
   Line 2: 'Vp_max: ' + VP_MAX + ' m/s | dt: ' +
            DT.toFixed(5) + 's'
   Line 3: 'f0: ' + F0 + 'Hz | src: (' + srcX + ',' +
            srcY + ')'

Output complete runnable tremors_p4.html.
```

---

## Phase 5 — Virtual seismograph display

**Goal:** Live seismogram trace accumulating in real time below the wavefield. Green trace, dark background, P-wave arrival marker. Visually consistent with SeisClaw's waveform display at [sos.allshookup.org](http://sos.allshookup.org).

**Deliverable:** `tremors_p5.html`

**Seismological basis:** The virtual seismograph records P_cur at the station cell each timestep — identical in principle to real EarthScope FDSN seismograph recordings at [service.earthscope.org](https://service.earthscope.org). The waveform produced is format-equivalent to MiniSEED trace data already processed by SeisClaw's existing pipeline.

```
You are building TREMORS — a 2D acoustic finite difference wave
propagation simulator (SOS instrument, allshookup.org).

STRICT CONSTRAINTS:
- Vanilla JS only. Zero dependencies. Single HTML file.
- Seismogram display uses a second Canvas element, separate from
  the main wavefield canvas.
- Aesthetic: dark background, green trace — consistent with the
  SeisClaw instrument aesthetic at sos.allshookup.org.

CARRY FORWARD from Phase 4:
[PASTE COMPLETE tremors_p4.html CONTENT HERE]

YOUR TASK — Phase 5: Live seismogram display.

1. SEISMOGRAM CANVAS
   Add below main canvas in HTML:
   <canvas id="seismo" width="512" height="120"></canvas>
   Background: #000000.

   const seismoCanvas = document.getElementById('seismo')
   const seismoCtx = seismoCanvas.getContext('2d')

2. SEISMOGRAM RENDER FUNCTION
   Function renderSeismogram():

   a) Clear to black:
      seismoCtx.fillStyle = '#000'
      seismoCtx.fillRect(0, 0, 512, 120)

   b) Centre line (zero amplitude reference):
      seismoCtx.strokeStyle = 'rgba(0,255,0,0.2)'
      seismoCtx.lineWidth = 1
      seismoCtx.beginPath()
      seismoCtx.moveTo(0, 60)
      seismoCtx.lineTo(512, 60)
      seismoCtx.stroke()

   c) Find displayable window:
      Last 512 samples written (seismogram[seismoIdx-512] to
      seismogram[seismoIdx-1]), or all samples if fewer than 512.
      Find peak = max absolute value in that window.
      If peak == 0, return early (nothing to draw yet).

   d) Draw trace:
      seismoCtx.strokeStyle = '#00ff00'
      seismoCtx.lineWidth = 1
      seismoCtx.beginPath()
      For px from 0 to 511:
        sampleIdx = seismoIdx - 512 + px
        if sampleIdx < 0: sampleIdx = 0
        val = seismogram[sampleIdx] / peak
        py = 60 - val * 55
        if px == 0: seismoCtx.moveTo(px, py)
        else: seismoCtx.lineTo(px, py)
      seismoCtx.stroke()

   e) Station label:
      seismoCtx.fillStyle = '#0f0'
      seismoCtx.font = '10px monospace'
      seismoCtx.fillText(
        'STATION (' + stationX + ',' + stationY + ')',
        4, 12)

3. P-WAVE ARRIVAL MARKER
   Detect first significant arrival: first index i where
   Math.abs(seismogram[i]) > peak * 0.01.
   If found and within the current display window:
   - Draw dim yellow vertical line at corresponding canvas x
   - Label 'P' in yellow (#ffff00), 10px monospace, above line
   seismoCtx.strokeStyle = 'rgba(255,255,0,0.4)'

4. INTEGRATE INTO ANIMATION LOOP
   Call renderSeismogram() after renderFrame() in animate().

5. SEISMOGRAM RESET
   In reset(), also clear the seismogram canvas to black:
   seismoCtx.fillStyle = '#000'
   seismoCtx.fillRect(0, 0, 512, 120)

Output complete runnable tremors_p5.html.
```

---

## Phase 6 — GranularEngine bridge (batch mode)

**Goal:** Synthetic seismogram feeds the GranularEngine for sonification. Full audio-visual instrument. The bridge is one-directional — simulation produces waveform data, GranularEngine consumes it. This is the same data handoff SeisClaw performs with real EarthScope MiniSEED data ([service.earthscope.org](https://service.earthscope.org)), making TREMORS output format-compatible with the existing SeisClaw pipeline.

**Deliverable:** `tremors_p6.html`

```
You are building TREMORS — a 2D acoustic finite difference wave
propagation and granular sonification instrument (SOS suite,
allshookup.org). This phase integrates the GranularEngine to
sonify the synthetic seismogram produced by the wave simulation.

The GranularEngine here is the same class architecture used in
SeisClaw at seisclaw.com / sos.allshookup.org. The synthetic
seismogram feeds it identically to how real EarthScope waveforms
feed SeisClaw (ref: https://service.earthscope.org).

STRICT CONSTRAINTS:
- Vanilla JS only. Zero dependencies. Single HTML file.
- Web Audio API only for audio.
- GranularEngine self-contained in this file.
- Do NOT modify the wave simulation code from Phase 5.
- Bridge is one-directional: simulation → engine only.
  Engine never writes back to simulation arrays.

CARRY FORWARD from Phase 5:
[PASTE COMPLETE tremors_p5.html CONTENT HERE]

YOUR TASK — Phase 6: GranularEngine + seismogram bridge.

1. WEB AUDIO CONTEXT
   let audioCtx = null

   Function initAudio():
     if (audioCtx) return
     audioCtx = new (window.AudioContext ||
                     window.webkitAudioContext)()
   (Must be called only on user gesture — browser autoplay
   policy requirement.)

2. GRANULAR ENGINE CLASS

   function GranularEngine(audioCtx) {
     this.ctx = audioCtx
     this.buffer = null
     this.grainSize = 0.08      // seconds per grain
     this.grainOverlap = 4      // grains active simultaneously
     this.playbackRate = 1.0    // position scan speed
     this.position = 0.0        // current read head (0.0–1.0)
     this.masterGain = audioCtx.createGain()
     this.masterGain.gain.value = 0.7
     this.masterGain.connect(audioCtx.destination)
     this.playing = false
     this.intervalId = null
   }

   GranularEngine.prototype.loadWaveform =
     function(float32Array, sampleRate) {
       // Find peak absolute value
       var peak = 0
       for (var i = 0; i < float32Array.length; i++) {
         if (Math.abs(float32Array[i]) > peak)
           peak = Math.abs(float32Array[i])
       }
       if (peak === 0) return  // nothing to load

       // Normalise to -1.0 / +1.0
       var normalised = new Float32Array(float32Array.length)
       for (var i = 0; i < float32Array.length; i++) {
         normalised[i] = float32Array[i] / peak
       }

       // Create AudioBuffer
       // sampleRate = Math.round(1.0 / DT) from simulation.
       // Browser pitch-shifts on playback, transposing infrasonic
       // seismic frequencies (~8Hz) into audible range.
       // This is correct and desirable for sonification.
       var buf = this.ctx.createBuffer(
         1, normalised.length, sampleRate)
       buf.getChannelData(0).set(normalised)
       this.buffer = buf
     }

   GranularEngine.prototype.spawnGrain = function() {
     if (!this.buffer || !this.playing) return
     var src = this.ctx.createBufferSource()
     src.buffer = this.buffer
     var grainGain = this.ctx.createGain()
     grainGain.gain.value = 1.0 / this.grainOverlap
     // Hann window envelope
     var now = this.ctx.currentTime
     grainGain.gain.setValueAtTime(0, now)
     grainGain.gain.linearRampToValueAtTime(
       1.0 / this.grainOverlap,
       now + this.grainSize * 0.5)
     grainGain.gain.linearRampToValueAtTime(
       0, now + this.grainSize)
     src.connect(grainGain)
     grainGain.connect(this.masterGain)
     var offset = this.position * this.buffer.duration
     src.start(now, offset, this.grainSize)
     // Advance read position
     this.position += this.playbackRate *
                      (this.grainSize / this.grainOverlap) /
                      this.buffer.duration
     if (this.position >= 1.0) this.position = 0.0
   }

   GranularEngine.prototype.start = function() {
     this.playing = true
     var grainInterval =
       (this.grainSize / this.grainOverlap) * 1000
     var self = this
     this.intervalId = setInterval(function() {
       self.spawnGrain()
     }, grainInterval)
   }

   GranularEngine.prototype.stop = function() {
     this.playing = false
     if (this.intervalId) clearInterval(this.intervalId)
     this.intervalId = null
   }

   GranularEngine.prototype.setPosition = function(p) {
     this.position = Math.max(0, Math.min(1, p))
   }

   GranularEngine.prototype.setGrainSize = function(s) {
     this.grainSize = Math.max(0.01, Math.min(0.5, s))
   }

   GranularEngine.prototype.setPlaybackRate = function(r) {
     this.playbackRate = Math.max(0.01, Math.min(4.0, r))
   }

3. BRIDGE FUNCTION
   var granularEngine = null

   Function sonifySeismogram():
     initAudio()
     if (!granularEngine)
       granularEngine = new GranularEngine(audioCtx)
     var recorded = seismogram.slice(0, seismoIdx)
     if (recorded.length < 10) return
     granularEngine.stop()
     granularEngine.loadWaveform(
       recorded, Math.round(1.0 / DT))
     granularEngine.start()

4. UI ADDITIONS
   Below existing controls add:

   [SONIFY] button → sonifySeismogram()
   [STOP AUDIO] button → if (granularEngine) granularEngine.stop()

   Slider: Grain size
     min=0.01 max=0.5 step=0.01 default=0.08
     label: 'Grain: ' + value + 's'
     oninput: if (granularEngine)
              granularEngine.setGrainSize(parseFloat(this.value))

   Slider: Scan speed
     min=0.1 max=4.0 step=0.1 default=1.0
     label: 'Speed: ' + value + 'x'
     oninput: if (granularEngine)
              granularEngine.setPlaybackRate(parseFloat(this.value))

   Slider: Position
     min=0.0 max=1.0 step=0.01 default=0.0
     label: 'Pos: ' + value
     oninput: if (granularEngine)
              granularEngine.setPosition(parseFloat(this.value))

5. AUTO-SONIFY OPTION
   Checkbox: 'Auto-sonify on reset' (id='autoSonify')
   In animate(): if stepCount === 2000 and
   document.getElementById('autoSonify').checked:
     sonifySeismogram()

Output complete runnable tremors_p6.html.
```

---

## Phase 7 — Polish, identity, deployment

**Goal:** Instrument-grade finish. TREMORS is ready to live at `allshookup.org/tremors.html` alongside `ONE.html`, `flow.html`, `seis.html`, and `ShadowZone.html`. Three stations recording simultaneously — basin amplification and delay made audible across station positions. This is the geophysical point of the instrument.

**Deliverable:** `tremors.html` — final file, SFTP-ready.

```
You are finalising TREMORS — a completed 2D seismic wave
propagation and granular sonification instrument for
allshookup.org/tremors.html, part of the SOS (Sounds of Seismic)
suite at sos.allshookup.org alongside ONE.html, flow.html,
seis.html, and ShadowZone.html.

CARRY FORWARD from Phase 6:
[PASTE COMPLETE tremors_p6.html CONTENT HERE]

YOUR TASK — Phase 7: Final instrument polish.

1. PAGE IDENTITY
   <title>TREMORS | SOS</title>

   Header above canvas (monospace, small, dim #888):
   'TREMORS — 2D Seismic Wave Propagation | sos.allshookup.org'

   Footer below all controls (very small, very dim #555):
   'Acoustic FDTD wave engine. Synthetic seismogram → granular
    synthesis. Physics: SW4/CIG lineage
    (github.com/geodynamics/sw4). D.V. Rogers /
    allshookup.org 2026'

2. VELOCITY MODEL BACKGROUND LAYER
   On reset/preset change, render geology to an offscreen canvas
   (document.createElement('canvas'), same size as main canvas):

   For every grid cell:
     gx, gy → canvas px, py (same mapping as renderFrame)
     vp = velocity[getIdx(gx, gy)]
     if vp > 3000:  pixel = very dim dark grey rgba(40,40,40,255)
     if vp 1500-3000: pixel = dim blue-grey rgba(30,40,60,255)
     if vp < 1500:  pixel = dim warm amber rgba(60,50,30,255)

   Each renderFrame(): draw geology offscreen canvas first:
     ctx.drawImage(geologyCanvas, 0, 0)
   Then ctx.putImageData(imageData, 0, 0) on top.
   Geology is always visible under the wavefield.

3. KEYBOARD SHORTCUTS
   Add handlers:
   S key → sonifySeismogram()
   X key → if (granularEngine) granularEngine.stop()
   (Space and R already from Phase 3.)

   Display legend below seismogram canvas (monospace, dim):
   'SPACE: pause  R: reset  S: sonify  X: stop audio'

4. THREE STATIONS
   Replace single station with three stations, each with its
   own seismogram buffer (Float32Array size 8000) and index:

   Station 1: stationX=280, stationY=80  — inside basin
   Station 2: stationX=320, stationY=60  — basin edge
   Station 3: stationX=360, stationY=40  — outside (bedrock)

   In stepOnce() record all three each step.

   Seismogram canvas height: 150px.
   Draw three stacked traces, each 50px tall:
     Station 1: y-centre 25px, colour #00ff00
     Station 2: y-centre 75px, colour #00ffff
     Station 3: y-centre 125px, colour #ff8800
   Dim horizontal dividing lines between traces.

   Three SONIFY buttons below controls:
   [SONIFY S1] [SONIFY S2] [SONIFY S3]
   Each feeds its station's seismogram to GranularEngine.

   The contrast between Station 1 (basin — high amplitude,
   delayed) and Station 3 (bedrock — lower amplitude, earlier
   arrival) is the geophysical point of TREMORS. Basin
   amplification as described in Louie's UNR lecture
   (https://youtu.be/-ZrIK3cfjFk?si=SxVMSyPqxEG7zaRh) and
   quantified in SW4 Nevada basin hazard models
   (https://sites.google.com/view/nevada-shake-zoning).

   Draw three station markers on main canvas at start of
   renderFrame() — each the colour of its trace.

5. LINK BACK
   Top-right corner: small dim link
   '← SOS' href='http://sos.allshookup.org'

6. FINAL CHECKS
   - Remove all console.log except errors
   - Confirm reset() fully zeroes all three station seismograms,
     all pressure arrays, stepCount, all seismoIdx values
   - Audio context created only on user gesture
   - File runs correctly from file:// protocol (no server needed)
   - Target file size: under 30KB unminified
   - Validate: open in Chrome and Firefox, confirm no errors

Output final tremors.html ready for SFTP upload to
allshookup.org. Deployment path: /public_html/tremors.html
```

---

## Phase 8 — Streaming bridge (deferred)

**Status:** Not built until Phases 1–7 are complete, deployed, and tested on M4 Mini hardware.

**What it requires:**
- A ring buffer feeding GranularEngine grain-by-grain in real time as the simulation steps forward — sonification happens simultaneously with the visual, not after
- Careful timing alignment between simulation DT and Web Audio grain scheduling to prevent clicks and dropouts
- Possible Web Worker offload of the stencil loop to free the main thread for audio scheduling
- Assessment of whether JS single-thread budget is sufficient at 400×300 or whether grid reduction is needed

**Context prompt:** To be written after `tremors.html` is live at allshookup.org and performance characteristics on M4 Mini are known.

---

## Deployment checklist

```
[ ] tremors_p1.html — console verification pass
[ ] tremors_p2.html — seismogram non-zero arrival confirmed
[ ] tremors_p3.html — wavefield visible, basin slowing visible
[ ] tremors_p4.html — all 4 presets working, reset clean
[ ] tremors_p5.html — seismogram trace live, P arrival marked
[ ] tremors_p6.html — audio output working, grain controls live
[ ] tremors.html    — three stations, geology layer, identity
[ ] SFTP upload to allshookup.org /public_html/tremors.html
[ ] Link from sos.allshookup.org instrument index
[ ] Test file:// protocol (no server dependency)
[ ] Test Chrome + Firefox
[ ] Phase 8 context prompt — written post-deployment
```

---

## Key architectural decisions locked

| Decision | Choice | Rationale |
|---|---|---|
| Language | Vanilla JS | KISS, consistent with all SOS instruments |
| Dependencies | Zero | Consistent with ONE.html, flow.html, seis.html, ShadowZone.html |
| Stencil order | 2nd order (4-neighbour Laplacian) | Sufficient accuracy, minimal complexity |
| Boundary condition | Sponge damping layer | Simpler than PML, invisible to ear downstream |
| Floating point | Float32Array throughout | Native typed array, JIT-friendly |
| Audio | Web Audio API, GranularEngine | Same class architecture as SeisClaw |
| Bridge mode | Batch first, streaming later | Reduces timing complexity in Phase 6 |
| Wasm | Not used | JS sufficient at 400×300; no build tooling needed |
| Grid size | 400×300 | Real-time in JS per jtiscione benchmark |
| Canvas API | 2D only, ImageData | No WebGL dependency |
| Sample rate | Math.round(1.0 / DT) | Auto-transposes seismic → audible frequencies |

---

*TREMORS — MIT - allshookup.org 2026*
*Physics lineage: SW4 (https://github.com/geodynamics/sw4) / CIG (https://geodynamics.org)*
*Reference browser implementation: https://github.com/jtiscione/webassembly-wave*
*EarthScope data: https://service.earthscope.org*
