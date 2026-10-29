# Spatial upscaling, dynamic resolution, and AI frame generation/upscaling review

Status: **research review complete (AI candidates stopped at Stage 1/2); implementation specification and handoff ready for `@foss-earth/renderer` (P8, P9)**.
Date: **2026-10-27**.

---

## 1. State and viability of realtime AI frame generation and upscaling

### 1.1 Executive verdict

Realtime neural frame generation (`Practical-RIFE 4.25.lite`) and neural super-resolution (`RT4KSR x2`) were evaluated in `~/gh/foss-earth/research` ([`benchmarks/ai-framegen-upscale/REPORT.md`](../../../foss-earth/research/benchmarks/ai-framegen-upscale/REPORT.md), [`tables.md`](../../../foss-earth/research/benchmarks/ai-framegen-upscale/tables.md), [`docs/ai-framegen-upscale.md`](../../../foss-earth/research/docs/ai-framegen-upscale.md), and [`docs/ai-framegen-upscale-models.md`](../../../foss-earth/research/docs/ai-framegen-upscale-models.md)) on **2026-10-13** using the target Apple M5 Mac (16 GiB, Chrome 155.0.8059.40, Metal, ONNX Runtime Web 1.30.0 native WebGPU Execution Provider).

**Both neural candidates failed their Stage 1/2 gates and stopped.** Zero runtime profiles were admitted, and Stages 3–6 of [`foss-earth/research/docs/proposals/ai-framegen-upscale.md`](../../../foss-earth/research/docs/proposals/ai-framegen-upscale.md) were not started.

### 1.2 Summary of Stage 1 and Stage 2 empirical findings

#### Candidate 1: `Practical-RIFE 4.25.lite` (IFNet, 5,622,700 parameters) — **STOPPED**

