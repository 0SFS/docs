# Temporal upscaling and display-rate frame generation

Status: **proposal. P8 and P9 have shipped as the analytical fallback. Neural frame generation and neural super-resolution stay stopped. This is the next renderer stage.**
Date: **2026-11-04**.

This replaces the idea that EASU plus a camera warp is how 0sfs matches a native game. EASU stays. It is the fallback for a frame that has no motion vectors, and for WebGL1. The quality path is a temporal upscaler with the same inputs as FSR 3.1. Frame generation is one extra presented frame on a high-refresh display, using those same vectors, and it does not pretend to be DLSS 4.5 multi-frame generation.

Owners: the passes, settings, and pure controllers live in `@foss-earth/renderer`. Flight registers the meshes that must not be warped (the cockpit screen, glass, precipitation, clouds) through a host port. The renderer does not import `0sfs`.

---

## 1. What the fastest implementations actually are

Compared in November 2026. Costs below are the vendor's or the reviewer's numbers, not a measurement of 0sfs. "Portable" means the algorithm can be reimplemented in WGSL from published MIT source. "Locked" means the fast path is a driver, a tensor core, or a Metal framework a page cannot call.

### 1.1 Upscalers

| Implementation | How it reconstructs | Where the speed comes from | Measured cost of the upscaler itself | Who can run it | Portable to this renderer |
| --- | --- | --- | --- | --- | --- |
| DLSS 4.5 Super Resolution | Second-generation transformer, temporal, motion vectors in | Tensor cores. The frame-rate win is still "shade fewer pixels," not a free network. Quality is about 67% linear scale. TechSpot, 1440p, RTX 5070 Ti: about +16% to +34% over native TAA depending on the game and the preset. The transformer is heavier than DLSS 3. | The network is small on RTX hardware. Community Arc ports of a comparable ML upscaler land around 0.6–1.2 ms at 1080p, which is the right order for "the reconstructor is cheap, the shading was not." | GeForce RTX 20 and newer for super resolution. Frame generation is a newer card. | No. Weights and tensor cores are not a WebGPU feature. A WebGPU reimplementation of a large NVIDIA-style network has been shown at seconds per frame. |
| FSR 4 / 4.1 ML upscaler | First AMD machine-learning upscaler. FP8. | RDNA 4 matrix units. TechSpot, 1440p Quality, RX 9070 XT: about +14% to +31% over native, in line with DLSS 4 on a 5070 Ti. Image quality jumps past FSR 3.1. | Similar frame-rate uplift to DLSS 4, so the reconstructor is in the same sub-2 ms class on that silicon. | RDNA 4 only (RX 9070 class). Earlier Radeon falls back to FSR 3.1. FP8 is not a WebGPU portable type. | No. |
| XeSS 2 Super Resolution | ML temporal upscaler. XMX on Arc, DP4a elsewhere. | XMX is the fast path. On an Arc A770 at 1080p Quality, XeSS SR was 0.84 ms. The DP4a fallback runs on other GPUs and is slower and softer. | 0.84 ms XMX, that one title and resolution. | Best on Arc. The DP4a network is the only ML upscaler that is even theoretically a WebGPU compute port, and it is still a vendor model. | Not as a dependency. Do not take XeSS as the implementation. |
| MetalFX temporal scaler | Temporal. Color, depth, motion, jitter. macOS 13 and later. Frame interpolation is a separate macOS 26 API. | It runs inside Metal, on the same GPU as the game, with no copy. One M5 Pro validation (1512×949, Lyra): full-resolution temporal anti-aliasing was 21.5 ms GPU against 21.9 ms with it off. Half-scale temporal was 10.7 ms GPU and 90 fps, against 21.9 ms and 45 fps at full resolution. The upscaler is lost in the noise. The win is the smaller shading. | Effectively free next to a 10–20 ms frame on that Mac. | Native Metal, and translation layers that turn DLSS calls into MetalFX. Not WebGPU. wgpu does not expose it. | No. This is the native-Apple bar we cannot call. |
| FSR 3.1 temporal upscaler | Analytical. Jitter, dilated motion vectors, depth, reactive mask, history clamp, then a sharpen. FSR 2's successor. MIT. | The reconstructor is a handful of compute dispatches. AMD's SDK sample, PIX, sharpening off, async compute off, Adrenalin 24.3.1: **RX 7900 XTX, 1080p Quality 0.3 ms, 1440p Quality 0.5 ms, 4K Quality 1.0 ms.** Performance mode is 0.3 / 0.4 / 0.8 ms. An RX 5700 XT is 1.2 ms at 1080p Quality and 4.4 ms at 4K Quality. | See the cell to the left. Memory for the upscaler alone, RX 7900 XTX: about 79 MB at 1080p Quality, 140 MB at 1440p Quality, 306 MB at 4K Quality. | Any GPU that can run the compute. FSR 3.1 frame generation is optional and decoupled, so the upscaler does not require frame generation. | **Yes. This is the implementation to reimplement.** |
| FSR 1 EASU + RCAS | One frame, 12-tap edge filter, then a 5-tap sharpen. No history. MIT. Already shipped here as `easu` and `easu-rcas`. | Cheapest possible reconstruct. Also the softest. Native games kept it only as a fallback. | Designed to be well under 1 ms. Not timed inside an 0sfs frame. | WebGPU and WebGL2, which is why it shipped. | Already in the tree. Stays the fallback. |
| Unreal TSR, Babylon `TAARenderingPipeline` | Engine temporal anti-aliasing. | TSR is a credible high-internal-resolution anti-aliaser and falls behind DLSS and XeSS as the scale drops. Babylon's pipeline is not an upscaler: its own note says it is for stills, and `disableOnCameraMove` defaults to true, which drops history whenever the flight camera moves. | Not a candidate. | In-engine. | Do not enable Babylon's TAA pipeline and call it the feature. |

