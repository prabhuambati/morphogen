# Morphogen — Interview Preparation Guide

Morphogen is a client-side image-evolution playground. It maps an image onto a grid, evolves the grid with a structure-preserving, stochastic variation of Conway's Game of Life, and then reconstructs the image from the evolved cell patches. It also includes an image-composition workspace, a cell inspector, PNG/video export, and optional synthesized audio.

This guide describes the code currently in [`index.html`](./index.html). It is designed to help you explain the project in an interview and answer likely follow-up questions. It is not a claim that every possible interview question has a single project-specific answer.

## The 30-second explanation

> Morphogen is a browser-only generative art application built with HTML, CSS, and vanilla JavaScript. It divides an image into a grid of cells, analyzes local color differences to identify important edges, and locks a configurable subset of those cells as a permanent structural skeleton. The remaining cells evolve using Conway-style neighbor rules with a probability of survival based on image importance. Each live cell carries the identity of an actual source-image patch, so births inherit visual material from their parent. When evolution ends—or when the user requests it—the app animates a reconstruction using either those evolved patches or the original patches. Canvas renders the result; browser APIs provide image loading, audio, and video capture.

## What the application does

- **Simulate:** Load an image or use the generated sample. Choose grid size, structure preservation, survival strength, and visual settings; then play, pause, step, reset, or replay the evolution.
- **Inspect:** Hover over a cell or click to pin it. See its state, neighbors, lock status, current and original patch colors, importance, fade, and age.
- **Reconstruct:** Rebuild from the current or completed state using a memory-based or original-patch fill, with outside-in, center-out, random, or scanline ordering.
- **Create a composition:** In Art mode, arrange image and skeleton layers, transform them, undo/redo, and export. Fixate a composition, select a grid, generate its structural mask, and send it into simulation.
- **Export:** Save a PNG of the current view or record the simulation to a browser-supported video format. Video uses a dedicated offscreen canvas so the UI is not included.
- **Hear the simulation:** Optionally synthesize generation and reconstruction sounds with the Web Audio API. Audio is off until the user enables it.

## How to run it

There is no application build step or framework runtime. The application is implemented in one HTML file.

1. Install a recent browser (Chrome, Firefox, or Safari) with Canvas support.
2. From the project directory, start a local static server:

   ```sh
   python3 -m http.server 8000
   ```

3. Open `http://localhost:8000` and use **Upload PNG / JPG / WEBP**, or start with the generated sample.

Opening the file directly may work for basic use, but a local HTTP server is the more reliable way to run browser APIs. Audio requires a user gesture because of browser autoplay policies. Video recording depends on the browser's `MediaRecorder` and canvas `captureStream()` support.

`package.json` lists Playwright as a dependency, but does not define a build, start, or real test command. Its current `npm test` script is a placeholder that exits with an error. Do not describe the project as having an automated test suite unless one is added.

## Technology and architecture

| Area | Implementation |
|---|---|
| UI and layout | Semantic HTML, CSS, responsive layout, and DOM event handlers |
| Simulation and application state | Vanilla JavaScript in the inline script in `index.html` |
| Main visual output | HTML Canvas 2D context |
| Image analysis | Offscreen canvas, `getImageData()`, typed arrays |
| State storage | In-memory objects and typed arrays; no application server or database |
| Audio | Web Audio API (`AudioContext`, oscillators, filters, gain nodes) |
| Video | `MediaRecorder` with `HTMLCanvasElement.captureStream()` |
| Analytics | Optional deferred Vercel Web Analytics script loaded by the page |

The central simulation configuration lives in `S`; Art mode state is held in `Art`. The program is organized into inline-script sections for audio, image processing, build/reset, simulation, lifecycle, reconstruction, rendering, inspection, pointer interaction, file intake, video recording, export, and the animation loop.

### Data flow

```text
Uploaded image / generated composition
        ↓
Source image scaled to at most 1024 px on its longest side
        ↓
Per-grid-cell image analysis and source-region rectangles
        ↓
Edge and importance maps → structural lock mask
        ↓
Generation-0 cell buffers
        ↓
Cellular-automaton generations (double-buffered)
        ↓
Canvas rendering and cell inspection
        ↓
Reconstruction animation → PNG/video export
```

In the default simulation, columns follow the image aspect ratio. The grid has at most 128 rows, at most 512 columns, and at most 262,144 cells. The UI offers row presets of 16, 32, 64, 96, and 128.

## Core simulation, step by step

### 1. Prepare the image and grid

