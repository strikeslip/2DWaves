# Seismic Waveform Visualisation
## Thread Summary — SeisClaw / TREMORS Research & Implementation

---

A transcript of John Louie's UNR seismic principles lecture (https://youtu.be/-ZrIK3cfjFk?si=SxVMSyPqxEG7zaRh) asking for key points and identification of the visualisation software. The lecture covered: earthquake wave propagation through Nevada basin structures, P-wave vs S-wave behaviour, basin trapping of S-waves, reflection/refraction physics (Snell's Law, Fermat's Principle, acoustic impedance), and rock velocity as a function of consolidation, saturation, and porosity. Additional references supplied with the lecture: Louie's Nevada shake zoning work (https://sites.google.com/view/nevada-shake-zoning) and a related drive document (https://drive.google.com/file/d/1LAPoZ4qdB5vqWlFQEeEPR-Fe_QONtA0d/view). The visualisation software was identified as most likely SW4 or its predecessor WPP from LLNL, post-processed through GMT or ParaView — the same finite difference wave propagation codes used by USGS for hazard calculations. A cross-section diagram was generated showing the basin wave trapping mechanism central to SeisClaw's entire conceptual territory.

---

## CIG / geodynamics.org

Both the main site (https://geodynamics.org/) and GitHub org (https://github.com/geodynamics) were studied. CIG is NSF-funded open-source computational geoscience infrastructure — 39 repos, predominantly C++/Fortran, MPI-parallel. Key repos identified:

- **SW4** (https://github.com/geodynamics/sw4) — 4th order seismic wave propagation, LLNL, most directly relevant. GPL v2+, v3.0 current, 157 stars, C++/Fortran, MPI cluster-scale. DOI: https://doi.org/10.5281/zenodo.8322590. Theory background at https://computing.llnl.gov/projects/serpentine-wave-propagation/theory
- **seismic_cpml** (https://github.com/geodynamics/seismic_cpml) — lean Fortran FDTD, 16 programs, cleaner for reading and understanding wave mechanics
- **ASPECT** (https://github.com/geodynamics/aspect) — mantle convection, most active repo in the org (commits as of March 2026)
- **PyLith** (https://github.com/geodynamics/pylith) — crustal deformation and earthquake fault mechanics
- **BurnMan** (https://github.com/geodynamics/burnman) — Python mineral thermodynamics, relevant to ree.codes territory

---

## The portability question

Asked whether SW4's seismic wave propagation physics (https://github.com/geodynamics/sw4?tab=readme-ov-file) could be ported to JavaScript and run in a browser. Research drew on the LLNL Serpentine project theory page (https://computing.llnl.gov/projects/serpentine-wave-propagation/software), OpenSWPC documentation (https://earth-planets-space.springeropen.com/articles/10.1186/s40623-017-0687-2), and a working WebAssembly scalar wave equation browser demo (https://jtiscione.github.io/webassembly-wave/index.html).

Conclusion: SW4 itself cannot port — it depends on MPI distributed memory parallelism, curvilinear mesh generation, and cluster-scale compute. However the underlying physics — the acoustic/elastic finite difference time domain (FDTD) stencil — is simple enough to run in vanilla JS in a browser at 2D scale. The scalar acoustic wave equation is four subtractions and a multiply per cell per timestep. At 400×300 grid size this is real-time on modern hardware. The output — a synthetic seismogram recorded at a virtual station — is format-identical to real EarthScope MiniSEED waveforms (https://service.earthscope.org). This opens the path to feeding synthetic physically-valid waveforms directly into SeisClaw's GranularEngine. Related open implementations for reference: FD_ACOUSTIC collection in Python/Matlab (https://github.com/florianwittkamp/FD_ACOUSTIC) and OpenSWPC (https://github.com/OpenSWPC/OpenSWPC).

---

## ELI5

The rock-in-a-pond analogy: ripples = seismic waves, muddy patch = sedimentary basin (waves slow, amplify, get trapped), SW4 = industrial full-3D simulation requiring supercomputers, the underlying math = one simple equation applied repeatedly to a grid, browser 2D version = fast enough to run live, output = waveform pipe-compatible with GranularEngine. Same physics, toy scale, right scale for a live instrument.

---

## The implementation plan

Two modules, seven build phases.

**Module A — 2D FDTD engine.** Define grid (cells, physical dimensions, spacing). Paint velocity model (fast bedrock, slow basin). Calculate CFL stability condition to lock timestep. Initialise pressure field arrays (current, previous, next). Define Ricker wavelet source — the standard seismic pulse used in SW4 and all FDTD codes. Apply finite difference stencil each timestep (2nd order, four neighbours — the Madariaga/Levander stencil lineage documented in https://earth-planets-space.springeropen.com/articles/10.1186/s40623-017-0687-2). Sponge absorbing boundaries at edges. Render wavefield to Canvas via ImageData each frame. Record synthetic seismogram at virtual station positions.

**Module B — GranularEngine bridge.** Characterise GranularEngine's existing input contract (Float32Array, amplitude range, sample rate) as implemented in SeisClaw v4 at https://seisclaw.com / https://sos.allshookup.org. Normalise FDTD pressure values to −1.0/+1.0. Resample to match expected sample rate via linear interpolation if needed. Batch mode first (full simulation → hand off complete array). Streaming mode second (ring buffer feeding GranularEngine in real time as simulation runs).

**Build order:** data structures → physics engine (verify headless) → Canvas render (verify visually) → seismogram output (verify waveform) → GranularEngine bridge batch mode (verify end-to-end sonification) → presets and UI → streaming mode.

**Hard constraints throughout:** vanilla JS, zero dependencies, GranularEngine untouchable, 2nd order stencil, sponge boundaries, batch before streaming — consistent with the architecture principles established across all SOS instruments at https://sos.allshookup.org.

---

## The artistic/strategic upshot

This instrument — a browser 2D basin simulator whose synthetic seismograms feed SeisClaw's granular engine — closes the loop between the geophysical theory in Louie's lecture and SeisClaw's live sonification practice. It gives compositional control over the seismic input (design your own basin, place your own earthquake) while preserving physical plausibility. The real-data provenance differentiator remains intact because the same validated wave physics underlying SW4 (https://github.com/geodynamics/sw4) and the broader CIG software ecosystem (https://geodynamics.org) underlies both the synthetic simulator and the EarthScope stream (https://service.earthscope.org) it complements. It is a natural next SOS instrument alongside the existing suite at https://sos.allshookup.org.