Digital Foundry's FSR 3.1 review (Ratchet, Spider-Man, and others, 1440p Balanced) is the quality ranking that matters: 3.1 is cleaner than FSR 2, and DLSS and XeSS still lead it in motion. That is the quality we should expect from a faithful port. We will not catch DLSS 4.5 or FSR 4 in a browser.

The performance pattern is the same on every row that is actually fast. The upscaler costs well under 2 ms on a capable GPU. The frame rate comes from shading at 67% or 50% of the linear resolution (about 2.25× or 4× fewer pixels). A reconstructor that costs more than the shading it removes is a loss. That is why RT4KSR, at 9.1 ms for a 720p frame on this M5, is not in this table as a contender.

### 1.2 Frame generation

Frame generation does not make the simulation sample input more often. Every vendor says some version of this. NVIDIA's multi-frame generation, AMD's FSR notes, and Intel's XeSS guidance all still tie responsiveness to the rate of rendered frames. Generated frames are for a high-refresh panel once the rendered rate is already around 60 fps. AMD's FSR 3 note says it runs best from a 60 fps base. Intel's floor for XeSS frame generation is 40 fps, with 60 fps recommended. Below that, the extra frames add latency and smear.

| Implementation | Multiplier | What it needs beyond color | Presentation | Cost | Latency | Portable |
| --- | --- | --- | --- | --- | --- | --- |
| DLSS 4.5 multi-frame generation | 2×, and up to 6× on RTX 50 (five generated frames per rendered frame). Dynamic mode picks the multiplier to approach the refresh rate. | Optical-flow accelerator, motion vectors, depth, Reflex. | The driver flips generated frames. The game does not get a callback per displayed frame. | Higher multipliers also steal GPU time from the rendered frames, so the real frame rate can fall while the counter rises. | One PCGamesN Cyberpunk pass: about 40 ms click-to-display with no frame generation, about 46 ms at 2×, about 60 ms at 4×. A later 6× set spans about 37 ms in Ultra Performance to about 67 ms in Quality. Reflex is what keeps this usable. | No. |
| FSR 4 ML frame generation | 2× class, ML interpolation. | Motion vectors plus the ML path. | A replacement DXGI swapchain (`FSR Frame Generation Swapchain` 4.0). | Vendor's current ML path. | Same class of added latency as other interpolation. | No. RDNA 4, and the swapchain is DirectX 12. |
| XeSS 2 / XeSS 3 frame generation | 2×, and XeSS 3 adds 3× and 4×. | Arc, plus XeLL for latency. | Driver / engine integration. | Not published as a portable kernel time. | Intel's own guidance: do not use it to rescue a slow base rate. | No. |
| FSR 3.1 frame generation | 2×. One interpolated frame between two rendered frames. Decoupled from the upscaler since 3.1, so it can sit on any temporal upscaler that supplies render-resolution motion vectors and depth. | Game motion vectors, depth, and AMD's optical-flow field. Optical flow is a 7-iteration 8×8-block luminance search (sum of absolute differences, 3×3 median, 2× refine). It exists because game vectors miss particles, translucency, and shadows. A scene-change histogram zeros the field instead of inventing motion. | **The part that makes it fast to watch is a replacement DXGI swapchain**, which paces the generated frame onto the display from an async queue. Hardware-accelerated GPU scheduling is recommended. | SDK sample, same conditions as the upscaler table. **RX 7900 XTX: 1080p Quality up to 0.6 ms, 1440p Quality up to 0.9 ms, 4K Quality up to 1.7 ms.** An RX 5700 XT is 1.6 / 2.4 / 5.0 ms. Async compute hides some of this. Memory, frame generation alone: about 72 MB at 1080p Quality, 150 MB at 1440p Quality, 268 MB at 4K Quality, plus swapchain overhead. WCCFtech cites AMD at about 124 / 214 / 457 MB once the swapchain is counted. | Interpolation waits until both surrounding rendered frames exist, then shows the middle one first. That is about half a rendered frame of added delay before the optical-flow cost. A hardware click-to-display comparison (Battlefield, RTX 5070 Ti and RX 9070 XT class): DLSS frame generation about +6.4 ms at 1080p and +17 ms at 4K; FSR frame generation up to about +17.5 ms at 1080p and +13.4 ms at 4K. At a 34 fps path-traced base, both added about 30 ms. | **The shaders, yes. The swapchain, no.** |
| Camera-only reprojection (ASW-style) | One predicted frame. | Depth and the two view-projection matrices. | Can run inside the app, which is why a browser can do it. | A depth reprojection is a small fullscreen pass. | No wait for a future frame. Holes where the new view sees a surface the old frame never shaded. Independent objects slide. | Yes, and it is not the native look. |
| Practical-RIFE 4.25 lite, measured here 2026-10-13 | Would be 2× if it were fast enough. | Two full frames, no engine motion vectors. | ORT WebGPU. | Fastest second inference at 720p was 114 ms. 1080p was about 264 ms. GPU image was 17–23 dB. | Unusable. Also needs the future frame. | Already rejected. |
| RT4KSR x2, same session | Not frame generation. A one-frame 2× network. | The color buffer. | ORT WebGPU. | Fastest second inference 9.1 ms at 720p, about 22 ms at 1080p. Lost to bicubic on real half-resolution renders (28.03 dB against 28.38 dB pooled at 720p). | Adds its whole cost to this frame. | Already rejected. |

