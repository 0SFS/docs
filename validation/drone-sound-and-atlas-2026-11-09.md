# Drone Sound Tuning and DSP Optimization Ledger

9 November 2026. Procedural electric sound synthesis and 2D imagery atlas texture-array refactoring validation.

## 1. 2D Imagery Atlas Texture Array (`@foss-earth/renderer` & `@foss-earth/engine`)

### Problem
- On WebGL 2 and WebGPU, the 2D imagery atlas allocated a single flat 2D texture (`createRawTexture2D`), bounding the GPU imagery budget to a single texture's maximum dimension ($4096 \times 4096 = 16\text{ Mpx} \approx 67\text{ MiB}$ at 8-bit RGBA, or $8192 \times 8192 \approx 334\text{ MiB}$ with mipmaps).
- Attempting to set higher memory budgets (e.g. 512–8,192 MiB) either clamped or failed allocation.

### Implementation
- Added `RawTextureBatchArray` / `createRawTexture2DArray` in `@foss-earth/renderer` ([textureDevice.ts](file:///Users/felg/gh/foss-earth/renderer/src/babylon/textureDevice.ts)) wrapping Babylon's `RawTexture2DArray` (`sampler2DArray` on WebGL 2, `texture_2d_array<f32>` on WebGPU).
- Extended `@foss-earth/engine`'s imagery atlas layout ([imageryAtlasLayout.ts](file:///Users/felg/gh/foss-earth/engine/src/terrain/imagery/imageryAtlasLayout.ts)), material plugin ([imageryMaterialPlugin.ts](file:///Users/felg/gh/foss-earth/engine/src/babylon/imagery/imageryMaterialPlugin.ts)), and binding tables ([imageryBinding.ts](file:///Users/felg/gh/foss-earth/engine/src/terrain/imagery/imageryBinding.ts)) to address tiles via 3D coordinates `(u, v, layer)`.
- Updated `@foss-earth/ui` parameter bounds for `map.imagery.gpuBudget` to allow budgets up to 8,192 MiB.
- Verified across `@foss-earth/renderer`, `@foss-earth/engine`, and `@0sfs/fly` test suites.

---

## 2. Drone / Electric Sound Synthesis Improvements (`@0sfs/electric-sound`)

### Problem
- On **Low** quality (`partials == 1` per rotor), the generator previously synthesized only a single bare sine wave at $1\times\text{BPF}$ with zero upper harmonics, while broadband noise extended up to $6\text{ kHz}$. This sounded like a dull sine wave drowned out by harsh static hiss.
- On **Med** and **High**, adding too many high-order Chebyshev harmonics (8–16) and brushless motor whining orders (7x electrical, 14x pole pass) created a synth-like metallic whine that masked the physical propeller air texture.
- An intermediate attempt over-damped the broadband noise (-32 dB reduction from stacked cuts), making the drone sound unnatural and detached from the physical air column.

### Implementation
1. **Low Tier Chebyshev BPF Harmonics**:
   - `partials == 1` synthesizes **3 Chebyshev BPF harmonics** ($1\times, 2\times, 3\times$) using the fast recurrence `s_next = 2 * c1 * s_curr - s_prev` ([lib.rs](file:///Users/felg/gh/0sfs/electric-sound/wasm/crates/electric-sound/src/lib.rs#L446-L470) and [reference.ts](file:///Users/felg/gh/0sfs/electric-sound/src/reference.ts#L278-L300)).
   - Natural `~1/k^2` harmonic weights (`[1.0, 0.30, -0.12]`) with Schroeder-style phase dispersion (`-` on odd upper harmonics) prevent constructive sample-to-sample slope and crest factor stacking, producing a warm blade buzz without extra polynomial `sine()` evaluations.
2. **Med & High Harmonic Balance**:
   - `Med` synthesizes 4 BPF harmonics + the $1\times$ mechanical shaft tone.
   - `High` synthesizes 5 BPF harmonics + the $1\times$ mechanical shaft tone.
3. **Broadband Propeller Air Texture ("Static")**:
   - Retained the physical 1-pole bandpass (`voice.high - voice.low`) and cutoff range (`650 + tip * 32 + (induced + axial) * 42` up to `0.38 * RATE`), preserving the authentic high-speed blade air rush.
   - Modulated air noise around unity (`1.0 + blade_mod_depth * blade_sin + vrs_mod_depth * sine(modulation)`), creating rhythmic blade chop without removing broadband energy.
   - Tuned `broadbandGain` to `0.11` on DJI Mini 2 and `0.14` on DJI Air 2S, with a gentle `0.82x` trim on Low so the air rush stays balanced with the tonal fundamental.
4. **DSP Performance**:
   - Hoisted control-rate math (`sqrt()`, 2 filter-coefficient divisions, anti-aliasing clamps, harmonic weights) to update every `CONTROL_STRIDE = 8` samples (or immediately on reset / tier change), cutting per-sample inner-loop work by ~70% while maintaining exact sample-for-sample parity between Rust/WASM and TypeScript.

---

## 3. Verification & Build Receipts
- `@0sfs/electric-sound`: `npm run build:wasm` compiled `wasm/single/electric-sound.wasm` (23,014 bytes, SHA-256 `842e63a6...`), `npm run check:wasm` verified receipts, `npm test` passed 9/9 unit and parity tests, and `npm run references` generated hover and ramp WAV references.
- `@0sfs/sound`: `node scripts/verify-audio-artifact.mjs` passed with zero drift; `npx vitest run tests/electricAudio.integration.test.ts` passed 17/17 tests.
- `@0sfs/fly`: full production build (`npm run build`) completed successfully with verified JSBSim and Audio artifacts.