`setSource()` keeps the source image in memory, scales a working copy down if its longest side exceeds 1024 px, makes a thumbnail, and rebuilds the simulation. The app creates an image region for each grid cell. Those regions map back to actual rectangles in the working source image.

### 2. Analyze structure and importance

`sampleAverages()` calculates one representative color per cell. It converts sRGB channel values to linear light before averaging, then converts the result back to sRGB. For large regions it samples fewer pixels to bound the work. It also finds a distinctive pixel in each region and can blend toward it according to the clarity setting.

The averages are **analysis metadata**, not the final cell artwork. The renderer draws each cell's source-image patch. Neighboring color differences produce an edge map and an importance map, normalized against a high percentile so an isolated extreme does not dictate the entire scale.

`applyLockMask()` ranks cells by edge strength and locks the top target count. The target count is approximately:

```text
round(numberOfCells × structurePercentage^1.6)
```

The exponent means the percentage slider is not a direct one-to-one mapping to the fraction of cells locked. A locked cell remains alive and keeps its own original image-region identity.

### 3. Evolve the cells

Each cell stores whether it is alive, its source-region ID, its fade value, and its age. The app uses typed arrays and two buffers: each step reads `cur`, writes `nxt`, and swaps them at the end. This prevents a cell updated early in a generation from affecting a neighbor later in that same generation.

The neighborhood is the eight surrounding cells. For unlocked cells:

- A live cell survives normally with two or three live neighbors.
- If it would otherwise die, it gets a probabilistic reprieve:

  ```text
  min(0.95, survivalStrength × importance^1.25)
  ```

- A dead cell is born with exactly three live neighbors.
- A new cell inherits the source-region ID of a randomly chosen live parent.

Locked cells are forced alive and reset to their own original region. The simulation stops on extinction, a stable alive/dead configuration, or a repeated non-locked occupancy state. The repeated-state check uses an FNV-1a hash to find candidates and a byte-for-byte comparison to avoid treating hash collisions as repeats.

### 4. Reconstruct the image

Reconstruction animates cells in one of four orders: outside-in from the structural skeleton (or the border if no cell is locked), center-out, random, or row-major scanline.

- **Memory fill:** surviving cells keep their evolved patch. Empty cells choose a patch from an already-restored neighbor where possible; otherwise they use their fading patch or their own original patch.
- **Original fill:** each cell transitions back to its own Generation-0 patch.

The renderer cross-fades the source-region patches as the reconstruction wave advances. In memory mode, changed patches are marked as mutants and can shimmer after restoration.

## Rendering and performance choices

- The main simulation is drawn into a canvas rather than creating a DOM element for every cell.
- Typed arrays store compact per-cell data; double buffering makes generation updates predictable.
- Rendering clips much of the cell loop to the visible viewport when the canvas is zoomed or panned.
- An optional smooth path draws the grid into a one-pixel-per-cell offscreen canvas and scales it with interpolation. This changes rendering, not the underlying simulation.
- The main `requestAnimationFrame` loop uses elapsed time to advance generations at the selected rate, caps elapsed time and work per frame, and redraws when needed instead of always drawing an unchanged board.
- Image analysis is capped to a working-image size, and large regions are sampled with a stride.
- Video is rendered to a separate offscreen canvas at the image/grid aspect ratio, capped at 1920 px on its longest side. It records the image rather than the toolbars and other page UI.

The simulation can still be computationally expensive near its maximum grid size: each generation checks up to eight neighbors for every cell, and some image/paint operations also visit many cells. Smooth rendering and viewport clipping do not make the simulation algorithm itself sublinear.

## Interview questions and suggested answers

Use the answers as a base, then explain them in your own words. Be ready to point to the corresponding implementation in [`index.html`](./index.html).

### Product and architecture

**1. What problem does the project solve?**<br>
It turns an ordinary image into an interactive generative-art experience. The image's important contours remain recognizable while its interior evolves, and reconstruction can show either the original image or a visual trace of the evolution.

**2. What is the main technical challenge?**<br>
Balancing image recognizability with visible stochastic change. The lock mask preserves a structural skeleton; importance-weighted survival and inherited image patches let the rest change without reducing the result to unrelated colored cells.

**3. Why build it with vanilla JavaScript?**<br>
The current app is a self-contained browser experience with no application server or framework build. Vanilla JS keeps the runtime and deployment simple and gives direct access to Canvas and browser media APIs. A framework could help if the UI grew more component-heavy, but it is not required for this app's present structure.