GDC 2024's Snowdrop / FSR 3 notes are the integration lesson. Frame generation consumes game motion vectors and optical flow, estimates disocclusion, and fills low-confidence gaps from a mip pyramid. UI has to be drawn in a callback after the generated frame, and that callback's cost counts. Translucency, fast shadows, and vignettes remain the unsolved artifacts. A custom swapchain is how the generated frame reaches the display without the game rendering it.

### 1.3 What "most performant" means for 0sfs

Three different winners:

1. **Best image for the GPU time, on a PC a native game owns:** DLSS 4.5 where the machine is NVIDIA, FSR 4 where it is RDNA 4, MetalFX temporal where it is Apple Silicon. The upscaler is about 1 ms or less. The multiplier on frame generation is a driver feature, up to 6×, and it spends latency to get there.
2. **Best algorithm we can reimplement and expect to finish inside a millisecond on a desktop GPU:** FSR 3.1 upscaler, then FSR 3.1 frame generation's motion-vector and optical-flow interpolation. MIT, published inputs, published costs. Quality behind DLSS and XeSS, far ahead of EASU.
3. **Best thing the browser can present:** the FSR 3.1 upscaler, unchanged. Frame generation only as a single extra animation callback, because a page is scheduled once per refresh and cannot install a swapchain. That caps the multiplier at 2×, and only when the display is fast enough that every other callback can skip the world. There is no 3×–6×, and there is no MetalFX call.

