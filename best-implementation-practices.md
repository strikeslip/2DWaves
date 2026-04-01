
## Best Claude practices for implementing TREMORS

### 1. One phase per conversation — no exceptions

Never try to combine phases in a single session. "Can you do Phase 1 and 2 together?" feels efficient but produces worse output. Each phase has a single clear goal and a single verifiable output. Opus does its best work with tight, unambiguous scope. Two phases in one session means two opportunities for scope creep, naming drift, and untestable intermediate states.

---

### 2. Verify before advancing

Before starting Phase N+1, the Phase N file must pass its specific verification test in a real browser — not just "looks reasonable in the response." Concretely:

- Phase 1: open browser console, confirm the logged Vp and sponge values match expected numbers
- Phase 2: confirm seismogram array has a clear non-zero peak at a physically plausible step number
- Phase 3: watch the wavefield — confirm waves slow visibly in the basin region
- Phase 4: change each preset, confirm VP_MAX and DT update correctly in the HUD
- Phase 5: confirm the P-wave arrival marker appears on the seismogram trace
- Phase 6: confirm audio output is audible and grain controls affect the sound

If a phase fails verification, fix it in the same session before pasting it forward. Pasting a broken file into the next phase prompt compounds the error.

---

### 3. Paste the entire file — never summarise

Already covered in the plan but worth restating as a practice point: when you paste the previous phase into the next prompt, paste it **raw and complete** including the `<!DOCTYPE html>` declaration, all style tags, all script tags, every line. Do not clean it up, reformat it, or remove comments. Opus needs to see exactly what exists, not an idealised description of it.

---

### 4. Name the constraint at the top of every prompt — every time

The single most important line in every TREMORS prompt is:

```
Vanilla JS only. Zero external dependencies. No imports, no npm, no CDN links.
```

Opus has a strong prior toward reaching for libraries, modules, and modern JS patterns. Without this constraint stated explicitly at the top of every prompt, by Phase 4 or 5 you will start seeing `import` statements, npm suggestions, or subtle ES6 module syntax creeping in. State the constraint every time regardless of how obvious it seems.

---

### 5. State what NOT to do as clearly as what to do

Opus responds well to explicit negative constraints. In every phase prompt include a short "do not" list relevant to that phase. Examples:

- Phase 3: "Do not use WebGL. Canvas 2D API only."
- Phase 6: "Do not modify the wave simulation code from Phase 5."
- Phase 7: "Do not add any new simulation features. Polish only."

Without these, Opus will occasionally "improve" things that were working correctly — refactoring the stencil, renaming variables, adding helper abstractions that break the flat array architecture.

---

### 6. Specify the verification test inside the prompt itself

Tell Opus exactly what the output should produce when run. For example in Phase 2:

```
After the headless run, log max absolute value in seismogram[0..499]
and the step at which it occurs. A peak around step 80-120 confirms
correct propagation speed for a 4000 m/s bedrock model at these
grid dimensions.
```

This does two things: it gives Opus a target to aim for during code generation (it will self-check its own logic), and it gives you a clear pass/fail criterion when you run it.

---

### 7. Lock variable names early and enforce them

In Phase 1 you establish the naming convention: `P_cur`, `P_prv`, `P_nxt`, `getIdx()`, `velocity`, `sponge`, `seismogram`, `seismoIdx`, `stationX`, `stationY`. Write these into Phase 1's prompt explicitly and they become the canonical names for the entire project. Every subsequent prompt should refer to these exact names.

If Opus ever renames a variable — say `P_cur` becomes `currentPressure` — catch it during verification and correct it before pasting forward. Name drift across phases is a silent source of bugs that only appear in later phases when two naming conventions collide.

---

### 8. The getIdx() indexing check is non-negotiable

The single most common bug in 2D FDTD implementations is mixing x and y in the flat index function. Put this comment directly in the stencil prompt and again in every subsequent phase that touches the stencil:

```
getIdx(x, y) = y*WIDTH + x
x is the column (horizontal), y is the row (vertical).
Laplacian neighbours: (x+1,y), (x-1,y), (x,y+1), (x,y-1)
— never (x,y+1) written as getIdx(y+1, x).
```

Belabour this point. It costs nothing to repeat it and saves a debugging session.

---

### 9. Keep Phase 6 (GranularEngine) as a clean boundary

The GranularEngine is architecturally the most complex component and the one most likely to attract scope creep. The prompt already states "bridge is one-directional, engine never writes back to simulation arrays." Enforce this by also stating:

```
The GranularEngine class must have no knowledge of the simulation.
It receives a Float32Array and a sampleRate. That is its entire
interface. It does not read velocity[], seismogram[], or any
simulation variable directly.
```

This keeps the two systems testable independently and matches the contract that GranularEngine already has in SeisClaw.

---

### 10. Audio context creation — always on user gesture

State this explicitly in Phase 6 and again in Phase 7:

```
initAudio() must only be called from within a click or keydown
event handler. Never call it automatically on page load or from
within animate(). Chrome and Safari will silently suspend the
AudioContext if created before a user gesture and TREMORS will
appear to produce no sound.
```

This is a gotcha that catches experienced developers. Opus knows about it but will occasionally generate auto-init code if not reminded.

---

### 11. Test on file:// protocol at every phase

All SOS instruments at allshookup.org are served as static files. No server, no localhost. Open each `tremors_pN.html` directly from the filesystem — `file:///path/to/tremors_p1.html` — not via a dev server. Some browser APIs behave differently under `file://` and catching this early is far cheaper than discovering it at deployment.

---

### 12. Phase 7 is polish only — say it three times

By Phase 7 the instrument is functionally complete. The risk at this phase is Opus deciding to "improve" the physics, refactor the stencil, or restructure the audio engine. The Phase 7 prompt already says "polish only" but reinforce this verbally when you open the session:

```
This is a finalisation pass only. Do not change any simulation
logic, stencil code, or GranularEngine internals. Changes are
limited to: visual identity, geology background layer, three
stations, keyboard shortcuts, and the link back to SOS.
```

---

### 13. The physical output is the artistic point — preserve it

Throughout implementation, the geological contrast between Station 1 (basin — high amplitude, late arrival) and Station 3 (bedrock — lower amplitude, early arrival) is not just a technical feature. It is the reason TREMORS exists as an instrument. When making any tradeoff in Phase 7 — between visual polish and performance, between three stations and one, between complexity and KISS — always favour the decision that makes that contrast most audible and visible. That is the instrument's identity.

---

### Summary table

| Practice | Why it matters |
|---|---|
| One phase per conversation | Tight scope = better output |
| Verify before advancing | Broken files compound errors |
| Paste entire file verbatim | Opus has no memory between sessions |
| State vanilla JS constraint every time | Opus defaults to modern tooling |
| Explicit "do not" lists | Prevents "improvements" that break things |
| Verification targets in prompts | Opus self-checks during generation |
| Lock variable names in Phase 1 | Prevents naming drift across phases |
| Belabour getIdx() indexing | Most common FDTD bug |
| GranularEngine as clean interface | Testable independently of simulation |
| Audio only on user gesture | Chrome/Safari autoplay policy |
| Test file:// at every phase | SOS deployment is static files |
| Phase 7 = polish only | Prevent late-stage refactoring |
| Preserve the geological contrast | That is what TREMORS is for |