- **Export parity (Gate A):** Requires opset 16–19 (`GridSample` in ORT's native WebGPU EP is registered only for opsets 16–19, NCHW) and input right/bottom zero-padding to multiples of 128 (`256×256`, `960×540 → 1024×640`, `1280×720 → 1280×768`, `1920×1080 → 1920×1152`). `fp32` passes Gate A at `small`, `540p`, and `720p` (max error $\le 8.69 \times 10^{-4}$), but **fails at `1080p`** across all three exporter variants (max error $1.78 \times 10^{-3} > 10^{-3}$). **`fp16` fails Gate A at all resolutions** (max error $0.388\text{–}0.679$), even when keeping `Resize`, `GridSample`, and grid arithmetic in `fp32`.
- **WebGPU numerical and 8-bit capture parity (Gates B and C):** **Fails catastrophically at every resolution and precision.** Although ORT places 100% of nodes on WebGPU (Gate D passes) and shares `GPUDevice` buffers without validation errors (Gate E passes), WebGPU output has max absolute errors of **$0.684\text{–}0.961$**, PSNR of **$17.17\text{–}22.81\text{ dB}$** (vs. the $50\text{ dB}$ floor), and only **$9.63\text{–}32.43\%$** of RGB bytes within $\pm 1$ 8-bit code (vs. the $99.5\%$ requirement; max code error reaches $245/255$). Running the exact same ONNX graph on ORT CPU reproduces the PyTorch reference (max error $\le 5.14 \times 10^{-4}$, $100\%$ of codes within $\pm 1$). Disabling ORT graph optimization (`320` vs. `336` native nodes) produces identical corrupted GPU values, isolating the defect to an upstream ORT native WebGPU kernel bug.
- **Accounted GPU memory (Gate F, $256\text{ MiB}$ cap):** Only `small` passes ($69.32\text{ MiB}$ `fp32` / $36.16\text{ MiB}$ `fp16`). All usable display sizes fail by wide margins:
  - `540p`: **$791.81\text{ MiB}$ (`fp32`) / $379.72\text{ MiB}$ (`fp16`)**
  - `720p`: **$780.58\text{ MiB}$ (`fp32`) / $429.07\text{ MiB}$ (`fp16`)**
  - `1080p`: **$2,040.08\text{ MiB}$ (`fp32`) / $993.47\text{ MiB}$ (`fp16`)**
- **CPU replay quality (Gate H):** On CPU reference outputs, `fp32` RIFE midpoints beat frame repeat and blend on continuous motion (`720p` 30→60: $29.96\text{ dB}$ vs. $15.92\text{ dB}$ repeat / $17.99\text{ dB}$ blend; 60→120: $32.40\text{ dB}$ vs. $17.92 / 19.87\text{ dB}$). Across scene cuts, RIFE produces double-exposure ghosting ($12.80\text{ dB}$ vs. $14.19\text{ dB}$ blend) and must be bypassed.
- **Warm inference latency:** In the separate cold/cache-warm start measurements ([`tables.md`](../../../foss-earth/research/benchmarks/ai-framegen-upscale/tables.md)), second-inference retirement took **~19–28 ms at $256^2$**, **~69–85 ms at `540p`**, **~97–117 ms at `720p`**, and **~215–275 ms at `1080p`**—far exceeding a $16.67\text{ ms}$ (60 Hz) or $33.33\text{ ms}$ (30 Hz) frame budget.

#### Candidate 2: `RT4KSR x2` (Cleanly reparameterized, 39,312 parameters, 60 ONNX nodes) — **STOPPED**

- **Export, WebGPU, and graph-capture parity (Gates A–E):**
  - **`fp32` passes Gates A–E at `small`, `720p`, and `1080p`** (max GPU error $\le 4.30 \times 10^{-6}$, PSNR $\ge 123.36\text{ dB}$, $100\%$ of RGB codes within $\pm 1$, max code error $1$), including with `enableGraphCapture: true`.
  - **`fp16` fails Gate C at all sizes** (only $73.03\text{–}79.00\%$ of codes within $\pm 1$ vs. the $98\%$ threshold) and fails Gate B mean error on `1080p` synthetic ($4.15 \times 10^{-3} > 4.0 \times 10^{-3}$).
- **Accounted GPU memory (Gate F, $256\text{ MiB}$ cap):** Passes at all sizes (`fp32` / `fp16`: `small` $4.95 / 4.10\text{ MiB}$, `720p` $66.25 / 57.91\text{ MiB}$, `1080p` $130.72 / 113.14\text{ MiB}$).
- **Full-frame replay quality (Gate H):** **Fails at both `720p` and `1080p`.**
  - Why it fails on real 3D renders: `RT4KSR` was trained on antialiased downsampled images. On bicubic-downsampled `720p` inputs, `RT4KSR` beats bicubic by $+1.24\text{ to }+3.03\text{ dB}$. However, rendering a 3D scene at half resolution produces **point-sampled pixel centers** with high-frequency aliasing. On directly rendered half-resolution frames, `RT4KSR` loses to plain bicubic interpolation on PSNR at `720p` (pooled **$28.028\text{ dB}$ vs. bicubic $28.376\text{ dB}$**; loses `cut`, `inertia`, `pan-fast`, `pan-slow`) and fails `pan-fast` at `1080p` ($29.521\text{ dB}$ vs. bicubic $29.667\text{ dB}$).
- **GPU cost screen (Gate G):** Formal Low Power Mode timing stopped after 120 s with 0 accepted rounds because background device idle utilization was 49–53% (above the 10% interference limit). Unqualified second-inference retirement wall times in [`tables.md`](../../../foss-earth/research/benchmarks/ai-framegen-upscale/tables.md) were **~9.1–10.9 ms at `720p` (`fp32`)** and **~21.7–26.0 ms at `1080p` (`fp32`)**.

### 1.3 Why neural frame generation and neural SR do not fit `0sfs`

1. **Main-thread CPU bottleneck vs. light GPU render pass:**
   - The October 2026 performance audit ([`foss-earth/docs/docs/performance-audit.md`](../../../foss-earth/docs/docs/performance-audit.md)) measured `/fly/` on the M5 at $1280 \times 800$ (DPR 1, WebGPU) at **58.8–59.8 fps** with **6.22 ms/frame of main-thread CPU** (36.6% of a core at 1×; 99.1% / 25.1 fps / 39.5 ms/frame under 4× CPU throttle), while the **GPU process used only 13% of a core** and the main globe pass cost **~3.0 ms/frame** on the M5 ([`optimization-backlog.md`](../../../foss-earth/docs/docs/optimization-backlog.md#L343-L345)).
   - Frame drops in `0sfs` (such as approach slowdowns near the ground or under CPU load) are driven by main-thread CPU work: imagery selection (`evaluateView`), `0sfs`'s second raster update (`C11`, ~59 ms/s), loaded-detail readout (`C12`, ~54 ms/s), 3D Tiles preparation, and terrain surface raycasts.
   - Adding a 9–26 ms neural upscale pass (`RT4KSR`) to save part of a ~3 ms GPU pass is a net loss in both frame rate and battery power.
2. **Closed-loop piloting latency:**
   - Midpoint interpolation `I(A, B)` requires frame `B` at $t = T$ before synthesizing $t = 0.5T$, adding at least one full source frame interval of lookahead buffer delay ($33.3\text{ ms}$ at 30 fps) plus inference time. During landing flare, rollout, or F-35B STOVL hover, 50–100+ ms of extra visual delay degrades pilot control loop stability.
3. **HUD, attitude indicator, and 3D cockpit screen desynchronization:**
   - `0sfs` draws its primary flight instruments in the DOM ([`fly/src/flight/hud/flightHud.ts`](../../fly/src/flight/hud/flightHud.ts)), its GPU attitude indicator directly on the shared `GPUDevice` ([`fly/src/flight/hud/attitudeWebGpu.ts`](../../fly/src/flight/hud/attitudeWebGpu.ts)), and mirrors DOM UI onto a 3D touchscreen inside the cockpit with an optional second chase-camera render ([`fly/src/flight/cockpit/cockpitScreen.ts`](../../fly/src/flight/cockpit/cockpitScreen.ts)). Delaying or warping the world image desynchronizes the artificial horizon from the real horizon and blurs cockpit avionics text.

---

## 2. High-viability path: Analytical Spatial Upscaling (P9) and Dynamic Resolution (P8)

In [`foss-earth/docs/docs/optimization-backlog.md`](../../../foss-earth/docs/docs/optimization-backlog.md#L113-L125) and [`0sfs/research/optimization-backlog.md`](../../research/optimization-backlog.md), the high-ROI rendering items are:

- **P1 / F1 (Resolution scale):** Already implemented via `renderer.resolutionScale`. Drawing 1.0–1.5 device pixels per CSS pixel cuts fragment shading by 45–75% on a 2× Retina screen while keeping DOM HUD and panels crisp.
- **P9 (Analytical spatial upscaling — FSR 1-style EASU + RCAS):** Reconstructs a lower-resolution 3D render (`resolutionScale < 1`) to native canvas device pixels using edge-adaptive spatial upsampling (EASU) and robust contrast-adaptive sharpening (RCAS). Runs in sub-millisecond GPU time, requires zero model downloads or ORT memory pools, and works on both WebGPU and WebGL2 (`B12`).
- **P8 (Dynamic resolution scaling):** Steers effective internal resolution scale from measured GPU frame time (`WebGPU` pass timestamps / `WebGL2` disjoint timer query) within user limits, quick to lower and slow to raise, holding a frame-time target without oscillation.

### 2.1 Existing code anchors in `@foss-earth/renderer`

All paths below are relative to `~/gh/foss-earth/renderer` (verified clean on `main` at commit `b19edd9`):

1. **`src/settings/catalogue.ts` (lines 25–43):**
   - Defines `renderer.resolutionScale` (`min: 0.25, max: 2, step: 0.25, scale: "log2", appliesLive: true`).
   - Defines `renderer.frameRateCap` (`min: 10, max: 240`, or `"off"`, `appliesLive: true`).
2. **`src/babylon/rendererLifecycle.ts` (lines 81–108, 136–152):**
   - `applyResolutionScale()` currently computes:
     ```ts
     const scale = settings.get("renderer.resolutionScale");
     const ratio = (platform.devicePixelRatio() || 1) * (typeof scale === "number" && scale > 0 ? scale : 1);
     renderer.engine.setHardwareScalingLevel(1 / ratio);
     ```
   - Watches `renderer.resolutionScale` and `platform.onResize`, calling `applyResolutionScale(); renderer.engine.resize(); scheduler.requestRender();`.
   - `renderFrame(prepare, afterRender)` brackets `renderer.engine.beginFrame()` → `prepare?.(); scene.render();` → `renderer.engine.endFrame()` → `afterRender?.()`.
3. **`src/babylon/frameProfiler.ts` (lines 12–135):**
   - `profileBabylonScene` attaches `webGpuPassTimer(engine)` on WebGPU (`"enableGPUTimingMeasurements" in engine`) or `webGlFrameTimer(engine)` on WebGL (`canTimeGpuFrames(engine)`).
   - **Important interaction to fix during P8:** `webGpuPassTimer` directly sets `engine.enableGPUTimingMeasurements = false` when `profiler.enabled` is false (`lines 89–92`), and consumes `engine.gpuTimeInFrameForMainPass?.counter` (`counter.count`, `counter.total`). If `P8` also reads GPU frame time when the profiler tab is off, `P8` and `profileBabylonScene` must share a reference-counted GPU frame-time reader on `engine` so disabling profiling does not shut off `P8`'s GPU timer (and vice versa).
4. **`src/vite/babylonBarrels.ts` and `src/shaderChunks.ts`:**
   - `babylonBarrels()` marks all `@babylonjs/core/**/index.js` files as `moduleSideEffects: false`.
   - Any Babylon `PostProcess` or rendering pipeline used in `@foss-earth/renderer` **must not** rely on barrel side effects to register shaders or scene components: import required Babylon modules directly from their concrete file paths and register both GLSL (`Effect.ShadersStore`) and WGSL (`ShaderStore.ShadersStoreWGSL`) shader sources explicitly.
5. **Pure entry vs. Babylon entry (`src/index.ts` vs. `src/babylon/index.ts`):**
   - `npm run test:consumer` compiles the pure API (`src/index.ts`) with `ES2022` and **no DOM library or Babylon/WebGPU types**. Keep pure controller logic and parameter specifications in `src/` free of DOM/Babylon imports, and place Babylon `PostProcess` / `Scene` / `AbstractEngine` wiring under `src/babylon/`.

---

### 2.2 Technical design for P9: Analytical Spatial Upscaling (EASU + RCAS)

#### Why `setHardwareScalingLevel` alone is soft
When `renderer.resolutionScale < 1` today, `engine.setHardwareScalingLevel(1 / (dpr * scale))` reduces the `<canvas>` drawing buffer dimensions, leaving the browser compositor to stretch the low-resolution canvas to CSS size with a basic bilinear filter.

#### How P9 works
1. **Settings (`src/settings/catalogue.ts`):**
   - Follow the project rule in [`foss-earth/docs/docs/proposals/implementation/performance-agent-state.md`](../../../foss-earth/docs/docs/proposals/implementation/performance-agent-state.md): *"A pixel-changing option goes behind a setting that defaults to the current behaviour, and onto the list for the user's decision."*
   - Add `renderer.upscaling`:
     - `kind: "choice"`, choices: `[{ id: "off", label: "Off (browser scaling)", shortLabel: "Off" }, { id: "spatial", label: "Spatial (EASU + RCAS)", shortLabel: "Spatial" }]`
     - `default: "off"`, `appliesLive: true`, `home: { tab: RENDERER_TAB, section: "main", level: "main" }`.
   - Add `renderer.upscaling.sharpness`:
     - `kind: "number"`, `unit: "ratio"`, `bounds: () => ({ min: 0, max: 1 })`, `step: 0.05`, `default: 0.5`, `appliesLive: true`, `home: { tab: RENDERER_TAB, section: "main", level: "advanced" }`.
   - Backend capability (`B12`): Supported on `webgpu` and `webgl2`. On `webgl` (WebGL1), keep `renderer.upscaling` bypassed and call `settings.setNote("renderer.upscaling", "Unavailable on WebGL1: spatial upscaling requires WebGPU or WebGL2.")`.
2. **Pass architecture (`src/babylon/spatialUpscaling.ts`):**
   - **When `renderer.upscaling === "off"`:**
     - Zero post-processes attached; `applyResolutionScale()` sets `engine.setHardwareScalingLevel(1 / (dpr * effectiveScale))` exactly as today (bit-identical off path).
   - **When `renderer.upscaling === "spatial"` and `effectiveScale < 1`:**
     - Set the canvas output backbuffer to native device pixels (`engine.setHardwareScalingLevel(1 / dpr)`), or to the target output pixel dimensions $(W_{\text{out}}, H_{\text{out}})$.
     - Render the 3D scene at reduced internal dimensions $(W_{\text{in}}, H_{\text{in}}) = (\max(1, \text{round}(W_{\text{out}} \cdot s)), \max(1, \text{round}(H_{\text{out}} \cdot s)))$, where $s = \text{effectiveScale}$.
     - **Pass 1 — EASU (Edge-Adaptive Spatial Upsampling):**
       - Reads the $(W_{\text{in}}, H_{\text{in}})$ scene color texture and writes at $(W_{\text{out}}, H_{\text{out}})$.
       - Computes local luminance ($Y = 0.5 G + 0.25 R + 0.25 B$) on the 12-tap neighborhood around the source coordinate $p = (\text{uv} \cdot \text{inputSize} - 0.5)$, accumulates directional gradients $(\text{dir}_x, \text{dir}_y)$ and edge length $L$, rotates/anisotropically stretches the kernel along the edge, and clamps the output to the 4-texel local bounding box `[min4, max4]` to prevent ringing.
     - **Pass 2 — RCAS (Robust Contrast-Adaptive Sharpening):**
       - Runs at $(W_{\text{out}}, H_{\text{out}})$ on the 5-tap cross neighborhood $(N, S, E, W, C)$.
       - Computes the analytical maximum negative lobe weight $w \in [-\text{peak}, 0]$ that does not clip $0$ or $1$ across RGB channels, scaled by `renderer.upscaling.sharpness` (where sharpness $s \in [0, 1]$ maps to FSR attenuation $2^{-2(1-s)}$).
   - **When `renderer.upscaling === "spatial"` and `effectiveScale >= 1`:**
     - Bypass EASU; if `sharpness > 0`, run only RCAS at native resolution (or bypass both when `effectiveScale >= 1` and `sharpness === 0`).
3. **Camera and lifecycle rules:**
   - Attach only to the active presentation camera (respecting camera switches and `onContextLostObservable` / `onContextRestoredObservable`), and dispose all `PostProcess` / `Effect` resources cleanly on `dispose()` or when turned `"off"`.
   - Never call `scheduler.requestRender()` from inside a per-frame render observer; only request a render when a setting or resize changes.

---

### 2.3 Technical design for P8: Dynamic Resolution Scaling

1. **Settings (`src/settings/catalogue.ts`):**
   - `renderer.dynamicResolution`:
     - `kind: "boolean"`, `default: false`, `appliesLive: true`, `home: { tab: RENDERER_TAB, section: "main", level: "main" }`.
     - Description: *"Lowers resolution automatically when the GPU falls behind the frame time target, and raises it slowly when spare GPU time returns."*
   - `renderer.dynamicResolution.minScale`:
     - `kind: "number"`, `unit: "ratio"`, `bounds: () => ({ min: 0.25, max: 1 })`, `step: 0.125`, `default: 0.5`, `appliesLive: true`, `home: { tab: RENDERER_TAB, section: "main", level: "advanced" }`.
2. **Pure controller (`src/dynamicResolution.ts`):**
   - Export `createDynamicResolutionController(options)` with zero DOM or Babylon imports so it can be unit-tested deterministically with synthetic GPU millisecond sequences:
     - **Ceiling:** `maxScale = userResolutionScale` (`renderer.resolutionScale`).
     - **Floor:** `minScale = Math.min(maxScale, userMinScale * maxScale)` (clamped to $\ge 0.25$).
     - **Discrete scale rungs:** Quantize between `minScale` and `maxScale` in steps of `0.125` (or 5 rungs: `0.5, 0.625, 0.75, 0.875, 1.0` × `maxScale`) so the engine never reallocates render targets for sub-pixel float jitter.
     - **Target GPU budget (`targetGpuMs`):**
       - Derived from `renderer.frameRateCap` when numeric (`1000 / cap`), or the observed display cadence (`16.67 ms` default when `"off"`), multiplied by a conservative GPU share (`0.80`, i.e., `13.33 ms` at 60 Hz, `26.67 ms` at 30 Hz).
     - **Asymmetric hysteresis (quick to lower, slow to raise):**
       - *Step down:* When `gpuMs > targetGpuMs` for `downFrames` (e.g., `3`) consecutive timed frames, drop immediately to the rung whose estimated quadratic pixel cost $\text{gpuMs} \cdot (s_{\text{new}} / s_{\text{curr}})^2 \le 0.85 \cdot \text{targetGpuMs}$ (at least one rung down), and reset counters.
       - *Step up:* Only consider stepping up one rung ($s_{\text{next}}$) when `gpuMs` satisfies $\text{gpuMs} \cdot (s_{\text{next}} / s_{\text{curr}})^2 \le 0.75 \cdot \text{targetGpuMs}$ (predictive headroom check!) for `upFrames` (e.g., `45`) consecutive timed frames. This predictive quadratic check prevents limit-cycle oscillation where stepping up immediately pushes `gpuMs` back over `targetGpuMs`.
     - **Cooldown & resize guard:** Ignore frames during an active window resize gesture (`P-044`), when `gpuMs <= 0` (untimed frame), or while the scheduler is paused/idle.
3. **Shared GPU timer (`src/babylon/frameProfiler.ts`):**
   - Refactor `webGpuPassTimer` / `webGlFrameTimer` so `createBabylonRendererLifecycle` and `profileBabylonScene` can both acquire a lease on GPU frame-time measurement on the same `AbstractEngine`. When either consumer is active, GPU timing stays enabled; when the last consumer releases its lease, `enableGPUTimingMeasurements` / `captureGPUFrameTime` is turned off.
   - If `!canTimeGpuFrames(engine)`, set a note on `renderer.dynamicResolution`: `"Unavailable: this browser or GPU does not expose GPU frame timestamps."` and keep `effectiveScale = userResolutionScale`.

---

## 3. Prompt to resume work in a new conversation (with access to `foss-earth`)

Copy everything below the line into a fresh conversation opened in a workspace that includes `/Users/felg/gh/foss-earth` (and optionally `/Users/felg/gh/0sfs`).

---

```text
We are implementing backlog items P9 (Analytical spatial upscaling: FSR 1-style EASU + RCAS) and P8 (Dynamic resolution scaling steered by GPU frame time) in `@foss-earth/renderer` (`/Users/felg/gh/foss-earth/renderer`).

First, read these files to load the context, rules, and complete technical specification:
1. `/Users/felg/gh/foss-earth/.github/AGENTS.md` and `/Users/felg/gh/foss-earth/renderer/AGENTS.md`
2. `/Users/felg/gh/0sfs/docs/proposals/spatial-upscaling-and-dynamic-resolution.md` — our full review of why neural framegen/upscaling (RIFE / RT4KSR) stopped at Stage 1/2 and the complete architecture & design spec for P9 and P8
3. `/Users/felg/gh/foss-earth/renderer/README.md`
4. `/Users/felg/gh/foss-earth/renderer/src/settings/catalogue.ts`
5. `/Users/felg/gh/foss-earth/renderer/src/babylon/rendererLifecycle.ts`
6. `/Users/felg/gh/foss-earth/renderer/src/babylon/frameProfiler.ts`
7. `/Users/felg/gh/foss-earth/renderer/src/vite/babylonBarrels.ts`

Before editing any file, run `git status` in `/Users/felg/gh/foss-earth/renderer` and `/Users/felg/gh/foss-earth/app` to verify working tree state, and preserve any unrelated uncommitted changes across sibling repositories.

Execute the implementation in these self-contained, verified steps:

### Step 1: Pure contracts and unit tests (`@foss-earth/renderer` pure entry)
1. Implement the pure dynamic resolution governor in `src/dynamicResolution.ts` (and export its public types/factory from `src/index.ts`):
   - Discrete resolution scale rungs bounded by `[minScale, userResolutionScale]`.
   - Asymmetric hysteresis: quick to step down (e.g., 3 consecutive frames over GPU budget, jumping to a rung whose quadratic pixel ratio fits within 85% of budget), slow to step up (e.g., 45 consecutive frames where predicted next-rung GPU time `gpuMs * (nextScale / currentScale)^2 <= 0.75 * targetGpuMs` so it never oscillates).
   - Pure spatial-upscaling configuration/math helpers (EASU/RCAS constants, input/output dimension calculation, capability check for `webgpu` and `webgl2` vs `webgl` fallback note).
2. Add comprehensive unit tests in `tests/dynamicResolution.test.ts` and `tests/spatialUpscaling.test.ts` covering:
   - Step-down response, predictive step-up anti-oscillation, user `resolutionScale` ceiling changes, `frameRateCap` changes, untimed (`0 ms`) frames, and WebGL1 fallback notes.
   - Verify that `src/index.ts` remains 100% free of DOM, Babylon, and WebGPU globals so `npm run test:consumer` passes.

### Step 2: Shared GPU frame timer lease (`src/babylon/frameProfiler.ts`)
1. Refactor the GPU frame timer in `src/babylon/frameProfiler.ts` so multiple callers on the same `AbstractEngine` (`profileBabylonScene` and `createBabylonRendererLifecycle` for P8) can acquire/release a GPU timing lease without one caller setting `engine.enableGPUTimingMeasurements = false` or disposing `EngineInstrumentation` while the other is still active.
2. Add unit tests verifying lease reference-counting and cleanup on both WebGPU and WebGL mock engines.

### Step 3: Analytical Spatial Upscaling (P9 — EASU + RCAS in `src/babylon/spatialUpscaling.ts`)
1. Implement `src/babylon/spatialUpscaling.ts` supporting both WGSL (`WebGPU`) and GLSL (`WebGL2`):
   - Remember `babylonBarrels()`: import concrete Babylon modules directly and register both WGSL (`ShaderStore.ShadersStoreWGSL`) and GLSL (`Effect.ShadersStore`) shaders explicitly.
   - When `renderer.upscaling === "off"` (the default, preserving pixel-identical baseline behavior), attach no post-processes and use existing `setHardwareScalingLevel`.
   - When `renderer.upscaling === "spatial"`, render the scene at internal scaled dimensions and reconstruct to output canvas dimensions via EASU (when effective scale < 1) followed by RCAS (controlled by `renderer.upscaling.sharpness`), clamping to the local neighborhood box to prevent ringing.
   - Handle camera changes, resize, context loss/restoration (`onContextLostObservable` / `onContextRestoredObservable`), and full disposal without leaking `PostProcess` or `RenderTargetTexture` instances, and without breaking `render-on-demand`.

### Step 4: Settings catalogue and lifecycle integration (`catalogue.ts` & `rendererLifecycle.ts`)
1. Add `renderer.upscaling`, `renderer.upscaling.sharpness`, `renderer.dynamicResolution`, and `renderer.dynamicResolution.minScale` to `src/settings/catalogue.ts` (defaulting to `"off"` / `false` so existing behavior is bit-identical unless opted in).
2. Wire P8 and P9 into `createBabylonRendererLifecycle` (`src/babylon/rendererLifecycle.ts`), setting appropriate `settings.setNote(...)` messages when a backend lacks support (WebGL1 for P9; missing GPU timestamp support for P8).
3. Update `renderer/README.md` if public exports or settings counts changed.

### Step 5: Verification
Run and report the exact output of the `@foss-earth/renderer` checks:
- `npm --prefix /Users/felg/gh/foss-earth/renderer run typecheck`
- `npm --prefix /Users/felg/gh/foss-earth/renderer run lint`
- `npm --prefix /Users/felg/gh/foss-earth/renderer run test`
- `npm --prefix /Users/felg/gh/foss-earth/renderer run build`
- `npm --prefix /Users/felg/gh/foss-earth/renderer run test:consumer`

Follow all rules in `AGENTS.md`: never add AI co-author attribution, do not push or deploy without explicit permission, and put any temporary scratch files in `renderer/build/`.
```