WebNN on the M5 does not join this list. Chrome's `npu` device is Core ML with every compute unit available. The flag that prefers the Neural Engine and leaves the GPU out is off by default. The color buffer is a WebGPU texture, so a Core ML graph pays a copy out and a copy back. None of the fast rows above work that way. They run next to the frame, on the GPU that already holds it.

---

## 2. Decision

Ship a WGSL reimplementation of the FSR 3.1 temporal upscaler, fed by real motion vectors, with EASU kept exactly as it is when those vectors are absent. Add frame generation as a second stage that is off by default, refuses to run below a 60 fps rendered base, and presents at most one generated frame per rendered frame on a display whose refresh is at least 100 Hz.

Do not port DLSS, FSR 4, XeSS, or MetalFX. Do not turn Babylon's `TAARenderingPipeline` on. Do not put another ONNX model in the frame.

The pixel saving is the same shape as native games, and it shows up when shading dominates: a 2× display, a 4K-class canvas, a headset. It does not show up on the audited `/fly/` view. That view, 1280×800 at DPR 1 on the M5, held 58.8 fps with 6.22 ms of main-thread work and a 3.0 ms globe pass, and the GPU process was at 13% of a core. An upscaler cannot shrink imagery selection, tile work, or terrain raycasts. Frame generation cannot invent an animation callback the main thread missed.

---

## 3. What already shipped, and what this must not break

P8 and P9 are in `@foss-earth/renderer` under these ids, which differ from the October draft:

- `renderer.upscaling`: `"off"` (default), `"easu"`, `"easu-rcas"`.
- `renderer.upscalingSharpness`: `0.2`, range `0..1`, step `0.05`. RCAS attenuation is \(2^{-2(1-s)}\).
- `renderer.dynamicResolution`: off. Steps of `0.05` between `0.5` and the manual scale. Drops after 3 over-budget GPU frames, rises one step after 45 comfortable frames, and holds when the CPU missed the frame while the GPU was inside its share.
- `renderer.targetFps`: `60`.
- `acquireGpuFrameTime` reference-counts WebGPU timestamps and the WebGL timer, shared with the profiler.
- Off still means `setHardwareScalingLevel(1 / (devicePixelRatio × resolutionScale))`. No post-process, bit-identical to the renderer before P9.

`DynamicResolutionController` lives in `src/dynamicResolution.ts` and must stay free of DOM and Babylon types. The new decisions go in new pure modules beside it.

Babylon 8.56 can build a geometry-buffer velocity texture, including skinned motion, and it can ignore listed skinned meshes so only the node moves. That is a useful source of vectors. It is not the upscaler. `TAARenderingPipeline` disables itself when the camera moves.

`babylonBarrels()` marks `@babylonjs/core/**/index.js` as `moduleSideEffects: false`. New passes import concrete modules and register GLSL and WGSL sources themselves, the same rule as `src/babylon/spatialUpscalingPostProcess.ts`.

The pure entry, `src/index.ts`, is typechecked with `lib: ["ES2022"]` and no DOM. Motion-vector math, jitter, history weight, and the frame-generation gate are pure. Passes stay under `src/babylon/`.

---

## 4. Upscaler

### 4.1 Settings

Extend `renderer.upscaling` with one choice, `"temporal"`. Default remains `"off"`. Description stays one sentence and names the behavior: history reprojection from motion vectors, with EASU when the frame has none.

Add `renderer.upscalingPreset`, choice, shown only when upscaling is `"temporal"`:

| Id | Linear scale | Pixel fraction | FSR 3.1 name |
| --- | --- | --- | --- |
| `native` | 1.00 | 1.00 | Native AA. History anti-aliasing, no smaller render. |
| `quality` | 0.67 | 0.44 | Quality. Default. |
| `balanced` | 0.59 | 0.35 | Balanced. |
| `performance` | 0.50 | 0.25 | Performance. |
| `ultra` | 0.33 | 0.11 | Ultra Performance. Hidden until Quality and Performance have been looked at in motion. Not in the first cut. |

Default `quality`. These are the rungs. While `"temporal"` is selected, `DynamicResolutionController` may move only between the rungs at or below the user's preset, and a rung change clears history for one frame instead of blending across a resized buffer. The 0.05 step remains the behavior for `"easu"` and `"easu-rcas"`, which have no history.