**4. What is the architecture?**<br>
It is a single-page, client-side app. HTML defines the UI, CSS styles it, and one inline JavaScript program owns state, image analysis, simulation, rendering, and browser integrations. Modules are separated by responsibility using code sections and closures, rather than separate bundled modules.

**5. Does the image get uploaded to a server?**<br>
The application code decodes and processes the selected image locally in the browser; it has no image-upload backend. The page does load a deferred Vercel analytics script, so it would be inaccurate to claim the page makes no external network requests.

**6. What happens when the user changes a source image or structural setting?**<br>
The app rebuilds analysis data, the lock mask, and generation zero. This starts a fresh simulation because the old cell buffers no longer correspond to the new image or structure.

### Algorithms and data structures

**7. Is this standard Conway's Game of Life?**<br>
It uses Conway's eight-neighbor birth rule and the usual two/three-neighbor survival rule, but adds permanently locked cells and probabilistic survival for cells that would otherwise die. So it is a Life-inspired stochastic cellular automaton, not standard deterministic Life.

**8. How is image structure preserved?**<br>
The app compares representative colors in neighboring image regions, normalizes edge strengths, ranks cells, and locks a configurable number of high-edge cells. Locked cells are always alive and display their original patch.

**9. What does the importance score do?**<br>
It contributes to the chance that an unlocked live cell survives outside the standard rule. The score derives from local color contrast and is scaled by edge sensitivity. It also informs the Art mode's visual hierarchy.

**10. Why average in linear light?**<br>
RGB values in common image formats are gamma-encoded. Averaging those encoded channel values directly can produce perceptually incorrect results. Converting to linear light, averaging, and converting back better reflects the amount of light represented by the samples.

**11. Why keep a region ID instead of only a color?**<br>
A region ID points back to the actual rectangle in the source image. Rendering can therefore show image patches, not just a flat average color. A newborn inherits a parent's region ID, which carries visual lineage through the simulation.

**12. Why use typed arrays?**<br>
The grid has many cells and stores several numeric values per cell. Typed arrays provide compact, predictable numeric storage and work well for indexed loops and bulk buffer operations.

**13. Why double-buffer each generation?**<br>
The rules should evaluate one consistent previous generation. Reading only `cur` and writing only `nxt` prevents update order from changing the result within a generation; swapping buffers commits the generation atomically.

**14. How are loops detected?**<br>
The app packs the alive/dead state of non-locked cells into bytes, hashes it with FNV-1a, and stores recent states. A matching hash is confirmed by comparing all bytes. It stores a bounded history to avoid unbounded memory growth.

**15. Is the simulation deterministic?**<br>
No. Survival exceptions, parent selection, and reconstruction choices use `Math.random()`, and no seed is exposed for replaying an identical run. The same image and settings can produce different evolutions.

**16. Is repeated occupancy a mathematical proof of a future loop?**<br>
Not for this stochastic variant. The app treats a repeated occupancy state as a loop and stops, but probabilistic survival can lead to different future transitions from the same state. A deterministic seed or deterministic update policy would be needed for strict cycle reasoning.

**17. How does memory reconstruction work?**<br>
Survivors keep their patch lineage. Empty cells prefer the patch of a restored neighbor; if that is unavailable they fall back to their own fading patch or their original patch. This lets the final image reflect some of the path taken through the simulation.

### Browser APIs and user experience

**18. Why use Canvas?**<br>
Canvas is a natural fit for drawing a large grid, image patches, overlays, and animation into one surface. It avoids a separate DOM node per cell and supports PNG and stream capture APIs.

**19. How does the animation loop work?**<br>
`requestAnimationFrame` supplies timestamps. The loop accumulates elapsed time and steps the simulation at the configured generations-per-second rate, with a cap to avoid an excessive catch-up loop after a delayed frame. Reconstruction and visual effects advance using elapsed time too.

**20. How does image upload work?**<br>
The file input accepts PNG, JPEG, and WebP MIME types. A temporary object URL is decoded into an `Image`, then revoked on load/error; the decoded image is drawn into an offscreen canvas for analysis. The file is not sent to an application backend.

**21. How is video recorded without recording the whole UI?**<br>
The recorder creates a dedicated offscreen canvas, draws the simulation into it, captures that canvas as a media stream, and passes the stream to `MediaRecorder`. It can add the existing Web Audio output stream if audio is enabled and available.

**22. Which recording format is guaranteed?**<br>
None universally. The app checks browser codec support and tries WebM VP9, WebM VP8, WebM, then MP4 variants. Actual support depends on the browser and operating system; the UI reports the selected MIME type.