`renderer.upscalingSharpness` still drives RCAS after the temporal accumulate. Same attenuation function.

WebGL1 sets the existing upscaling note and stays on the manual hardware scale. WebGL2 may run the temporal pass if the multiple render targets it needs are actually available; if the draw of the velocity target fails, fall back to EASU for that session and set a note. WebGPU is the path we tune.

### 4.2 Inputs, matching FSR 3.1

At render resolution, every shaded frame:

- Color, HDR if the scene is HDR, otherwise the scene's working format. Do not feed the DOM HUD through this.
- Depth, the raw device depth, plus the camera's near and far so the pass can reconstruct position. Reversed-Z is already how this renderer starts; the pass must use the same convention.
- Motion vectors, 2 channels, in pixels of the **render** resolution, pointing at where this pixel was in the previous frame. Scale is passed in, not baked as a mystery.
- A reactive mask. Optional in FSR, mandatory here for the cockpit screen, glass, clouds, and precipitation. Texels above a threshold take no history.
- The jitter that was applied this frame, in the same sign convention as the projection offset, so the pass can remove it.
- The previous and current view-projection, so a missing vector on a static pixel can still be rebuilt from depth. This is the camera-only fallback inside the temporal pass, not a second feature.

Jitter is a Halton(2, 3) sequence whose length is the FSR phase count for the current scale (8 phases is the usual Quality count; use the published phase-count rule, \(8 \times (\text{output}/\text{input})^2\), rounded, and at least 8). The offset is applied to the projection's clip-space translation and nowhere else. The same phase is what the history was rendered with. A resize, a rung change, a camera cut, or a device loss sets phase to 0 and history weight to 0 for that frame.

### 4.3 Where the vectors come from

Static world (terrain, tiles, buildings, the photogrammetry that did not move): camera reprojection. Previous clip position minus current clip position, from the depth and the two view-projections. No per-object buffer.

Anything that moved in the world: the velocity target. The aircraft, rotors, control surfaces, wheels, and traffic write it from the previous world matrix (and bones, for a skinned mesh). Babylon's geometry-buffer velocity path is the first implementation to try, with its ignored-skinned list left empty for those meshes. If that path cannot see a mesh the flight cares about, the flight submits the previous world matrix through the host port and a small dedicated pass writes the vectors. Camera reprojection alone is wrong for these pixels, and it is the bug that makes an external camera smear the aircraft across the ground.

Reactive, history weight zero, and excluded from the vector field that frame generation will trust:

- The cockpit-screen mesh. Flight already mirrors the DOM onto it (`fly/src/flight/cockpit/cockpitScreen.ts`). Warping that text is the native-game UI failure.
- Glass and other translucents.
- Precipitation, clouds, and particles.
- The DOM HUD and the GPU attitude indicator are not in the scene color at all. They composite after, as they do now. Leave them there.

The host port is a list of meshes plus a per-frame "this one is reactive" bit. The renderer does not know aircraft types.

### 4.4 The accumulate

Reimplement the published FSR 3.1 upscaler structure in WGSL and GLSL. Do not vendor the SDK, and do not paste the HLSL. The structure, which is what makes it faster and stabler than EASU:

1. Dilate motion vectors and depth so a silhouette samples the foreground's motion, not the background's.
2. Reproject history with the dilated vector. Convert the previous color into YCoCg for the clamp.
3. Clamp history to the current frame's neighborhood (the 3×3 min and max, with the variance clip FSR uses). This is the ghost killer.
4. Drop history where the reactive mask is hot, where depth says the surface is new, or where the vector is invalid.
5. Blend. More history when the vector is small, less when it is large. The first frame after a reset uses the current sample only.
6. RCAS at output resolution, with the existing sharpness control.

Output size is the device-pixel canvas. Input size is output times the preset scale, rounded to at least 1. Hardware scaling stays at \(1 / \text{devicePixelRatio}\) while temporal is on, so the browser is not bilinear-stretching a small canvas at the same time. That double scale is the bug P9 already avoids for EASU.

A frame with no velocity target and no depth falls back to the shipped EASU pass for that frame, still at the preset scale. The user does not see the choice flip.

### 4.5 Cost gate

On the M5, at the canvas size the user actually has, the temporal pass plus RCAS must come in under 2 ms of GPU time, read from the shared lease. FSR's own 1080p Quality number is 0.3 ms on a 7900 XTX and would be higher on an M5 and on an RX 5700-class GPU (1.2 ms at 1080p in AMD's table). 2 ms is the reject line, not the target. If a week of the approach replay sits over it, the preset falls back one rung and the note says so. If Performance mode is still over it, temporal turns itself off for the session and EASU remains.

The comparison that decides whether the feature is worth its default-off setting: the same replay at DPR 2, native scale against Quality temporal. Record GPU pass time, main-thread time, and a side-by-side of runway lights, a propeller, and the cockpit screen. Main-thread time must not rise by more than the jitter and the mask submission. GPU time must fall by more than the upscaler's own cost. Cockpit text must not shimmer.

---

## 5. Frame generation

### 5.1 The presentation fact

A page gets one animation callback per display refresh. FSR and DLSS get their multiplier by presenting frames the game did not render, from a swapchain the browser will not let us replace. So:

- A 60 Hz panel has nowhere to put a generated frame. The setting stays off and the note says the display is too slow.
- A 120 Hz panel can render the world on even callbacks and show a generated frame on odd callbacks. That is 2×, which is FSR 3.1's multiplier, and it is the ceiling.
- 3×, 4×, and 6× are out. They need the driver to flip several frames between callbacks.

Measure the refresh from the spacing of animation callbacks, not from a claimed `devicePixelRatio`. Require a median interval of 10 ms or shorter (100 Hz or faster) across 30 callbacks before generation arms. A 60 Hz timer that occasionally delivers a fast pair must not arm it.

### 5.2 Settings and the gate

`renderer.frameGeneration`, boolean, default false, shown when upscaling is `"temporal"`. Off under every other upscaler, because EASU has no vectors worth interpolating.

Pure gate, `src/frameGeneration.ts`, no DOM:

- Refresh median ≤ 10 ms.
- The last completed world render's GPU time, and the one before it, were each at most the frame budget of the **rendered** rate. On a 120 Hz panel the rendered rate is 60, so the budget is 16.7 ms. This is AMD's "60 fps base" rule, applied to our own timer.
- A velocity target and a depth target exist for the previous world render.
- Not paused, not hidden, not inside the three-frame resize ignore P8 already has, not the frame after a history reset.
- The optical-flow or warp pass itself has stayed ≤ 2 ms. Three strikes and generation disarms until the user turns the setting off and on.

CPU-bound frames do not generate. If the main thread missed the callback, there is no callback to decorate. Generation never calls `scheduler.requestRender()` from inside a frame observer. The scheduler is already running at display rate. The lifecycle chooses "render the world" or "present a generated frame" at the start of `renderFrame`.

### 5.3 Algorithm

Use FSR 3.1's inputs, and accept that we cannot use its swapchain.

On a world callback: render as in section 4. Keep color (after RCAS), depth, dilated motion vectors, and the reactive mask.

On a generated callback:

1. Do not run `scene.render()`.
2. Build the halfway motion. Game vectors, scaled by half the rendered-frame interval, plus optical flow where the game vector is low-confidence. Optical flow is the 8×8 luminance search from the published FSR technique. It is the first thing to cut if it misses the 2 ms budget: vectors alone, then a note that particles and translucency will smear.
3. Reject a scene cut. The FSR histogram test, or a cheaper variant: if the reactive-or-luma change is above the published threshold, show the last real frame again and render a real frame next callback.
4. Disocclusion gets a mip-pyramid fill, the FSR approach, not a network.
5. After the warp, redraw registered overlay meshes with their **current** textures and the current camera. The cockpit screen is the required one. Its DOM mirror already updates on its own schedule (`nextMirrorAt`). The generated frame samples that latest texture. It does not sample the warped pixels underneath.
6. The DOM HUD and the attitude indicator draw as they always do, after the canvas, from the current aircraft state.