**23. Why does audio require a click?**<br>
Browsers commonly suspend or block audio until a user gesture. The app initializes or wakes its `AudioContext` from user interaction and keeps sound optional.

**24. What is the purpose of the cell inspector?**<br>
It makes the model observable. A user can connect a visual cell to its row/column, neighbor count, lock status, region colors, importance, fade amount, and age—useful both for debugging and explaining the simulation.

**25. What does Art mode add?**<br>
It allows the user to build a composition from independent image/skeleton layers, transform and order them, then fixate the result into a raster image. The user selects a grid, generates the structural mask, locks the grid, and can simulate that composition.

### Trade-offs, limitations, and next steps

**26. What are the main limitations?**<br>
The app is a single HTML file; there is no formal test suite or build pipeline; randomness is not seedable; maximum grids can be expensive; and media recording/codec support varies by browser. State is in memory, so a reload loses the current composition and simulation.

**27. How would you test it?**<br>
I would first extract pure functions for cell stepping, edge/importance calculation, lock selection, and reconstruction ordering. Then add deterministic tests with an injected random source, including tiny known grids, boundaries, locked-cell invariants, buffer swaps, and stop conditions. Browser-level tests could cover file loading, controls, drawing, and recorder capability fallbacks.

**28. How would you improve determinism?**<br>
Inject a seeded pseudorandom number generator and store its seed with the run settings. Use it for every random decision. That would make runs reproducible, improve debugging, and make algorithm tests stable.

**29. How would you improve performance at larger grids?**<br>
Profile first. Potential follow-ups include optimizing neighbor counting, avoiding repeated allocations, throttling inspector updates, moving simulation work to a Web Worker, and considering WebGL/WebGPU for rendering. Worker communication and rendering complexity should be justified by measurements.

**30. How would you make the code easier to maintain?**<br>
Split the inline script into modules by responsibility, add typed documentation or TypeScript if the project needs stronger contracts, centralize DOM/state wiring, and create unit and browser tests. Keep the current single-file deployment as an explicit constraint if simplicity is important.

**31. How would you handle accessibility?**<br>
Preserve semantic controls and labels, add clear keyboard focus states and accessible descriptions for the canvas, ensure all important actions are keyboard-operable, announce phase/progress changes appropriately, and avoid using color alone to communicate cell state.

**32. What security/privacy considerations matter?**<br>
Image decoding and analysis are local in the app, which avoids sending user images to an application backend. Still, the page loads third-party analytics, and the app should be served over HTTPS in production. Uploaded file MIME checks improve UX but are not a security boundary; robust handling should also consider file size, decode errors, and browser resource limits.

**33. What is not implemented yet?**<br>
There is no persistence, shareable/reproducible seeded run, server API, real automated test suite, or production build pipeline in the current repository. These would be future work, not existing features.

## Suggested live demo

1. Start with the generated sample image and show the source image and grid.
2. Point out the structural lock percentage and explain that it preserves high-contrast contours.
3. Start Morph, pause after a few generations, and inspect one locked cell and one evolving cell.
4. Reconstruct in **MEMORY** mode and explain patch inheritance; optionally compare with **ORIGINAL** mode.
5. Show the Gen 0 comparison view and export a PNG.
6. If time permits, show Art mode's composition → fixate → generate grid → lock → simulate flow.
7. Mention video/audio as browser-dependent optional features; do not make the interview depend on them working on an unfamiliar machine.

## Useful project facts to memorize

- **Default grid:** 128 × 128 before aspect-ratio adjustment.
- **Maximum grid:** 262,144 cells; at most 512 columns.
- **Source working-image limit:** 1024 px on the longest side.
- **Grid neighborhoods:** eight adjacent cells.
- **Birth rule:** exactly three live neighbors.
- **Ordinary survival:** two or three live neighbors.
- **Probabilistic survival exception:** strength × importance^1.25, capped at 0.95.
- **Structural locks:** ranked from image-derived edge strength; always alive and retain their original patch.
- **Rendering:** Canvas 2D; optional offscreen smoothing.
- **Persistence/backend:** none for app state or image processing.
- **Tests/build:** no real test or build scripts currently configured.

## How to talk about your contribution

Be precise about what you personally designed or implemented. A strong answer describes one concrete decision, its reason, and its trade-off. For example:

> “I used double-buffered typed arrays so a generation reads a consistent snapshot, and I kept source-region IDs on cells so the visualization could evolve actual image patches rather than just colors. The trade-off is that patch lineage and reconstruction require additional per-cell state and bookkeeping.”

Replace “I” with “we” if this was collaborative, and do not claim features or testing that you did not personally verify.