This is interpolation data (two real frames) only when both exist. Scheduling it on the off-tick without holding the newer real frame back means the generated image is a half-step **extrapolation** from the last real frame, using vectors, rather than FSR's hold-and-insert. That is a deliberate flight-sim choice. FSR's insert adds about half a rendered frame of latency because the middle frame is shown before the newer real one. Extrapolation does not wait, and its holes are worse, because the future frame is not available to fill them. The proposal accepts the holes. It does not accept the extra delay on top of a browser compositor that already presents later than a native queue, and that has no Reflex and no XeLL.

If a captured approach shows the extrapolation tearing the aircraft in the external view, the fix is the velocity target, not a neural interpolator. If the velocity target is correct and the tear remains, generation stays off for external cameras until that case is understood. Cockpit view is the first camera that must look right.

### 5.4 What generation will not do

It will not raise a 30 fps CPU-bound approach to a fluid 60. The vendors' own guides say the same thing, and our main thread is the usual reason an approach drops. It will not make the controls sample faster. It will not run on a 60 Hz panel.

---

## 6. Frame order

Every animation callback:

1. Flight has already stepped physics. JSBSim stays at 120 Hz. This proposal does not touch it.
2. The camera for this callback is known.
3. If generation is armed and this callback is the off-tick, run section 5.3 and return before `scene.render()`.
4. Otherwise run the world. P8, on its temporal rungs if `"temporal"` is on, otherwise on its 0.05 rungs. Jitter the projection only in the temporal path.
5. The scene renders into the low-resolution targets (color, depth, velocity).
6. Temporal accumulate, or EASU if this frame has no vectors. Then RCAS when the mode includes it.
7. DOM HUD, attitude indicator, and the cockpit-screen mirror composite from the same camera time. They are not inputs to step 6.

A device loss drops the history and the velocity targets (`suspend` already exists for the EASU passes). The restored frame blends with weight 0.

---

## 7. Files

Pure, exported from `src/index.ts`:

- `src/temporalUpscale.ts`. Preset to scale. Phase count. Halton jitter. History weight given vector length, reactive flag, and disocclusion. Rung list for the controller.
- `src/frameGeneration.ts`. The gate in section 5.2. No engine types.
- Tests beside those files. The consumer fixture stays on `ES2022` without DOM.

Babylon, exported from `src/babylon/index.ts`:

- `src/babylon/motionVectors.ts`. Camera reprojection from depth, plus the velocity-target pass.
- `src/babylon/temporalUpscalePass.ts`. WGSL and GLSL. Concrete module imports.
- `src/babylon/frameGenerationPass.ts`. Warp, scene-cut reject, overlay redraw.
- `rendererLifecycle.ts` chooses the path. It does not grow a second render loop.

Flight, in `0sfs` and not imported by the renderer:

- Register the cockpit-screen mesh, glass, cloud, and precipitation as reactive.
- Confirm the aircraft and its rotors write velocity. A test with a known camera move and a known rotor angle checks that the external view's aircraft pixels are not the ground's vectors.

Settings copy stays in the catalogue's voice. One sentence each. Pixel-changing options default to today's picture, which is `"off"`.

---

## 8. Out of scope

- DLSS, FSR 4, XeSS, MetalFX, and any ONNX or WebNN model, including another look at RIFE or RT4KSR.
- Multipliers above 2×, and frame generation on displays slower than 100 Hz.
- Changing the 0.05 dynamic-resolution behavior while the upscaler is EASU.
- Turning the feature on by default, or advertising it as DLSS.
- A swapchain of our own. If a future browser API lets a page present twice per callback, this design is the one that can use it. Until then the off-tick is the API.

---

## 9. Read before implementing

1. This file.
2. `docs/proposals/spatial-upscaling-and-dynamic-resolution.md`, for the neural gates and the reason EASU shipped.
3. `foss-earth/renderer/src/settings/catalogue.ts`, `src/dynamicResolution.ts`, `src/spatialUpscaling.ts`, `src/babylon/spatialUpscalingPostProcess.ts`, `src/babylon/frameProfiler.ts`, `src/babylon/rendererLifecycle.ts`.
4. AMD's FSR 3.1 upscaler and frame-generation pages, and the optical-flow technique note, for the algorithm. Reimplement it. Do not copy the SDK into the repository.
5. `fly/src/flight/cockpit/cockpitScreen.ts`, for the mesh that must be redrawn after a generated frame.
