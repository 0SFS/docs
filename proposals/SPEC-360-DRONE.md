# SPEC-360-DRONE

Implementation specification for the removable 360° drone mod.
Hand this file to the coding agent. Every architecture choice, file path, type, equation, buffer layout, and state transition below is already decided. Translate it. Do not invent a second design.

Targets for the manual checks in the last section: Firefox on an M5 MacBook Air, and Chrome on a Galaxy Note 20 Ultra 5G. Both are WebGPU. The drone pipeline has no WebGL path.

---

## 0. Decisions already made

1. **The drones are declarative mods, not built-in aircraft.** Repository content lives in `Quadcopter/`, `DJI-Quadcopter/`, `DJI-Mini2/`, and `DJI-Air-2S/`. None of them is in `fly/src/flight/mods/modSession.ts` `OFFICIAL`, `BUNDLED_REPOS`, `fly/src/flight/aircraft/fleet.ts`, or `installBuiltinMods.ts`. A fresh flight downloads nothing from them.
2. **Adding the mod downloads only `0sfs-mod.json`.** That registers the gallery entries. It does not import the flight code, does not fetch JSBSim XML, and does not start a capture pipeline.
3. **JSBSim XML is fetched only when a drone flight is booted**, through the existing `hydrateModJsbsimData` (`fly/src/flight/jsbsim/hydrateJsbsimData.ts`), from the mod `baseUrl`.
4. **Babylon, WebGPU, the flight controller and rotor model, optical flow, capture, zip export, and the phone quad UI are a lazy chunk.** The startup module graph may import `fly/src/flight/drone/droneGate.ts` and the small data-only quad/camera-preview registries; it imports no drone implementation. `droneGate.ts` contains one dynamic `import("./droneFeature")`, and that call runs only when the pilot applies a drone aircraft. There is no idle prefetch and no hover prefetch. Invariant 4 of the [JIT code-download specification](https://github.com/foss-earth/docs/blob/main/docs/proposals/jit-code-download.md) applies: off until the pilot asks. Prefetch `modulepreload` is not used for this feature.
5. **Airframe and camera-profile mods stay `execution: "declarative"`.** Do not ship a `js` or `wasm` entry in those mods. A WASM mod cannot import the host (`mods/src/wasmRunner.ts` rejects imports) and cannot touch WebGPU or Babylon. Flight, sensor rendering, capture and phone implementations stay behind the host lazy chunks. The optional camera-preview display provider is a separate JS plugin, described below, and uses the existing mod code confirmation rules.
6. **Removing the mod unloads the feature.** The Mods tab gains Remove. Removal disposes the live feature, unregisters the aircraft, and if the current aircraft is a drone, switches to `cessna-172`. The chunk is not imported again until a drone is applied while the mod is installed.
7. **Physics integration is `@0sfs/multirotor`'s rotor plant and flight controller at 120 Hz, writing one body-axis force and moment at the CG to JSBSim external reactions.** `sdk.run()` remains the only integrator. Do not write latitude or attitude over the integrator. Rotors come from `@0sfs/multirotor`, not from JSBSim propulsion, XML functions or FCS. The XML aerodynamics are body drag only, with no lift and no aerodynamic moments. (Revised 28 October; the original was a cascaded P controller writing world-axis forces to a zero-aero XML.)
8. **WebGPU is required for capture and optical flow.** If `runtime.renderer.mode !== "webgpu"`, the feature refuses capture and flow and reports that in telemetry. It does not read the canvas, does not set `preserveDrawingBuffer`, and does not add a WebGL fallback. The globe renderer in FOSS Earth keeps its existing fallback. Do not edit it.
9. **Default physics rate stays 60 Hz for every other aircraft.** Applying a drone sets `osfs.physics.rate` to `"120"` for that flight and restores the previous rate when leaving the drone.
10. **Optical-flow reduction is TypeScript in the lazy chunk.** Do not add a Rust/WASM crate. Attitude it fuses with is the JSBSim property tree, which is already WASM.
11. **Camera follow defaults belong to aircraft data.** `AircraftDefinition.camera.follow` selects Aircraft, Momentum, or Planet. Planes declare Aircraft; quadcopters declare Momentum. Pilot choices override those defaults. The chase setting remains `osfs.camera.chaseFrame`; old Attitude values migrate to Aircraft, and old No roll/Heading values migrate to Momentum.
12. **Propellers and gimbals are shared host components.** `rotating-prop/` and `camera-gimbal/` each provide a local TypeScript package and a declarative mod manifest. C172 depends on `0SFS/rotating-prop`; Quadcopter depends on both. The generic C172 prop code may load at startup; drone binding and camera-gimbal code stay behind the drone feature. These components neither integrate physics nor run remote JavaScript.

---

### Modular cameras and rate controls, 28 October follow-up

- `drone-camera/` owns offscreen WebGPU still capture, JPEG EXIF/XMP, zip export, optical flow and spherical sequencing. It imports no fly modules. The small declarative registry stores raw installed records; the lazy feature supplies its profile builder on apply, so camera code stays off the startup and mod-add graphs. JPEG remains a dynamic import at the shutter.
- Render targets explicitly select enabled loaded scene meshes. Submit Babylon's pending WebGPU draws before direct readback or flow compute; normalize the bottom-up RTT image to top-down pixels for both JPEG and flow. The target recomputes the camera projection at the sensor aspect ratio, preserving canvas size. Fly supplies a generic visibility predicate to exclude its own airframe and orbit-compass helpers. The shared capture package contains no host-specific mesh names.
- The local JPEG encoder returns Uint8Array when the browser has no Node Buffer. It remains a dynamic import only inside capture and stays in optimizeDeps.include. A real 4000×3000 Mini 2 still was inspected after the browser fix; full 360 and device checks remain manual.
- `DJI-FC7303/` (`0sfs/dji-fc7303`) owns the Mini 2 sensor/lens/exposure/metadata profile. `DJI-FC3411/` (`0sfs/dji-fc3411`) owns the Air 2S equivalent. Each depends on `0SFS/drone-camera`. The airframes select `quad.cameraId` and depend on their camera mod. Mass, motion and gimbal mount limits remain on the airframe; DJI-Quadcopter owns the gallery family and roll stabilization setting.
- `AircraftDefinition.controls.vertical` selects `throttle` or `climb-rate`. Both DJI airframes select `climb-rate`. Their desktop lever reads CLIMB and shows commanded m/s, springing to zero on release/cancel/blur. Shift/Ctrl and rate buttons are instantaneous normalized climb/descent commands; they never integrate motor throttle. Center commands altitude hold. Planes and a future FPV airframe can retain throttle control.
- `controls.keyboard` declares per-aircraft key-pair defaults. DJI defaults are W/S fore/aft, Q/E lateral, A/D yaw and Shift/Ctrl climb/descent. Controls → Keyboard axes chooses each pair live; choosing an occupied pair swaps the displaced axis. The on-screen key hint reads the same settings. Phone left-stick wording is CLIMB / YAW.

### Optional live preview, 28 October follow-up

`camera-preview/` is a separate optional local mod/package (`0sfs/camera-preview`). Add `0SFS/camera-preview` through **Mods → This computer**, then select it in the Camera tab's **Preview** choice. **Off** disables the display; installed alternatives share that same choice. No aircraft, camera-model or capture mod depends on a preview provider. Removing the preview leaves the selected aircraft, gimbal controls and still/360 capture available.

The preview mod owns presentation and pacing, with no aircraft, camera-model, Babylon, JPEG or flight-controller imports. Its manifest declares `kind: "plugin"`, `execution: "js"`, `js: { "path": "preview.js" }`, and `cameraPreview: { "apiVersion": 1 }`. The loader validates the API and relative module path, applies its existing security/confirmation rules, and registers only `{ id, label, moduleUrl }` on add. Unsupported APIs fail before code is loaded. The selected ES module is imported only after a drone is applied and the Camera tab is visible; startup and mod add fetch no preview implementation. There is no preview prefetch.

Replacement modules export this provider contract:

```typescript
createCameraPreview({ document, label, readFrame, maxFps?, attachInteraction? }): {
  element: HTMLElement;
  setActive(on: boolean): void;
  destroy(): void;
}
// readFrame(): Promise<{ width: number; height: number; rgba: Uint8Array }>
// attachInteraction?(canvas): { setActive(on: boolean): void; destroy(): void }
```

Pixels are packed RGBA with the first row at the top. Providers start inactive. The default provider serializes reads at up to 15 fps, ignores stale frames after hiding, shows errors with Retry, and releases its canvas on destruction. Closing or hiding the tab deactivates it; choosing Off, removing the provider, changing sensor bindings or disposing the feature releases its resources. Reopening the tab resumes the selected preview.

`drone-camera/sensorFrame` owns the WebGPU sensor-frame renderer; `fly/src/flight/drone/droneCameraPreview.ts` adapts the active stabilized gimbal camera to the provider. Preview uses a bounded 640-pixel-wide target with sensor-derived height, the selected camera's vertical FOV, and the same host mesh visibility predicate as capture. It neither resizes the main canvas nor follows the chase view, and adds no optical-flow compute or JPEG import. It refuses non-WebGPU renderers. Render/readback work remains in the shared sensor renderer rather than being copied into a display provider.

Camera identity, sensor dimensions, lens FOV and metadata stay in the camera-model mod selected by `quad.cameraId`; motion and mounting stay in the airframe. An aircraft variant can depend on another camera profile and change that ID without changing the preview component or duplicating capture code. The preview receives pixels from whichever camera is selected and contains no sensor or aircraft constants.

The Camera tab places Roll above the central preview, Yaw below it and a vertical Pitch slider to its right. Capture commands follow the gimbal controls. The optional display remains a host-owned slot; the surrounding sliders work with no preview installed.

The optional `attachInteraction(canvas)` port supplies an initially inactive host input binding. Providers forward visibility and destruction to it. Fly reuses the 360 viewer's `attachPanoramaLook` controls and Photograph → Looking settings for drag, two-finger swipe, wheel, pinch and glide. Aiming changes bounded gimbal yaw/pitch targets and leaves roll to its slider; a 360 sequence locks manual input. Slider changes resynchronize the gesture target. Zoom narrows only the preview projection, from the native lens view down to the shared minimum FOV. Rendering restores the shared sensor camera's lens and projection synchronously before awaiting readback, preserving still/cockpit/flow views. Hidden/Off/removed previews release held gestures and glide as well as frame work. A provider need not import the input implementation or depend on an aircraft.

### Quadcopter engine widget, 28 October follow-up

`engines/quadEngineMonitor` reuses the Cessna RPM summary, shaft renderer and simulation-time animation for four motor circles in a top-view X layout: front-left/front-right above rear-left/rear-right, with a nose arrow. The compact engine widget opens the existing Engine tab, which shows the same motor layout at a larger size. The quad display omits combustion-engine fuel, history and test-stand controls. Each circle has its own RPM; the component receives plain data and imports no drone controller, airframe or capture code. Its factory is supplied by the applied lazy drone feature through a type-only hook, so startup/mod add fetch no quad widget or flight model.

Each circle shows its rotor's **simulated** speed from `@0sfs/multirotor`: the same rotor speed that makes that rotor's thrust. It follows the flight controller's command through a first-order lag; the ESC and motor are not simulated, so it is not ESC telemetry. The telemetry is `{ aircraftId, label, hoverRpm, maxRpm, simTimeS, motors: [{ position, rpm, direction }], source: "simulated" }`. `hoverRpm` is the static-thrust hover speed at the live mass and density. `maxRpm` is the propeller's `maxRpm`, the animation's envelope. (Revised 28 October; before that the widget showed an **Estimated** RPM worked backward from the old controller's force request.)

Actual prop pivots are classified by their position in the aircraft root, since HD and procedural asset rotor indices differ. The same per-position simulated RPM drives prop animation and the monitor; spin direction comes from the rotor layout's `spin`, and adjacent corners counter-rotate and opposite corners share a direction. The clock advances only with physics. Pause holds readings and motion; a paused first apply shows zero until the first physics step. Aircraft changes/removal clear stale motor telemetry. These local readings do not enlarge the phone heartbeat. Quad gauge drawing borrows the active WebGPU device and falls back to numeric readouts without attempting WebGL or requesting a second device; existing plane gauge backend behavior stays unchanged.

## 1. Task breakdown and dependency graph

Sequential spine:

1. Phase 0 doc edit (independent).
2. Catalog unregister + family-id registration + `engine: "none"` + bootstrap skip + `createJsbsimRuntime` mod hydrate branch + `removeMod`. These are the host seams. No drone math in these files.
3. Declarative mod: `drones/public/0sfs-mod.json` and the two JSBSim XML files.
4. `droneGate.ts` only. Wire `createFlightSimApp.ts` to the gate with null-hook calls. Confirm the app file does not import `droneFeature`.
5. After step 4, the lazy chunk and the phone chunk can be written in parallel.
6. Wire `switchAircraft` to `await loadDroneFeature()` only when the applied id is a drone and the mod record exists.
7. Verification.

Parallel after step 4:

| Track | Files | Depends on |
| --- | --- | --- |
| A. Dynamics, gimbal, mesh | `droneFeature.ts`, `droneTypes.ts`, `droneController.ts`, `droneFlightModel.ts`, `droneGimbal.ts`, `buildDroneProceduralMesh.ts`; the rotors and flight controller in `multirotor/` | gate types, JSBSim XML property names |
| B. Capture and packaging | `captureFrame.ts`, `jpegExif.ts`, `zipStore.ts`, `fly/package.json` (`jpeg-js`) | WebGPU device from the host, profiles from A |
| C. Optical flow | `flowWgsl.ts`, `opticalFlow.ts`, `flowEstimator.ts` | capture proxy size from profiles |
| D. Spherical sequence | `sphericalGrid.ts`, `captureSequence.ts` | A + B + C |
| E. Phone UI | `rc/src/drone/droneProtocol.ts`, `rc/src/drone/PhoneQuadcopter.tsx`, `rc/src/drone/loadQuadcopter.ts` | wire types in section 6; no static import from `PhoneController.tsx` except `loadQuadcopter` |

Do not start track A–E by adding imports to `createFlightSimApp.ts` or `PhoneController.tsx` beyond the gate and the loader function.

---

## 2. Phase 0 — docs, WebGPU-only drone pipeline

### File: `.github/AGENTS.md`

Insert this row as the last row of the task table:

```markdown
| 360° drone mod, drone capture, optical flow, quadcopter phone controls, drone flight model | [SPEC-360-DRONE](../docs/proposals/SPEC-360-DRONE.md); its section 4.8 gives the frames and signs. WebGPU is the only backend for that pipeline. The drones are the declarative `0SFS/Quadcopter`, `0SFS/DJI-Quadcopter`, `0SFS/DJI-Mini2`, and `0SFS/DJI-Air-2S` mods. Runtime code loads through `droneGate.ts` only when a drone aircraft is applied. The flight model is [`multirotor/`](../multirotor/README.md). The [flight-model roadmap](../docs/proposals/360-drone-flight-model-roadmap-2026-10-28.md) has its status and open decisions. |
```

### File: `docs/parity-checklist.md`

Append this section. Do not rewrite RENDER-01, RENDER-02, or RENDER-03. Those still describe the globe.

```markdown
## 360° drone mod

The drone pipeline follows `docs/proposals/SPEC-360-DRONE.md`.

- The drones are absent until the pilot adds `0SFS/DJI-Mini2` or `0SFS/DJI-Air-2S`. That add also fetches `0SFS/DJI-Quadcopter` and `0SFS/Quadcopter`. Adding does not download the drone chunk. Network shows the manifest JSON files only.
- Applying DJI Mini 2 or DJI Air 2S is the demand load. The drone chunk request starts then, and not before.
- Removing an airframe drops that aircraft. Removing `0SFS/DJI-Quadcopter` or `0SFS/Quadcopter` drops the airframes that depend on it. If a drone was selected, the flight returns to the Cessna 172.
- Capture and optical flow run only when the renderer mode is `webgpu`. A WebGL renderer shows the refusal on the phone and does not read pixels from the canvas.
```

Do not edit `foss-earth/renderer`. Do not delete WebGL fallback from the globe.

---

## 3. Mod identity, add, remove, and the lazy gate

### 3.1 Mod repositories

The drone airframes and camera profiles are declarative mods. Their host-linked camera implementation and flight controller remain behind the fly lazy feature; the phone quad UI remains behind its rc lazy feature. Installing a manifest never executes remote JavaScript.

On reload, `createFlightSimSession` awaits `ensureModsSession()` before reading the saved flight or normalizing the saved aircraft selection. Both validate IDs against the live catalogue. Reading them before saved mods are restored rejects a drone snapshot and falls back to Cessna/start position. Restoring manifest data alone still loads no drone feature or JPEG encoder; only the restored drone selection applies its lazy feature. Respect Resume last flight being off and an explicit URL start position as before.

| Repo | id | kind | Depends on | Owns |
| --- | --- | --- | --- | --- |
| `rotating-prop/` | `0sfs/rotating-prop` | plugin | — | Shared frame-rate-aware propeller presentation: blade phase, alias limit, generated 3D swept volume (high-RPM default) and opt-in flat disc; also used by C172. Local Git repository, no commits or remote. |
| `camera-gimbal/` | `0sfs/camera-gimbal` | plugin | — | Shared roll/pitch stabilization, pitch slew and camera adoption |
| `drone-camera/` | `0sfs/drone-camera` | plugin | — | Shared WebGPU capture, metadata, flow, sequencing and zip |
| `camera-preview/` | `0sfs/camera-preview` | plugin (optional JS) | — | Interchangeable live-preview display and pacing |
| `DJI-FC7303/` | `0sfs/dji-fc7303` | plugin | `0SFS/drone-camera` | Mini 2 camera profile |
| `DJI-FC3411/` | `0sfs/dji-fc3411` | plugin | `0SFS/drone-camera` | Air 2S camera profile |
| `Quadcopter/` | `0sfs/quadcopter` | plugin | `0SFS/rotating-prop`, `0SFS/camera-gimbal` | Stick `kPos` and deadzone, position-hold speed, grid overlap, gimbal slew, rotor wrench and arming property names, 120 Hz |
| `DJI-Quadcopter/` | `0sfs/dji-quadcopter` | plugin | `0SFS/Quadcopter` | Gallery family id `dji-quadcopter`, mount roll stabilization |
| `DJI-Mini2/` | `0sfs/dji-mini2` | aircraft | `0SFS/DJI-Quadcopter`, `0SFS/DJI-FC7303` | Mini 2 flight modes, propeller, rotor layout, CG, controller gains, mount limits, JSBSim XML (mass, inertia, feet, body drag), controls, one gallery variant |
| `DJI-Air-2S/` | `0sfs/dji-air-2s` | aircraft | `0SFS/DJI-Quadcopter`, `0SFS/DJI-FC3411` | Air 2S flight modes, propeller, rotor layout, CG, controller gains, mount limits, JSBSim XML (mass, inertia, feet, body drag), controls, one gallery variant |

Each aircraft manifest carries the family with only its own variant. `registerAircraftMod` appends a variant when the family already exists, so installing both drones yields one DJI family with two variants. `thumbnail.path` stays `""`.

Topics, when published: `0sfs-mod`, plus `0sfs-plugin` or `0sfs-aircraft`. Do not publish from this task.

Local dev: Vite serves each `public/` directory at `/dev-mods/<repo-lower>/` as static files. The Mods tab offers GitHub or This computer. This computer sends `local:0SFS/DJI-Mini2`, which loads `/dev-mods/dji-mini2/0sfs-mod.json` and keeps dependency shorthands on this computer too. The recorded manifest directory is the JSBSim `baseUrl`, so a GitHub add still asks `https://0sfs.github.io/` for `jsbsim-data/` even when a fetch rewrite served the manifest. These four repos are unpublished. Use This computer until they are. A copy already saved as `0SFS/DJI-Mini2` stays the GitHub one until it is removed and added again.

### 3.2 Host seams that stay in the startup graph

These files change, and the change is the gate, not the feature.

**`aircraft/src/aircraftIds.ts`**

- Add `extraFamilyIds = new Set<string>()` next to `extraAircraftIds`.
- `registerAircraftFamilyId(id: string)`, `unregisterAircraftFamilyId(id: string)`.
- `isAircraftFamilyId` returns true for the const tuple or `extraFamilyIds`.
- `unregisterAircraftId(id: string)` deletes from `extraAircraftIds` and refuses to delete a built-in id.

**`aircraft/src/aircraftCatalog.ts`**

- Add optional `procedural?: "drone"` on `AircraftDefinition`.
- Add `aircraftCatalogRevision(): number`, starting at 0. Increment it at the end of `registerAircraftMod` and `unregisterAircraftMod`.
- `unregisterAircraftMod(familyId: string, aircraftIds: readonly string[])`:
  - Remove those ids from `AIRCRAFT_CATALOG`.
  - `unregisterAircraftId` for each.
  - If the family has no remaining definitions, remove it from `AIRCRAFT_FAMILIES` and `unregisterAircraftFamilyId`.
  - Increment the revision.
- `registerAircraftMod` calls `registerAircraftFamilyId(family.id)` before pushing the family.
- Change `AircraftFamilyDefinition.id` to `string`. Built-in families still use the existing ids.

**`fly/src/flight/hud/FlightControlPanel.tsx`**

The `aircraftSelections` state is initialized once from `AIRCRAFT_FAMILIES`. Subscribe to `aircraftCatalogRevision` (poll it in the snapshot the panel already receives, or pass the revision from the host). When it changes:

- Append selections for families that are new.
- Drop selections whose family is gone.
- If `selectedAircraftFamily` is gone, set it to `AIRCRAFT_FAMILIES[0].id`.

**`fly/src/flight/jsbsim/fdmProfiles.ts`**

Extend `FdmProfile`:

```typescript
engine: "piston" | "turbine" | "none";
dataSource?: { kind: "mod"; modId: string };
kinematic?: "drone-pgps";
requiresPhysicsHz?: 120;
```

`registerFdmProfile` already accepts a partial structural profile. Add `unregisterFdmProfile(id: string): void` that `delete`s `FDM_PROFILES[id]` and the matching `engineModelProfiles` keys.

**`engines/src/engineControl.ts`**

`EngineStartProfile.engine` gains `"none"`. `createEngineControl`, when `profile.engine === "none"`, returns a stub: `step` and `shutdown` do nothing, `reading()` is `{ state: "running", startProgress: 1, blocked: null }`. It does not read `propulsion/engine`.

**`fly/src/flight/jsbsim/bootstrapC172.ts` and `resetFlightLocation.ts`**

When `profile.engine === "none"` or `profile.kinematic === "drone-pgps"`:

- Do not call `propulsion/set-running`, `propulsion/engine/set-running`, `propulsion/magneto_cmd`, or `fcs/mixture-cmd-norm`.
- Still `runIc()` twice, as the function does today, so external reactions and ground contacts initialize.
- `initialProperties` still run.

**`fly/src/flight/jsbsim/createJsbsimRuntime.ts`**

After the aircraft id is known, if `getFdmProfile(aircraftId).dataSource?.kind === "mod"`:

- Resolve the installed `LoadedModRecord` whose `manifest.id` equals `dataSource.modId`. If it is missing, throw `The ${modId} mod is not installed.`
- Replace `downloadJsbsimData(...)` with `downloadModJsbsimFiles(record.baseUrl, record.manifest.jsbsimFiles ?? [])`.
- Do not fetch `public/jsbsim-data/manifest.json` for that aircraft.

Pass the mod lookup in as a function argument `findMod(modId: string): { baseUrl: string; jsbsimFiles: readonly string[] } | null` so this file does not import the mod session. `createFlightSimApp.ts` supplies it from `getModsSession().records`.

**`aircraft/src/createAircraftModel.ts`**

Resolve the selected LOD first. For an empty-path LOD with `definition.procedural === "drone"`, mount the aircraft-local `droneMeshHook` once for that aircraft and LOD; an Auto refresh at the same level must retain the same meshes. The hook returns a disposer owned by the model loader. A nonempty-path HD LOD uses the normal glTF loader. Switching either way clears the old geometry when its replacement is ready. `@0sfs/aircraft` imports its own mesh hook, never the fly feature.

**`fly/src/flight/mods/applyDynamicMod.ts`**

After the existing aircraft registration, no extra call. Do not import `droneFeature.ts`. Registration alone is the add path.

**`fly/src/flight/mods/modSession.ts`**

Add `removeMod(source: string): Promise<void>`:

- `await ensureModsSession()`.
- Normalize `source`. Refuse when `normalized.bundled` is true (built-ins stay).
- Drop that canonical key from `records`, from `added`, and from `confirmed`. Write both localStorage lists.
- If `manifest.kind === "aircraft"`, call `unregisterAircraftMod` with the family id and definition ids, and `unregisterFdmProfile` for each fdm key.
- Call `droneGate.onModRemoved(manifest.id)` (section 3.3).
- `publish()`.

`ModsPanel` (`mods/src/ModsPanel.tsx`): each installed row that is not `listedAbove` and whose record is not bundled gets a Remove button. `ModAboutPart` gains optional `source?: string` and `removable?: boolean`, set in `aboutTree.ts` from `record.source` when `record.source.bundled` is false. `onRemove(source: string)` is a new prop. `ModsTab.tsx` passes `removeMod`.

Also delete the source from the `?mods=` share URL by rewriting `added` (already how `shareUrl()` works).

### 3.3 `fly/src/flight/drone/droneGate.ts`

This is the only drone module the startup graph imports. Allowed imports: none from `./droneFeature`, none from `@babylonjs/core`, none from `jpeg-js`.

```typescript
export const DRONE_MOD_ID = "0sfs/dji-quadcopter";
export const DRONE_AIRCRAFT_IDS = ["dji-mini-2", "dji-air-2s"] as const;
export type DroneAircraftId = (typeof DRONE_AIRCRAFT_IDS)[number];

export function isDroneAircraftId(id: string): id is DroneAircraftId {
  return (DRONE_AIRCRAFT_IDS as readonly string[]).includes(id);
}

export interface DroneMeshMount {
  mount(modelRoot: import("@babylonjs/core").TransformNode): void;
  dispose(): void;
}

export interface DroneFeatureHooks {
  beforeStep(step: DroneStepContext): void;
  afterView(view: DroneViewContext): void;
  dispose(): void;
}

export interface DroneStepContext {
  dtSeconds: number;
  sdk: import("@felipegalind0/jsbsim").JSBSimSdk;
  command: DroneStickCommand;
  terrainMeters: number | null;
}

export interface DroneViewContext {
  state: import("@0sfs/ground-contact/flightState").FlightState;
  scene: import("@babylonjs/core").Scene;
  aircraftRoot: import("@babylonjs/core").TransformNode;
  requestRender(): void;
}

export interface DroneStickCommand {
  pitch: number;
  roll: number;
  vertical: number;
  yaw: number;
  yawMode: "angularVelocity" | "angle";
  yawAngleDeg: number | null;
  gimbalPitchDeg: number;
  sport: boolean;
  shutter: boolean;
  capture360: boolean;
}

type FeatureModule = {
  attach(hooks: { setHooks(next: DroneFeatureHooks | null): void; setMesh(next: DroneMeshMount | null): void }): void;
};

let hooks: DroneFeatureHooks | null = null;
let mesh: DroneMeshMount | null = null;
let loading: Promise<void> | null = null;
let disposed = false;

export function droneMeshHook(): DroneMeshMount | null {
  return mesh;
}

export function droneHooks(): DroneFeatureHooks | null {
  return hooks;
}

export function loadDroneFeature(): Promise<void> {
  if (disposed) return Promise.reject(new Error("The drone mod was removed."));
  loading ??= import("./droneFeature").then(module => {
    const feature = module as FeatureModule;
    feature.attach({
      setHooks(next) { hooks = next; },
      setMesh(next) { mesh = next; },
    });
  });
  return loading;
}

export function onModRemoved(modId: string): void {
  if (modId !== DRONE_MOD_ID) return;
  disposed = true;
  hooks?.dispose();
  mesh?.dispose();
  hooks = null;
  mesh = null;
  loading = null;
}

export function onModAdded(modId: string): void {
  if (modId === DRONE_MOD_ID) disposed = false;
}
```

`applyDynamicMod` calls `onModAdded` when the manifest carries `quadcopter`, `dji`, or `quad`. That call does not import the feature. `DRONE_MOD_ID` is the DJI family plugin. Removing the last quad aircraft disposes the feature.

`loadDroneFeature` is the only `import("./droneFeature")` in the repository. A test reads `createFlightSimApp.ts`, `PhoneController.tsx`, `fleet.ts`, and `installBuiltinMods.ts` and asserts none of them contain `droneFeature`, `flowWgsl`, `captureFrame`, `droneController`, or `PhoneQuadcopter`.

### 3.4 Where the flight calls the gate

**`createFlightSimApp.ts` `switchAircraft`**, immediately before `createJsbsimRuntime`:

```typescript
if (isDroneAircraftId(selection.aircraftId)) {
  const installed = getModsSession().records.some(record => record.manifest.id === DRONE_MOD_ID);
  if (!installed) return "Install the DJI quadcopter mod from the Mods tab.";
  await loadDroneFeature();
  const previousRate = parameters.get(PHYSICS_RATE_PARAMETER_ID);
  if (!droneRateRestore) droneRateRestore = previousRate;
  parameters.set(PHYSICS_RATE_PARAMETER_ID, "120");
} else if (droneRateRestore) {
  parameters.set(PHYSICS_RATE_PARAMETER_ID, droneRateRestore);
  droneRateRestore = null;
}
```

`droneRateRestore` is a `string | null` closed over by `switchAircraft`. The same force-to-120 runs on the initial `createJsbsimRuntime` at the `physicsHz: readPhysicsHz(parameters)` call when the resumed aircraft id is a drone and the mod is installed. If the resumed id is a drone and the mod is not installed, substitute `cessna-172` and do not call `loadDroneFeature`.

Inside the physics `applyInputs` callback, after `applyFlightControls` and before `return contact`:

```typescript
droneHooks()?.beforeStep({ dtSeconds: physicsDt, sdk: jsbsim.sdk, command: currentDroneCommand(), terrainMeters });
```

`currentDroneCommand()` reads the phone drone command if the owner is the phone, otherwise maps the desktop `ControlSurfaceState` (section 6.3). When the aircraft is not a drone, `droneHooks()` is null and this is one null check. It does not import the feature.

Immediately after `floatingOrigin?.apply(displayState)`:

```typescript
droneHooks()?.afterView({ state: displayState, scene: runtime.scene, aircraftRoot: floatingOrigin.aircraftRoot, requestRender: () => runtime.requestRender() });
```

`onModRemoved(DRONE_MOD_ID)` while the selected aircraft is a drone calls `session.onAircraftSwitch({ aircraftId: "cessna-172", lodId: "auto" })` after unregister. Unregister happens first so the gallery drops the family, then the switch boots the Cessna. The switch path sees a non-drone id and restores `droneRateRestore`.

---

## 4. Phase 1 — vehicles, JSBSim, gimbal, controller

All files in this phase except the host seams in section 3 live under `fly/src/flight/drone/` and are imported only from `droneFeature.ts`. The rotor plant and flight controller are the sibling package [`multirotor/`](../../multirotor/README.md) (`@0sfs/multirotor`), which only that lazy chunk imports.

### 4.1 Sensor and vehicle constants

`fly/src/flight/drone/droneTypes.ts`

Diagonal field of view converts to horizontal and vertical by the tangent form. Pixel aspect is the sensor aspect.

```typescript
export function fovFromDiagonal(widthPx: number, heightPx: number, diagonalDeg: number): { hfovRad: number; vfovRad: number } {
  const diagonal = Math.hypot(widthPx, heightPx);
  const tanHalf = Math.tan(diagonalDeg * Math.PI / 360);
  return {
    hfovRad: 2 * Math.atan((widthPx / diagonal) * tanHalf),
    vfovRad: 2 * Math.atan((heightPx / diagonal) * tanHalf),
  };
}
```

Locked outputs, which a unit test recomputes from the function rather than hard-coding the grid only:

| | Mini 2 | Air 2S |
| --- | --- | --- |
| id | `dji-mini-2` | `dji-air-2s` |
| massKg | 0.249 | 0.595 |
| sensorPx | 4000 × 3000 | 5472 × 3648 |
| diagonal FOV | 83° | 88° |
| HFOV | 70.58020933259331° | 77.56386793045911° |
| VFOV | 55.92186837389926° | 56.3530369286837° |
| gimbal pitch | −90° … +20° | −90° … +24° |
| Normal: speed, tilt, yaw rate, climb, descent | 10 m/s, 25°, 60°/s, 3 m/s, 3 m/s | 15 m/s, 25°, 90°/s, 6 m/s, 6 m/s |
| Sport: speed, tilt, yaw rate, climb, descent | 16 m/s, 40°, 130°/s, 5 m/s, 3.5 m/s | 19 m/s, 35°, 250°/s, 6 m/s, 6 m/s |
| propeller | 4726F: radius 0.0597 m, 2 blades, pitch 0.06604 m | 7238F: radius 0.0917 m, 2 blades, pitch 0.09652 m |
| front hubs (x aft, y right, z up), m | (−0.0648, ±0.0877, 0.027) | (−0.0871, ±0.1278, 0.0348) |
| rear hubs, m | (0.0604, ±0.0844, 0.005) | (0.093, ±0.1135, 0.0135) |
| spin (+1 clockwise from above) | front-left and rear-right +1; front-right and rear-left −1 | same |
| CG, m | (−0.0022, 0, −0.005) | (0.003, 0, −0.005) |
| empty weight | 0.548951 lb | 1.311750 lb |
| Ixx / Iyy / Izz, slug·ft² | 2.360e-4 / 3.202e-4 / 5.113e-4 | 1.2118e-3 / 1.6566e-3 / 2.6545e-3 |
| Ixx / Iyy / Izz, kg·m² | 3.200e-4 / 4.341e-4 / 6.932e-4 | 1.643e-3 / 2.246e-3 / 3.599e-3 |
| body drag area x / y / z, m² | 0.005 / 0.006 / 0.013 | 0.0085 / 0.010 / 0.022 |

The mode rows are DJI's published limits. The flight controller enforces them: speed, tilt and the climb and descent rates through its velocity loop and tilt limit, and yaw rate through its yaw loop. Cine mode is not modelled.

The hubs, spins, CG, propeller and controller gains live in each aircraft manifest's `quad` section: `quad.modes`, `quad.propeller`, `quad.rotors`, `quad.cgM` and `quad.control`. Positions use JSBSim structural axes, metres from the XML origin. The hubs were measured from the HD meshes. The propellers' radius and pitch come from DJI's propeller names. Chord, section data, rotor inertia, the speed time constant and the speed limits are estimates.

Inertia is a component estimate: motors, arms, body and battery, with the body longer than it is wide. It replaces the earlier model of four point masses at the motors, which was about three times too high. The drag areas are bounded by each mode's published top speed at or under its tilt limit.

The manifest data looks like this, abridged for the Mini 2:

```json
"quad": {
  "cameraId": "0sfs/dji-fc7303",
  "gimbalMinDeg": -90, "gimbalMaxDeg": 20,
  "modes": {
    "normal": { "speedMps": 10, "tiltMaxDeg": 25, "yawRateDeg": 60, "climbMps": 3, "descentMps": 3 },
    "sport": { "speedMps": 16, "tiltMaxDeg": 40, "yawRateDeg": 130, "climbMps": 5, "descentMps": 3.5 }
  },
  "propeller": {
    "radiusM": 0.0597, "blades": 2, "chordM": 0.011, "pitchM": 0.06604, "zeroLiftDeg": -1.5,
    "liftSlope": 5.5, "profileDrag": 0.035, "inertiaKgM2": 8e-7, "timeConstantS": 0.025,
    "maxRpm": 13300, "idleRpm": 3100
  },
  "rotors": [
    { "position": "front-left", "hubM": [-0.0648, -0.0877, 0.027], "spin": 1 },
    { "position": "front-right", "hubM": [-0.0648, 0.0877, 0.027], "spin": -1 },
    { "position": "rear-left", "hubM": [0.0604, -0.0844, 0.005], "spin": -1 },
    { "position": "rear-right", "hubM": [0.0604, 0.0844, 0.005], "spin": 1 }
  ],
  "cgM": [-0.0022, 0, -0.005],
  "control": {
    "kv": 2.5, "kvi": 2, "viLimitH": 8, "viLimitV": 4, "shapeGain": 2, "accelMaxH": 6, "accelMaxV": 4,
    "jerkMax": 30, "kAtt": 7, "rateMaxDeg": 250, "kPsi": 3, "yawLeadDeg": 30, "kRate": 18, "kRi": 5, "riLimit": 10
  }
}
```

The shared `Quadcopter` manifest holds what every quadcopter shares:

```json
"quadcopter": {
  "kinematic": "drone-pgps",
  "gains": { "kPos": 0.6, "deadzone": 0.03 },
  "gridOverlap": 0.7,
  "physicsHz": 120,
  "gimbalSlewDeg": 180,
  "positionHoldMps": 4.8,
  "properties": {
    "fx": "multirotor/force-x-lbs", "fy": "multirotor/force-y-lbs", "fz": "multirotor/force-z-lbs",
    "l": "multirotor/moment-l-ftlbf", "m": "multirotor/moment-m-ftlbf", "n": "multirotor/moment-n-ftlbf",
    "armed": "multirotor/armed"
  }
}
```

Sensor plate for EXIF, millimetres, forced to the pixel aspect:

- Mini 2: width 6.17, height `6.17 * 3/4` = 4.6275. Focal length `f = (diag/2) / tan(83°/2)` = 4.35869772740751 mm. 35 mm equivalent `36/6.17 * f` = 25.431623693139443 mm.
- Air 2S: 13.2 × 8.8. `f` = 8.214046816072269 mm. 35 mm equivalent = 22.401945862015282 mm.

EXIF model names: Mini 2 `FC7303`, Air 2S `FC3411`. Make `DJI`.

Camera optical center in the sim body frame (+X left, +Y up, +Z nose), metres, relative to the CG: `(0, -0.02, 0.05)`. That is 2 cm below the CG and 5 cm forward.

Babylon `camera.fov` is the vertical FOV in radians. Set it to `vfovRad`. Set `camera.fovMode` to `Camera.FOVMODE_VERTICAL_FIXED`.

### 4.2 FDM profile JSON

Both profiles share this shape. `model` is the JSBSim file stem (`dji-mini-2`, `dji-air-2s`).

```json
{
  "model": "dji-mini-2",
  "dataSource": { "kind": "mod", "modId": "0sfs/dji-mini2" },
  "kinematic": "drone-pgps",
  "requiresPhysicsHz": 120,
  "engine": "none",
  "rudderSign": 1,
  "stance": { "staticMeters": 0.04, "staticPitchRad": 0, "pitchArmMeters": 0.08, "rollArmMeters": 0.08 },
  "gauges": { "primary": "multirotor/armed", "secondary": null, "label": "ARM" },
  "startSpeed": { "property": "multirotor/armed", "runningAt": 1 },
  "initialThrottleNorm": 0.5,
  "initialAirspeedKts": 0,
  "initialPitchDeg": 0,
  "initialGearDown": true,
  "flapPosition": { "property": "fcs/flap-pos-norm", "fullTravel": 1 },
  "automaticFlaps": { "kind": "native", "commandProperty": "fcs/flaps-auto-enabled" },
  "forceControlSurfaces": [],
  "runwayPresets": {
    "departure": { "airspeedKts": 0, "throttleNorm": 0.5, "flapsNorm": 0, "pitchDeg": 0 },
    "arrival": { "airspeedKts": 0, "throttleNorm": 0.5, "flapsNorm": 0, "pitchDeg": 0 }
  },
  "initialProperties": {
    "multirotor/force-x-lbs": 0,
    "multirotor/force-y-lbs": 0,
    "multirotor/force-z-lbs": 0,
    "multirotor/moment-l-ftlbf": 0,
    "multirotor/moment-m-ftlbf": 0,
    "multirotor/moment-n-ftlbf": 0,
    "multirotor/armed": 0,
    "fcs/flap-pos-norm": 0,
    "fcs/flaps-auto-enabled": 0
  }
}
```

`engine: "none"` stays: there is no JSBSim propulsion, and the rotors' speeds go to the quad engine monitor instead of gauges. The ARM gauge reads `multirotor/armed`, which the runtime writes every step (section 4.4). `fcs/flaps-auto-enabled` and `fcs/flap-pos-norm` only satisfy the host's flap seams.

`bodyCollisionProbes` stays omitted. `registerFdmProfile` fills `[]`. Do not edit `ground-contact`.

Aircraft definitions:

```json
{
  "id": "dji-mini-2",
  "familyId": "dji-quadcopter",
  "label": "DJI Mini 2",
  "summary": "249 g camera drone. 4:3 stills, gimbal −90° to +20°.",
  "modelYawRad": 0,
  "modelOffset": { "x": 0, "y": 0, "z": 0 },
  "cockpitOffset": { "x": 0, "y": -0.02, "z": 0.05 },
  "procedural": "drone",
  "propellerBlades": 0,
  "lods": [{
    "id": "lod0",
    "label": "Procedural",
    "triangles": 12,
    "path": "",
    "autoFromMeters": 0,
    "credit": { "artist": "0SFS", "note": "Boxes parented by the drone feature after it loads." }
  }]
}
```

Air 2S is the same with its id, label, summary (`595 g`, `3:2`, gimbal to `+24°`). `path` stays `""` because `procedural: "drone"` skips the fetch.

### 4.3 JSBSim XML

Both files use this document. Substitute the row in the table. Units that JSBSim accepts on the element are written on the element so the agent does not convert again.

Constants used by both:

- `N_PER_LBF = 4.4482216152605`
- `NM_PER_FTLBF = 1.3558179483314004`
- The runtime converts SI to these properties. The XML only names them.

```xml
<?xml version="1.0"?>
<fdm_config name="DRONE_ID" version="2.0" release="BETA">
  <fileheader>
    <author>0SFS</author>
    <filecreationdate>2026-10-27</filecreationdate>
    <description>rotor force and moment come from @0sfs/multirotor</description>
  </fileheader>
  <metrics>
    <wingarea unit="FT2">1</wingarea>
    <wingspan unit="FT">1</wingspan>
    <chord unit="FT">1</chord>
    <htailarea unit="FT2">0.1</htailarea>
    <htailarm unit="FT">0</htailarm>
    <vtailarea unit="FT2">0.1</vtailarea>
    <vtailarm unit="FT">0</vtailarm>
    <location name="AERORP" unit="M"><x>CG_X</x><y>0</y><z>-0.005</z></location>
    <location name="EYEPOINT" unit="M"><x>0.05</x><y>0</y><z>0.02</z></location>
    <location name="VRP" unit="M"><x>0</x><y>0</y><z>0</z></location>
  </metrics>
  <mass_balance>
    <ixx unit="SLUG*FT2">IXX_SLUG</ixx>
    <iyy unit="SLUG*FT2">IYY_SLUG</iyy>
    <izz unit="SLUG*FT2">IZZ_SLUG</izz>
    <emptywt unit="LBS">EMPTY_LBS</emptywt>
    <location name="CG" unit="M"><x>CG_X</x><y>0</y><z>-0.005</z></location>
  </mass_balance>
  <ground_reactions>
    <!-- One contact per foot, in this order: FL_FOOT, FR_FOOT, RL_FOOT, RR_FOOT. -->
    <contact type="STRUCTURE" name="FL_FOOT">
      <location unit="M"><x>FRONT_X</x><y>-FRONT_Y</y><z>FRONT_Z</z></location>
      <static_friction>0.8</static_friction>
      <dynamic_friction>0.6</dynamic_friction>
      <spring_coeff unit="LBS/FT">FRONT_SPRING</spring_coeff>
      <damping_coeff unit="LBS/FT/SEC">DAMPING</damping_coeff>
      <damping_coeff_rebound unit="LBS/FT/SEC">REBOUND</damping_coeff_rebound>
      <max_steer unit="DEG">0</max_steer>
      <brake_group>NONE</brake_group>
      <retractable>0</retractable>
    </contact>
    <!-- FR_FOOT at (FRONT_X, FRONT_Y, FRONT_Z). RL_FOOT at (REAR_X, -REAR_Y, REAR_Z) and
         RR_FOOT at (REAR_X, REAR_Y, REAR_Z), with REAR_SPRING. Otherwise identical. -->
  </ground_reactions>
  <external_reactions>
    <force name="multirotor-force-x" frame="BODY">
      <function><property>multirotor/force-x-lbs</property></function>
      <direction><x>1</x><y>0</y><z>0</z></direction>
      <location unit="M"><x>CG_X</x><y>0</y><z>-0.005</z></location>
    </force>
    <force name="multirotor-force-y" frame="BODY">
      <function><property>multirotor/force-y-lbs</property></function>
      <direction><x>0</x><y>1</y><z>0</z></direction>
      <location unit="M"><x>CG_X</x><y>0</y><z>-0.005</z></location>
    </force>
    <force name="multirotor-force-z" frame="BODY">
      <function><property>multirotor/force-z-lbs</property></function>
      <direction><x>0</x><y>0</y><z>1</z></direction>
      <location unit="M"><x>CG_X</x><y>0</y><z>-0.005</z></location>
    </force>
    <moment name="multirotor-moment-l" frame="BODY">
      <function><property>multirotor/moment-l-ftlbf</property></function>
      <direction><x>1</x><y>0</y><z>0</z></direction>
    </moment>
    <moment name="multirotor-moment-m" frame="BODY">
      <function><property>multirotor/moment-m-ftlbf</property></function>
      <direction><x>0</x><y>1</y><z>0</z></direction>
    </moment>
    <moment name="multirotor-moment-n" frame="BODY">
      <function><property>multirotor/moment-n-ftlbf</property></function>
      <direction><x>0</x><y>0</y><z>1</z></direction>
    </moment>
  </external_reactions>
  <aerodynamics>
    <axis name="X"><function name="aero/force/X"><product><property>atmosphere/rho-slugs_ft3</property><property>velocities/vt-fps</property><property>velocities/u-aero-fps</property><value>DRAG_X</value></product></function></axis>
    <axis name="Y"><function name="aero/force/Y"><product><property>atmosphere/rho-slugs_ft3</property><property>velocities/vt-fps</property><property>velocities/v-aero-fps</property><value>DRAG_Y</value></product></function></axis>
    <axis name="Z"><function name="aero/force/Z"><product><property>atmosphere/rho-slugs_ft3</property><property>velocities/vt-fps</property><property>velocities/w-aero-fps</property><value>DRAG_Z</value></product></function></axis>
    <axis name="ROLL"><function name="aero/moment/Roll"><product><property>aero/qbar-psf</property><property>metrics/Sw-sqft</property><property>metrics/bw-ft</property><value>0</value></product></function></axis>
    <axis name="PITCH"><function name="aero/moment/Pitch"><product><property>aero/qbar-psf</property><property>metrics/Sw-sqft</property><property>metrics/cbarw-ft</property><value>0</value></product></function></axis>
    <axis name="YAW"><function name="aero/moment/Yaw"><product><property>aero/qbar-psf</property><property>metrics/Sw-sqft</property><property>metrics/bw-ft</property><value>0</value></product></function></axis>
  </aerodynamics>
</fdm_config>
```

| Token | Mini 2 | Air 2S |
| --- | --- | --- |
| DRONE_ID | dji-mini-2 | dji-air-2s |
| CG_X | -0.0022 | 0.003 |
| EMPTY_LBS | 0.548951 | 1.311750 |
| IXX_SLUG / IYY_SLUG / IZZ_SLUG | 0.0002360 / 0.0003202 / 0.0005113 | 0.0012118 / 0.0016566 / 0.0026545 |
| FRONT_X / FRONT_Y / FRONT_Z | -0.0701 / 0.0954 / -0.0282 | -0.0876 / 0.1279 / -0.0397 |
| REAR_X / REAR_Y / REAR_Z | 0.0394 / 0.0157 / -0.0235 | 0.0294 / 0.0153 / -0.0334 |
| FRONT_SPRING, REAR_SPRING, DAMPING, REBOUND | See the XML | See the XML |
| DRAG_X / DRAG_Y / DRAG_Z | -0.02690975 / -0.03229170 / -0.06996535 | -0.045746575 / -0.05381950 / -0.11840290 |

Axes and signs:

- **Locations** are JSBSim structural axes: +X aft, +Y right, +Z up, metres from the XML origin. The CG and AERORP coincide, so the body drag makes no moment.
- **The rotor wrench** is BODY: forces along body x forward, y right, z down, and moments l, m, n about those axes. Each force is applied at the CG, so it adds no moment of its own. The plant has already summed each rotor's r × F about the CG into l, m, n. A negative `force-z` is upward thrust.
- **Body drag** is on JSBSim's body X, Y and Z aerodynamic axes, which carry no sign flip. Each value is −½ × that axis's drag area in ft², so the force is −½ ρ V CdA × the air-relative body velocity on that axis. The force opposes the air-relative velocity, and the weather wind enters through `velocities/{u,v,w}-aero-fps`. The aerodynamic moments are zero.
- **Feet** are four STRUCTURE contacts under the arms. The runtime reads ground contact from `contact/unit[0..3]/WOW`; `gear/wow` counts only BOGEY contacts. The spring and damping coefficients are tuned against the settling and landing tests in `fly/src/flight/drone/quadJsbsimLab.test.ts`. The XML, not this table, is their source. The rear springs are stiffer, so both aircraft rest level with the CG about 10 mm above the ground.
  - The settling test passes: the aircraft stays still on four feet for ten seconds.
  - The landing tests pass (28 October) with section 4.4's flare, idle on contact and one second of grounded descent stick to disarm. They cover a 1 m/s approach, full Normal and Sport descent from 40 m, and landing after climb, hover and forward flight through the shipped controller. Touchdown is about 0.06 m/s; all runs stay below 1.63 g and 0.72 mm rebound, keep the CG above ground, and finish level on all four feet with the motors disarmed. The acceptance limits remain 3 g and 5 mm.
- **EYEPOINT** is in the same structural axes, so (0.05, 0, 0.02) is 5 cm aft and 2 cm up. Nothing reads it; the camera's optical center is set in section 4.1.

There is no `<propulsion>` element. The rotors are not JSBSim engines: `@0sfs/multirotor` computes them, and the runtime writes their sum to the wrench properties every step.

### 4.4 Controller

`fly/src/flight/drone/droneController.ts` (commands, holds, arming), `fly/src/flight/drone/droneFlightModel.ts` (JSBSim I/O) and `@0sfs/multirotor` (flight controller and rotor plant). Revised 28 October: the earlier velocity-P, tilt-clip and moment law, with its world-axis force, is gone.

Runs from `beforeStep` at `dtSeconds === 1/120`. Assert `Math.abs(dtSeconds - 1/120) < 1e-9`. If the flight was not switched onto 120 Hz, throw once, then skip each wrong-rate step; the wrench properties keep their last values.

Each step, before `sdk.run()`, `droneFlightModel` reads one property batch:

| Property | Scale |
| --- | --- |
| `velocities/v-north-fps`, `v-east-fps`, `v-down-fps` | × 0.3048 → m/s, ground-relative NED, for the velocity loop |
| `velocities/u-aero-fps`, `v-aero-fps`, `w-aero-fps` | × 0.3048 → m/s, air-relative body axes, for the rotor plant |
| `velocities/p-rad_sec`, `q-rad_sec`, `r-rad_sec` | rad/s |
| `attitude/phi-rad`, `theta-rad`, `psi-rad` | radians |
| `inertia/mass-slugs` | × 14.5939029 → kg |
| `inertia/ixx-slugs_ft2`, `iyy-slugs_ft2`, `izz-slugs_ft2` | × 1.3558179483314004 → kg·m² |
| `atmosphere/rho-slugs_ft3` | × 515.3788184 → kg/m³ |
| `contact/unit[0]/WOW` … `contact/unit[3]/WOW` | on the ground when any foot is 1 |
| `simulation/sim-time-sec` | s |
| `position/h-sl-ft` | × 0.3048 → m |
| `position/lat-geod-deg`, `position/long-gc-deg` | degrees |

AGL is `altM - terrainMeters`, from the host's terrain sample; NaN when terrain is unknown.

A sample with any non-finite value, or a non-positive mass, inertia or density, writes a zero wrench and skips the step. If simulation time goes backwards or jumps by more than 1.5 steps, the plant, controller, holds and arming are reset.

It writes one batch, created if missing:

| Property | Value |
| --- | --- |
| `multirotor/force-x-lbs`, `force-y-lbs`, `force-z-lbs` | Rotor force on body x forward, y right, z down, at the CG. N ÷ 4.4482216152605 |
| `multirotor/moment-l-ftlbf`, `moment-m-ftlbf`, `moment-n-ftlbf` | Rotor moment about the CG on the same axes. N·m ÷ 1.3558179483314004 |
| `multirotor/armed` | 1 when armed, otherwise 0 |

The names come from the shared manifest's `properties`. The XML in section 4.3 applies them.

`G = 9.80665`.

Stick deadzone is the shared manifest's `gains.deadzone`, 0.03, the same threshold as `isCentered` in `rc/src/protocol.ts`.

```typescript
const dead = (value: number) => Math.abs(value) < 0.03 ? 0 : value;
```

Commands, body level frame, then NED. `ψ` is JSBSim heading, 0 at north, positive clockwise toward east. `vXyMax`, `climbMax` and `descentMax` are the selected mode's `speedMps`, `climbMps` and `descentMps` (Sport when the command's `sport` flag is set, otherwise Normal).

```text
vFwd   = dead(pitch) * vXyMax
vRight = dead(roll)  * vXyMax
vUp    = dead(vertical) >= 0 ? dead(vertical) * climbMax : dead(vertical) * descentMax
vN     =  vFwd * cos(ψ) - vRight * sin(ψ)
vE     =  vFwd * sin(ψ) + vRight * cos(ψ)
vD     = -vUp
```

`vertical` is −1…+1 with 0 = hover. Positive is climb. See section 6.3 for the centered desktop climb-rate mapping.

Horizontal and vertical holds acquire their anchors independently. Moving either pitch or roll clears the horizontal anchor; moving vertical clears only the altitude anchor. When a stick group returns inside the deadzone, command zero velocity for that group while it brakes. Capture the current lat/lon once ground-relative horizontal speed `hypot(vn, ve)` is at most 0.1 m/s, and capture the current altitude once `abs(vd)` is at most 0.1 m/s. Each group can capture while the other stick group remains active. This holds the stopping point rather than returning to the position where the sticks were released. Keep horizontal anchors in degrees and altitude in metres; convert the error each step after capture:

```text
northErrorM = (latDeg - latAnchor) * 111320
eastErrorM  = wrapDeg(lonDeg - lonAnchor) * cos(latAnchor * π/180) * 111320
vN = clamp(-K_pos * northErrorM, -holdMax, holdMax)    when centered with a horizontal anchor
vE = clamp(-K_pos * eastErrorM,  -holdMax, holdMax)
vD = clamp(K_pos * (altM - altAnchor), -climbMax, descentMax)   when centered with an altitude anchor
```

The signs pull toward the anchor. The opposite signs are positive feedback and must not be restored. `K_pos` is the shared manifest's `gains.kPos`, 0.6 s⁻¹, and `holdMax` is its `positionHoldMps`, 4.8 m/s. A captured anchor stays fixed even if wind pushes the aircraft above the capture-speed threshold. Only new input from its own stick group, a reset or an aircraft change invalidates it; the other group's anchor keeps holding.

Landing slowdown, using `terrainMeters` from the host (null means unknown):

```text
agl = altM - terrainMeters
touchdownAgl = groundContactClearanceMeters(sdk, 0, 0)
if vD > 0 and agl is finite: vD = min(vD, max(0.1, (agl - touchdownAgl) * 0.5))
```

`touchdownAgl` is the loaded airframe's level CG height above its lowest foot, read once on SDK binding from the JSBSim contact geometry: 0.0232 m for the Mini 2 and 0.0347 m for the Air 2S. The 0.5 s⁻¹ slope starts slowing a 6 m/s descent about 12 m above the feet, allowing for the controller's acceleration, jerk and rotor response limits. A steeper 1.5 s⁻¹ slope passed the 1 m/s test but started slowing 6 m/s only 4 m up and failed full-descent landings. The flare approaches a 0.1 m/s command near the feet; measured touchdown speeds are about 0.06 m/s in both aircraft. Contact selects idle, and holding the descent stick for one second disarms. With unknown terrain the mode's descent limit applies. The touchdown speed remains a feel setting for user testing.

Yaw maps to the multirotor's yaw command:

- `yawMode === "angularVelocity"`: rate mode, `yaw = dead(yaw) * yawRateMax`.
- `yawMode === "angle"`: heading mode, `yaw = yawAngleDeg * π/180`, or the current `ψ` when `yawAngleDeg` is null.

`yawRateMax` is the mode's `yawRateDeg` in rad/s, and also limits the heading mode's turn rate. The tilt limit passed on is `tan(tiltMaxDeg)`.


Arming and rotor mode:

- On the first valid sample, an aircraft on the ground starts disarmed and one in the air starts armed, with every rotor already at hover speed for the live mass and density.
- On the ground, `dead(vertical) > 0.5` arms. `dead(vertical) < -0.5` held for 1 s disarms; releasing it or lifting off restarts the timer. In the air the aircraft never disarms.
- The rotor mode passed to the controller is STOPPED when disarmed, IDLE when on the ground with `dead(vertical) ≤ 0`, and FLYING otherwise. STOPPED commands zero rotor speed, IDLE the propeller's `idleRpm`.

The command `{ vN, vE, vD, yawMode, yaw, mode, tanTiltMax, yawRateMax }` then goes to `multirotor.step(sample, command)`. In one step, `@0sfs/multirotor`:

1. **Velocity loop.** It shapes the velocity target with acceleration and jerk limits, and runs PI on the velocity error. The result is the specific force the rotors must make.
2. **Attitude.** It limits that force to the tilt limit, tilting the thrust axis first, then turns the heading. The heading turn uses the commanded rate, or the heading error within `yawRateMax`. The attitude error comes from the rotation matrix.
3. **Rates.** A PI loop on body rates, with the gyroscopic term and the live inertia, gives the moments the rotors must make.
4. **Allocation.** The minimum-norm inverse of the rotor layout about the CG turns thrust and moments into four rotor thrusts. Thrust and tilt come first, and yaw gives way.
5. **Speed commands.** Static thrust ∝ ω², scaled by the measured thrust efficiency, turns those thrusts into four rotor-speed commands.
6. **Rotor plant.** For each rotor:
   - the speed follows its command through a first-order lag;
   - blade element and momentum theory gives thrust, H-force, torque and hub moment, from the hub's air-relative velocity, density and AGL;
   - the inflow covers the vortex ring state, with buffet, and the windmill brake state;
   - ground effect is applied.

   It sums the rotors, with r × F, reaction torque and the gyroscopic moment, into one force and moment about the CG.
7. **Thrust efficiency.** The plant's total thrust over the static thrust becomes next step's efficiency, filtered with a 0.1 s time constant and clamped to 0.3–3. That is what an ideal accelerometer would give.

The wrench is computed from the pre-step state and applied by JSBSim for the coming step. The gains are each aircraft's `quad.control`. Module-level detail is in [multirotor/README.md](../../multirotor/README.md).

WASM: `droneFlightModel` starts `loadMultirotor()` when it is created, and the TypeScript kernels fly until the WASM loads. Then `useCore` moves the buffers into WASM memory. The two are identical bit for bit, so the hand-over does not change the trajectory. If loading fails, `refuseCore` records why. If a WASM kernel throws on anything but bad input, the runtime copies its state back and stays on TypeScript for good. `status()` reports the backend and the reason. Rotor speeds from the plant go to the props and the quad engine monitor (section 4.7 and the engine widget follow-up).

Gimbal pitch state is not a JSBSim property. The command layer slews it at the shared manifest's `gimbalSlewDeg`, 180°/s, toward `command.gimbalPitchDeg`, clamped to the vehicle's gimbal range. `isSettled` for the gimbal is `|error| < 0.15°` and `|slew rate| < blurLimit` from section 7.

### 4.5 Wind

Revised 28 October: there is no drone-specific wind. `droneWind.ts`, its Ornstein–Uhlenbeck process and the controller's drag law are deleted.

- **Wind.** The host writes the weather wind into JSBSim's atmosphere (`atmosphere/wind-north-fps`, `wind-east-fps`), as it does for every aircraft. JSBSim then gives the air-relative body velocities `velocities/{u,v,w}-aero-fps`.
- **Body drag.** The XML's X, Y and Z axes apply body drag from those velocities, with one drag area per axis (section 4.3).
- **Rotors.** The rotor plant reads the same air-relative velocities, so the rotors' H-force, translational lift and vortex-ring state all see the wind.
- **The controller** sees wind only as velocity error, and leans into it as a real one does. In the real-JSBSim lab, in a 10 m/s wind, the Mini 2 leans about 11° and the Air 2S about 8°. Both hold position to within 1 cm/s.

Turbulence is not wired.

### 4.6 Gimbal hierarchy

`fly/src/flight/drone/droneGimbal.ts`, called from `afterView` after `floatingOrigin.apply`, which has already set `aircraftRoot.rotationQuaternion` to `flightAttitudeToQuaternion(roll, pitch, heading)` (`ground-contact/src/ecefBridge.ts`).

The aircraft entity root is already parented to `floatingOrigin.aircraftRoot` by `createPlaceholderAircraft`. Build this once, parented to `entity.root` (`aircraft-visual`):

```text
aircraft-visual                          full airframe attitude
  drone-gimbal-base                      fixed mount; position (0, -0.02, 0.05)
    drone-gimbal-yaw                     Y joint
      drone-gimbal-roll                  Z joint
        drone-gimbal-pitch               X joint; camera housing/lens only
          drone-gimbal-camera            UniversalCamera, fov = vfovRad
```

`createPlaceholderAircraft` does not know about this. The feature parents the sensor base to `aircraft-visual`. It adopts the existing first-person camera onto the sensor pitch node and restores its parent, position and rotation on view change or disposal. Chase follow remains independent.

Body attitude is `RotY(π - ψ) * RotX(-θ) * RotZ(φ)`. Stabilization cancels roll/pitch in the reverse order, `RotZ(-φ) * RotX(θ)`. The desired local sensor pose, including manual offsets, is:

```text
Q_local = RotZ(-φ) * RotX(θ) * RotY(-yawOffset) * RotX(-gimbalPitch) * RotZ(rollOffset)
```

Do not reverse the two body-cancellation factors. Positive pitch looks up; positive yaw looks clockwise from aircraft heading; positive roll banks about the optical axis. `camera-gimbal` decomposes the desired pose into the physical yaw(Y) → roll(Z) → pitch(X) joints, leaving the mounting base at identity. The same function poses the sensor rig and the HD model pivots. The older two-node API remains supported for assets without separate joint stages.

The Camera tab has Pitch, Roll and Yaw sliders with target/actual readouts and 5° nudges; Roll/Yaw have zero resets. Roll/Yaw are simulator manual offsets, not a claim that the stock remote provides those controls. Their airframe data uses the manufacturer's mechanical ranges: Mini 2 roll ±35°, yaw ±20°; Air 2S roll ±45°, yaw ±100° ([Mini 2 manual](https://dl.djicdn.com/downloads/DJI_Mini_2/DJI_Mini_2_User_Manual_EN.pdf), [Air 2S manual](https://dl.djicdn.com/downloads/DJI_Air_2S/DJI_Air_2S_User_Manual_v1.0_enIV.pdf)). These bounds apply to the requested stabilized offsets; body compensation is composed by the shared rig. Pitch retains the existing −90° to +20°/+24° controllable ranges. Overrides slew on physics time; paused controls queue a target and the tab explains that resuming moves the gimbal.

A 360 temporarily slews Roll/Yaw to zero and locks manual controls, then restores the pilot targets after stopping. Their motion contributes to the sequence's settling rate gate. A paused aircraft switch initializes its new controller/mount limits before posing, so the previous aircraft's wider range cannot leak into the new camera.

At −90° pitch the sensor looks down. The camera itself has one `RotationYawPitchRoll(π, 0, 0)` at creation; all pose changes live on its parents. `minZ = 0.02`, `maxZ = 250000`.

### 4.7 Airframe models and camera

Follow-up on 28 October: Mini 2's checkout and mod source are `DJI-Mini2` / `0SFS/DJI-Mini2`. Its internal aircraft and JSBSim ID remains `dji-mini-2`. The dev server retains the old `/dev-mods/dji-mini-2/` alias for saved sources.

Both aircraft have a local HD level by yigeshichen: [Mini 2](https://sketchfab.com/3d-models/dji-mini-2-85678e6bf8a74829984193fed2d025b5) and [Air 2S](https://sketchfab.com/3d-models/dji-air-2s-e29628f7c3b64c1daf238fd0a807ad93). Source files live under each checkout's `3d_model/HD/`; `npm run prepare:hd` preserves the source and writes a centered runtime GLB in metres under `public/3d_model/HD/`. These CC BY-NC 4.0 local reference files stay ignored by Git. HD attribution and its source link belong to the level's existing credit UI. Mini 2 has 109,470 triangles; Air 2S has 142,400. Auto uses HD below 15 m and the procedural level at or above 15 m. An explicit LOD choice is also supported.

`AircraftDefinition.camera` supplies camera defaults through the existing settings registry, with pilot overrides retained. Mini 2 starts 0.8 m behind and 0.24 m above its origin, with a 0.25 m minimum distance. Air 2S starts 1.1 m behind and 0.33 m above, with a 0.35 m minimum. Both use a 0.01 m near clip and 500 m maximum distance. Ordinary aircraft restore the normal camera defaults.

**Camera follows**, in Aircraft → Camera:

- **Aircraft** uses the full body attitude, as the original fixed-wing chase camera does.
- **Momentum** uses ground-relative north/east/up velocity. Its heading follows the horizontal course and its pitch follows climb/descent angle. Heading takes the shortest wrapped turn, and heading/pitch ease independently with a 0.25 s time constant before constructing a zero-roll frame. Do not slerp complete direction quaternions: combined turns and climbs can introduce bank. Direction confidence fades smoothly between 1.2 and 0.9 m/s; below 0.9 m/s the target holds while the view continues easing toward it. Heading uses horizontal speed for that confidence, so vertical travel retains the last reliable heading. Travel pitch and the final chase-camera view are limited to ±85°. The chase rig applies orbit yaw about planet up and combines orbit/travel pitch before clamping; tilting an orbit parent can bank sideways views or carry the camera past a pole. Mode entry starts level at the current aircraft heading and eases toward travel instead of snapping into it.
- **Planet** uses the local north/up frame, independent of body attitude or travel direction. User orbit and zoom remain available in every mode.

HD preparation reads `3d_model/HD/rotors.json` and `gimbal.json` to add authored pivots while preserving each mesh's rest-world transform. The downloaded meshes merge camera housings and support frames; prep partitions their disconnected shells by index, retaining vertex attributes, materials and all original triangles. Fixed mounting geometry stays outside the moving stages; yaw arms attach to yaw, roll yokes to roll, and only the camera housing/lens to pitch. The mapping includes expected component triangle counts so a different source cannot be partitioned silently. Aircraft definitions declare rotor node names, axes, radii in metres, blade counts, opposing rotation directions, and visible gimbal base/yaw/roll/pitch node names. Procedural models also have four blade pivots. `@0sfs/rotating-prop` owns frame-interval smoothing, the alias limit (π / blades / frameSeconds) with 0.8 hysteresis, and what is drawn above it. Below the limit the authored 3D blades turn. Above it the blades hide and a **semi-transparent 3D swept propeller volume** is shown by default. That volume is the solid of revolution the blades pass through in one turn, measured from the blade meshes, so it keeps thickness edge-on. Sharp 3D blades are never the high-RPM default. **Aircraft → Flat propeller discs** (`osfs.aircraft.propellerDiscs`, Off by default) is a live opt-in that substitutes a cheaper flat disc. Both looks take their radius from the measured blades. The component integrates only simulated time, so pause freezes rotor phase. The C172 shows its authored swept volume, `Propeller_Disc`, through the same alias-limit implementation; propeller planes without one get the generated sweep.

Rotor speed is simulated. Each pivot turns at its rotor's speed from `@0sfs/multirotor`'s plant, the same speed that makes that rotor's thrust, matched by position (front-left and so on). Its direction is the rotor layout's `spin`. There is no ESC or motor model, so the speed follows the controller's command through a first-order lag. (Revised 28 October; before that it was a visual estimate from a declared 6,000 RPM hover.) Capture and the visible HD camera assembly share `@0sfs/camera-gimbal` stabilization and three-axis pose targets. The component restores an adopted cockpit camera's original parent, position and rotation on view change or disposal.

The coarse procedural level remains:

`buildDroneProceduralMesh.ts` parents to `modelRoot`:

`mountDroneProceduralMesh(modelRoot, { rotors, cgM, propellerRadiusM })` takes the aircraft's `quad.rotors`, `quad.cgM` and `quad.propeller.radiusM`. The flight transform places `modelRoot` at the CG, so each hub is converted from structural axes about the CG into the model frame. The model frame is glTF's: +X left, +Y up, +Z forward, the same frame `structuralPointToDisplay` in `aircraft/src/aircraftForces.ts` uses. The conversion is X = cg.y − y, Y = z − cg.z, Z = cg.x − x. `droneMotorPosition` in `droneMotorTelemetry.ts` classifies pivots in the same frame, so +X is left.

Integration handoff E corrected the mesh and classifier together to use +X left, with tests for both. Each rotor's speed and spin now bind to its physical side. About an upward visual axis the rotation direction is `-spin`: front-left and rear-right turn clockwise when viewed from above.

- Body box `0.16 × 0.05 × 0.12` m (width, height, depth) at the origin. Babylon `CreateBox` size is full extent.
- One arm box `0.02 × 0.012 × |hub|` per rotor, from the origin to the real hub, pitched and yawed along it. A `0.02` m motor box at each hub.
- One pivot `drone-rotor-<i>` per rotor at its hub, holding a two-blade stand-in `0.014 × 0.002 × 2 · propellerRadiusM` m. It uses the same shared rotating-prop presentation as HD. The stand-in blades are flat 2 mm bars, so their generated sweep is correspondingly thin.

`setModelLoaded(true)` so the Cessna placeholder stays hidden. Dispose on feature dispose. Call `requestRender()` once after mount. Do not call it per step; the sim is already running (`setSimRunning`).

### 4.8 Frames and signs

Drone code crosses five frames. On 28 October the drone mesh, the prop classifier and their tests all assumed the model's +X was right, and the drone was drawn mirrored. Every frame here is right-handed. The aircraft force overlay uses the same conventions; see [aircraft-forces.md](../../aircraft/docs/aircraft-forces.md).

| Frame | Axes | Used by |
| --- | --- | --- |
| JSBSim structural | +x aft, +y right, +z up. Metres from the XML origin in manifests; JSBSim reports inches | XML locations, `quad.rotors[].hubM`, `quad.cgM` |
| Body | +x forward, +y right, +z down | JSBSim `u/v/w-aero` and `p/q/r`, the multirotor's sample and wrench, the `multirotor/force-*` and `moment-*` properties |
| NED | north, east, down | velocity commands and position holds |
| Model | +X left, +Y up, +Z forward (glTF's) | everything under `modelRoot`: the procedural mesh, HD pivots, rotating-prop axes |
| World | +X east, +Y up, +Z south | the Babylon scene. FOSS Earth's renderer sets `useRightHandedSystem`, and `flightAttitudeToQuaternion` in `ground-contact/src/ecefBridge.ts` places the aircraft |

Conversions:

- **Structural point to model, about the CG:** X = cg.y − y, Y = z − cg.z, Z = cg.x − x. `structuralPointToDisplay` in `aircraft/src/aircraftForces.ts` does the same in inches.
- **Body vector to model:** X = −y, Y = −z, Z = x. This is a rotation, with determinant +1. A conversion with determinant −1 mirrors the aircraft.
- **Pivot side:** in the model frame +X is left. Classify pivots with `droneMotorPosition(leftM, forwardM)`.
- **Spin:** a rotor's `spin` is its turn about body +Z, which points down, by the right-hand rule. So +1 is clockwise seen from above. rotating-prop turns by `+direction` about its binding's `axis`, also by the right-hand rule. Body +Z is model −Y, so about an upward axis `direction` is `−spin`. Front-left and rear-right have spin +1 and `direction` −1.

Two components that share a wrong assumption agree with each other, so tests that only compare them still pass. Check a new frame mapping against the flight transform or `structuralPointToDisplay`, and look at the result in the browser: the front-left prop should be on the pilot's left, seen from behind.

---

## 5. Phase 2 — phone quad mode

### 5.1 Wire types

`rc/src/drone/droneProtocol.ts` is imported by the lazy phone chunk and by `droneFeature.ts` on the computer (the computer chunk may import `@0sfs/rc/drone/droneProtocol` only from `droneFeature.ts`, so the flight startup graph does not include it).

Add one static check in `rc/src/protocol.ts` `status()` so the phone can see availability without loading the quad UI. Unknown or malformed `drone` is deleted, matching the `camera` pattern. A well-formed availability flag survives:

```typescript
export interface DroneAvailability { available: true }
```

Inside `status()`, after the existing checks: if `value.drone` is present and is not `{ available: true }` and is not a full telemetry object validated by a 15-line local predicate, `delete value.drone`. The predicate allows exactly the fields in `DroneTelemetry` below and no others. Keep the predicate in `protocol.ts` so a hostile payload cannot smuggle extra keys through `status`. It is data checks only. It does not import the quad UI.

```typescript
export interface DroneTelemetry {
  available: true;
  droneId: "dji-mini-2" | "dji-air-2s";
  yawAngleDeg: number;
  gimbalPitchDeg: number;
  gimbalRollDeg: number;
  isSettled: boolean;
  isCapturing: boolean;
  capturePhase: "idle" | "slewing" | "optical-flow-fine-align" | "settling-zero-velocity" | "capturing" | "next-node" | "zip-export" | "done" | "aborted";
  nodeIndex: number;
  nodeCount: number;
  flowConfidence: number;
  flowSource: "visual" | "imu";
  omegaVisualRadPerSec: number;
}
```

Bounds: angles −180…180 except `gimbalPitchDeg` −90…24, `nodeIndex` and `nodeCount` integers 0…256, `flowConfidence` 0…1, `omega` −50…50. Round telemetry to 0.01 before send. If the encoded heartbeat would exceed 1800 bytes, omit `engine.orbs` first. `MAX_MESSAGE_BYTES` stays 2048.

Commands ride on `ControlFrame` as optional `drone`. Malformed `drone` is deleted and the frame still applies, same as `camera`.

```typescript
export interface DroneCommand {
  droneId: "dji-mini-2" | "dji-air-2s";
  pitch: number;
  roll: number;
  vertical: number;
  yaw: number;
  yawMode: "angularVelocity" | "angle";
  yawAngleDeg?: number;
  gimbalPitchDeg: number;
  sport?: boolean;
  shutter?: 1;
  capture360?: boolean;
}
```

`pitch`, `roll`, `yaw` are −1…1. `vertical` is −1…1. `gimbalPitchDeg` is −90…24. `shutter: 1` is an edge; the phone sends it on one frame. `capture360` is level, held while the sequence should run.

`AircraftStatus` gains `drone?: DroneTelemetry`. `ControlFrame` gains `drone?: DroneCommand`.

`createPhoneControlSession.ts` stores the latest valid `DroneCommand` and exposes `getDroneCommand(): DroneCommand | null`. The host reads it inside `currentDroneCommand()`. Telemetry is whatever `getStatus()` already returns; the feature sets it on the status object the host builds for the heartbeat. Do not add a new message type. Heartbeats already carry `status`.

### 5.2 Phone UI, loaded on demand

`rc/src/drone/loadQuadcopter.ts`:

```typescript
export function loadQuadcopter(): Promise<typeof import("./PhoneQuadcopter")> {
  return import("./PhoneQuadcopter");
}
```

`PhoneController.tsx` may import `loadQuadcopter` and nothing from `./drone/PhoneQuadcopter`. When `status?.drone?.available === true` and the module is not loaded, call `loadQuadcopter()` and store the component in state. Until it resolves, keep the fixed-wing controls. When `status.drone` is absent, unmount the quad component and show the fixed-wing controls again. Unmounting does not unload the module; it stops running it.

`PhoneQuadcopter.tsx` replaces the fixed-wing cluster while mounted. Mode 2:

- Left stick: existing `PhoneStick`, labeled `THROTTLE / YAW`. `x` → `drone.yaw` (+ right). `y` → `drone.vertical` with the same sign flip the pitch stick uses (`vertical = -y` if `y` is down-positive on screen; document the prop as `onChange(x, y)` and set `vertical = y` where `y` is already up-positive inside `PhoneStick` — read `PhoneStick.tsx` and match its `elevator = -y` convention so up on the stick is positive `vertical`).
- Right stick: existing `PhoneStick`, labeled `PITCH / ROLL`. `x` → `drone.roll` (+ right). Up on the stick → `drone.pitch` positive (forward).
- Gimbal: `PhoneControlRange` vertical, min −90, max the active drone's max (+20 or +24). Switching Mini 2 clamps a stored +24 down to +20.
- Chip grid: two buttons `Mini 2` and `Air 2S` (send `droneId`; the host calls `onAircraftSwitch`), `Shutter` (sets `shutter: 1` for one frame), `360` toggle (`capture360`), and a tape `360` showing `nodeIndex/nodeCount` plus `capturePhase`.
- While the quad UI is mounted, still fill `controls` with the mapped fixed-wing fields from section 6.3 so an older host that ignores `drone` does not see a zero stick. This host reads `drone` when present.

The quad UI does not render flaps, brake, trim, or the engine widget.

### 5.3 Desktop stick mapping

When no phone `DroneCommand` is present, `currentDroneCommand()` maps `ControlSurfaceState`:

```text
pitch    = elevator          // +1 nose-up stick → forward
roll     = aileron           // +1 right
yaw      = rudder            // +1 yaw right
vertical = inputManager.getClimbRateCommand()  // centered −1…+1; independent of throttle
yawMode  = "angularVelocity"
gimbalPitchDeg = gimbal held by the feature, unchanged by this mapping
sport    = false
shutter  = false
capture360 = capture held by the feature
```

Keyboard W/S/Q/E/A/D and Shift/Ctrl fly the drone through the configurable input manager. The spring-centered CLIMB lever shows requested climb/descent in m/s. The host homes gimbal and capture actions in the **Camera** tab, available through the existing tab picker only while a drone is applied. It contains a bounded pitch slider, 5° down/up nudges, target/actual angles, Take still, Start/Stop 360, progress and visible errors. Nothing is appended to the flight toolbar. `drone-camera/cameraControls` owns the reusable DOM UI and lazy CSS; the lazy flight feature supplies commands and telemetry, and the startup panel receives only an optional element through `droneGate`. Closing the tab detaches that element without stopping or resetting a sequence. The feature destroys it on disposal. A paused flight permits a still and a queued manual gimbal target, but starting a 360 is disabled until resume; an active paused sequence shows Continue guidance and retains Stop. Stop passes the raw released capture level to the sequence even while its motion-command override is active, and also suppresses a phone-held sequence until that toggle is released. Phone controls remain available as before.

A phone `droneId` that differs from the current aircraft calls `onAircraftSwitch` once per change, which is the path that loads the feature if it is not loaded. The feature is already loaded if a drone is selected. Switching Mini 2 ↔ Air 2S goes through `switchAircraft` and therefore reboots JSBSim at 120 Hz. Do not hot-swap mass inside one SDK instance.

---

## 6. Phase 3 — offscreen capture, EXIF, zip

`drone-camera/src/captureFrame.ts`, imported only from the lazy chunk.

### 6.1 Refuse without WebGPU

```typescript
if (runtime.renderer.mode !== "webgpu") throw new Error("360° capture requires WebGPU.");
const device = (runtime.renderer.engine as WebGPUEngine)._device;
if (!device) throw new Error("WebGPU device is missing.");
```

`runtime.renderer` is the `RendererSelection` on the Babylon runtime (`createFlightSimApp.ts` already narrows it this way around the `gpuDevice:` lines).

### 6.2 Resolution

```typescript
export function captureSize(width: number, height: number, maxTexture: number, maxBuffer: number): { width: number; height: number; scale: number } {
  let scale = 1;
  const fits = (s: number) => {
    const w = Math.round(width / s);
    const h = Math.round(height / s);
    const bytesPerRow = Math.ceil(w * 4 / 256) * 256;
    return w <= maxTexture && h <= maxTexture && w >= 2 && h >= 2 && bytesPerRow * h <= maxBuffer;
  };
  while (!fits(scale)) scale += 1;
  return { width: Math.round(width / scale), height: Math.round(height / scale), scale };
}
```

`maxTexture = device.limits.maxTextureDimension2D`. `maxBuffer = device.limits.maxBufferSize`. Record `scale` in the XMP field `drone-dji:CaptureScale`.

### 6.3 One-shot render

Create one `RenderTargetTexture` per drone, lazily, on the first shutter of that size. Options: no mipmaps, `Constants.TEXTURETYPE_UNSIGNED_BYTE`, `Constants.TEXTUREFORMAT_RGBA`, nearest sampling, `doNotChangeAspectRatio: false` (Babylon recomputes the sensor projection after binding the target; the main canvas stays unchanged). Do not push it onto `scene.customRenderTargets`. That array renders every frame and would ignore render-on-demand.

```typescript
rtt.activeCamera = captureCamera;
captureCamera.fov = profile.vfovRad;
captureCamera.aspect = width / height;
rtt.render(true);
```

`render(true)` draws this target once. It does not present the HTML canvas. Do not call `runtime.requestRender()` for the capture itself. Do not set `preserveDrawingBuffer`.

GPUTexture access, Babylon 8.56.2 (`fly/package.json`):

```typescript
function gpuTextureOf(rtt: RenderTargetTexture): GPUTexture {
  const hardware = rtt.getInternalTexture()?._hardwareTexture as { underlyingResource?: GPUTexture } | undefined;
  const texture = hardware?.underlyingResource;
  if (!texture || (texture.usage & GPUTextureUsage.COPY_SRC) === 0) {
    throw new Error("Drone capture texture is missing COPY_SRC.");
  }
  return texture;
}
```

Babylon's render-target usage in this version includes `CopySrc` (`webgpuTextureManager.js`). If the assert fails, throw. Do not fall back to `readPixels` on the canvas.

### 6.4 Copy and unpad

```typescript
const bytesPerRow = Math.ceil(width * 4 / 256) * 256;
const buffer = device.createBuffer({
  size: bytesPerRow * height,
  usage: GPUBufferUsage.COPY_DST | GPUBufferUsage.MAP_READ,
});
const encoder = device.createCommandEncoder();
encoder.copyTextureToBuffer(
  { texture },
  { buffer, bytesPerRow, rowsPerImage: height },
  { width, height, depthOrArrayLayers: 1 },
);
device.queue.submit([encoder.finish()]);
await buffer.mapAsync(GPUMapMode.READ);
const packed = new Uint8Array(buffer.getMappedRange()).slice();
buffer.unmap();
buffer.destroy();
```

Unpad:

```typescript
const row = width * 4;
const rgba = new Uint8Array(row * height);
for (let y = 0; y < height; y++) rgba.set(packed.subarray(y * bytesPerRow, y * bytesPerRow + row), y * row);
```

Reuse one readback buffer across shots of the same size. Destroy it on feature dispose.

### 6.5 JPEG and metadata

`fly/package.json` gains `"jpeg-js": "file:../jpeg-js"`. The package is CommonJS. Encode only inside `capture()`, never to load or fly the drone:

```typescript
const encoder = await import("jpeg-js");
const encoded = encoder.default.encode({ data: rgba, width, height }, 92);
```

`fly/vite.config.ts` lists `jpeg-js` in `optimizeDeps.include` so the dev server prebundles it and the first still does not restart the page. The browser fetch is still the dynamic import. Removing it from that list matches a stricter just-in-time rule and can full-reload the flight on the first shutter. Leave that choice until a still is tested.

`jpegExif.ts` inserts segments immediately after SOI `FF D8`:

1. APP1 EXIF. Marker `FF E1`. Payload starts with `Exif\0\0`, then a little-endian TIFF header `49 49 2A 00` and IFD0 offset 8. IFD0 tags, sorted by tag id: Make `0x010F` ASCII `DJI`, Model `0x0110` the camera model, Orientation `0x0112` SHORT 1, XResolution `0x011A` RATIONAL 72/1, YResolution `0x011B` RATIONAL 72/1, ResolutionUnit `0x0128` SHORT 2, Software `0x0131` ASCII `0SFS`, DateTime `0x0132` ASCII `YYYY:MM:DD HH:MM:SS`, ExifIFD pointer `0x8769`. Exif IFD: ExposureTime `0x829A` RATIONAL 1/1000 for Mini 2 and 1/800 for Air 2S, FocalLength `0x920A` RATIONAL of the millimetre value in section 4.1 (numerator/denominator reduced), FocalLengthIn35mmFilm `0xA405` SHORT of the rounded equivalent, PixelXDimension `0xA002`, PixelYDimension `0xA003`.
2. APP1 XMP. Marker `FF E1`. Namespace header `http://ns.adobe.com/xap/1.0/\0` then this packet, with the numbers printed to 4 decimal places:

```xml
<?xpacket begin="" id="W5M0MpCehiHzreSzNTczkc9d"?>
<x:xmpmeta xmlns:x="adobe:ns:meta/">
 <rdf:RDF xmlns:rdf="http://www.w3.org/1999/02/22-rdf-syntax-ns#">
  <rdf:Description rdf:about=""
    xmlns:drone-dji="http://www.dji.com/drone-dji/1.0/"
    drone-dji:FlightYawDegree="YAW"
    drone-dji:FlightPitchDegree="PITCH"
    drone-dji:FlightRollDegree="ROLL"
    drone-dji:GimbalYawDegree="GIMBAL_YAW"
    drone-dji:GimbalPitchDegree="GIMBAL_PITCH"
    drone-dji:GimbalRollDegree="GIMBAL_ROLL"
    drone-dji:RelativeAltitude="AGL_M"
    drone-dji:AbsoluteAltitude="MSL_M"
    drone-dji:CalibratedFocalLength="F_PX"
    drone-dji:CalibratedOpticalCenterX="CX"
    drone-dji:CalibratedOpticalCenterY="CY"
    drone-dji:CaptureScale="SCALE"
    drone-dji:ImageSource="0SFS"/>
 </rdf:RDF>
</x:xmpmeta>
<?xpacket end="w"?>
```

`YAW`, `PITCH`, `ROLL` are the airframe degrees at the shutter (`heading`, `pitch`, `roll` from `FlightState`). `GIMBAL_PITCH` is the gimbal degree. `F_PX = (height/2) / tan(vfov/2)` at the captured pixel height. `CX = width/2`, `CY = height/2`. Gimbal yaw is aircraft heading plus the actual yaw offset, wrapped to ±180°; gimbal roll is the actual optical-axis roll override. Omitted legacy metadata fields default to aircraft heading and zero roll.

### 6.6 Zip

`zipStore.ts` writes a STORE-method zip (method 0). JPEGs are already compressed. No dependency.

Local file header: `50 4B 03 04`, version 20, flag 0, method 0, time 0, date 0, crc32, compressed size = uncompressed size, name length, extra length 0, then the UTF-8 name, then the bytes.

Central directory header per file: `50 4B 01 02`, version made 20, version needed 20, then the same sizes and the crc, relative offset of the local header.

End of central directory: `50 4B 05 06`, disk numbers 0, entry count, central directory size, offset, comment length 0.

CRC-32 is ISO-HDLC, polynomial `0xEDB88320`, init `0xFFFFFFFF`, final xor `0xFFFFFFFF`.

File names: `DJI_<index>_<pitchDeg>_<yawDeg>.jpg` with pitch and yaw printed to 2 decimals and a zero-padded index of 3 digits. Plus `manifest.json` containing the grid and the per-shot telemetry.

Trigger the download with an `<a download="0sfs-360-<droneId>.zip">` whose href is a `Blob` of type `application/zip`, then revoke the object URL. One shot from the shutter button downloads a zip of that single JPEG the same way, name `0sfs-still-<droneId>.zip`, so both paths share the packer.

---

## 7. Phase 4 — optical flow

Runs inside the lazy chunk, once per rendered frame, only while a drone is the current aircraft and the simulation is not paused. It uses a proxy render, not the full still.

### 7.1 Proxy

`FLOW_WIDTH = 320`. Height is `Math.round(320 * sensorH / sensorW / 2) * 2`.

- Mini 2: 320 × 240.
- Air 2S: 320 × 214.

Render the capture camera into a second, small `RenderTargetTexture` of that size only while `capture360` is true or a shutter is pending. While neither is true, do not render the proxy and do not dispatch compute. One proxy `rtt.render(true)` per displayed frame during a sequence is the flow's input. It is offscreen. Do not request an extra canvas frame for it; `setSimRunning(true)` is already drawing the view.

Luma, written by a compute pass into an `r32float` storage texture:

```text
Y = 0.2126 R + 0.7152 G + 0.0722 B
```

### 7.2 Workgroup size

WebGPU's minimum `maxComputeInvocationsPerWorkgroup` is 256, so 8×8 is legal on the M5 and on Adreno 650. Still read the limit:

```typescript
const limit = device.limits.maxComputeInvocationsPerWorkgroup;
const wgX = 8;
const wgY = Math.max(1, Math.min(8, Math.floor(limit / wgX)));
```

Bake `@workgroup_size(${wgX}, ${wgY})` into the WGSL string and create the pipeline once per `(device, wgY)`. Both target devices keep `wgY = 8`. If `wgY < 1`, skip the GPU path and set `flowSource = "imu"`.

### 7.3 WGSL

`flowWgsl.ts` exports two strings, `lumaWgsl(wgX, wgY)` and `lkWgsl(wgX, wgY)`.

Luma bindings:

```text
@group(0) @binding(0) var src: texture_2d<f32>;
@group(0) @binding(1) var dst: texture_storage_2d<r32float, write>;
```

Lucas–Kanade, one pyramid level per dispatch. Window radius `R = 3` (7×7). Bindings:

```text
@group(0) @binding(0) var prev: texture_2d<f32>;          // r32float luma
@group(0) @binding(1) var curr: texture_2d<f32>;
@group(0) @binding(2) var<storage, read> flow_in: array<vec2<f32>>;
@group(0) @binding(3) var<storage, read_write> flow_out: array<vec2<f32>>;
@group(0) @binding(4) var<storage, read_write> confidence: array<f32>;
struct LkParams { width: u32, height: u32, level: u32, scale: f32 }
@group(0) @binding(5) var<uniform> params: LkParams;
```

`scale` is 1 at the fine level and 2 at the coarse level (the coarse flow was computed on half-res and is multiplied by 2 when sampled as the initial guess).

Kernel, for each pixel `(x, y)` inside the image and outside a 3 px border. Coarse level is half resolution, dispatched separately; the TypeScript side downsamples luma with a 2×2 box in a third tiny shader of the same workgroup, bindings `src` r32float and `dst` r32float.

```wgsl
let ix = (textureLoad(prev, vec2<i32>(x + 1, y), 0).r - textureLoad(prev, vec2<i32>(x - 1, y), 0).r) * 0.5;
let iy = (textureLoad(prev, vec2<i32>(x, y + 1), 0).r - textureLoad(prev, vec2<i32>(x, y - 1), 0).r) * 0.5;
```

Accumulate over `dx, dy` in `[-3, 3]`:

```text
Gxx += Ix*Ix;  Gxy += Ix*Iy;  Gyy += Iy*Iy
It = curr(x+dx, y+dy) - prev(x+dx, y+dy)     // after warping the curr sample by the initial flow
Gxt += Ix*It;  Gyt += Iy*It
```

Initial flow at the fine level is `flow_in` from the upsampled coarse field. At the coarse level `flow_in` is zeros.

```text
det = Gxx*Gyy - Gxy*Gxy
if det > 1e-8:
  ux = (Gyy * Gxt - Gxy * Gyt) / det
  uy = (Gxx * Gyt - Gxy * Gxt) / det
else:
  ux = 0; uy = 0
lambda_min = 0.5 * (Gxx + Gyy) - sqrt((0.5 * (Gxx - Gyy))² + Gxy²)
```

Write `flow_out = initial + vec2(ux, uy)` and `confidence = max(lambda_min, 0)`.

Dispatch order per frame: luma full, luma half (box), LK coarse with zero initial flow, LK fine with `flow_in = 2 * coarse` sampled at `pixel/2`.

### 7.4 Estimator

`flowEstimator.ts`. Do not sort the full 320×240 field. Stride 16.

```typescript
const fPx = (flowHeight / 2) / Math.tan(vfovRad / 2);
```

Collect `flow` and `confidence` where the index `i % 16 === 0`. Median flow x, median flow y, median confidence. Median is the middle of a copied array after `sort`.

```text
Δψ = atan(medianFlowX / fPx)          // image +x is right, positive yaw is clockwise, matching JSBSim
Δθ = atan(-medianFlowY / fPx)         // image +y is down, positive gimbal pitch is up
ω  = hypot(Δψ, Δθ) / dtFrame
```

Ignore the sample when `dtFrame <= 0` or `dtFrame > 0.1`.

`CONFIDENCE_MIN = 1e-4`. Also require at least 20% of the stride samples to have `lambda_min > 1e-4`. If either fails, `flowSource = "imu"` and the visual deltas are not applied. Sky and water fall through this test because their structure-tensor eigenvalues sit near 0.

Fusion, only while the sequence is in `OPTICAL_FLOW_FINE_ALIGN`:

```text
if flowSource is visual:
  visualYaw   = imuYaw + Σ Δψ          // sum since the node started
  visualPitch = gimbalPitch + Σ Δθ
else:
  decay Σ toward 0 with time constant 0.5 s
  visualYaw   = imuYaw
  visualPitch = gimbalPitch
```

`imuYaw` is `attitude/psi`. The fine-align error is `wrap(targetYaw - visualYaw)` and `targetGimbalPitch - visualPitch`.

Motion-blur gate uses the capture focal length in pixels, not the proxy:

```text
fCapture = (captureHeight / 2) / tan(vfov/2)
Texp = 1/1000 s for Mini 2, 1/800 s for Air 2S
ωBlur = 0.5 / (fCapture * Texp)       // rad/s, 0.5 px during the exposure
```

That is 10.138°/s on a full-res Mini 2 frame and 6.731°/s on a full-res Air 2S frame. If `captureSize` scaled the frame, recompute `fCapture` from the scaled height. The angular threshold stays "half a pixel at the resolution that will be stored."

```text
ωGate = flowSource is visual ? ωVisual : hypot(p, q, r)
settled = ωGate < ωBlur
```

---

## 8. Phase 5 — spherical grid and state machine

### 8.1 Grid

`sphericalGrid.ts`. Overlap target is 30% in the image, so the step between optical axes is at most `0.70` times the covered field.

Vertical: endpoints are exactly `pitchMin = -90°` and `pitchMax` (+20 or +24). Let `span = pitchMax - pitchMin` in radians and `vStepMax = 0.70 * vfovRad`. Ring count is the smallest `n ≥ 2` such that `span / (n - 1) ≤ vStepMax`. Ring `i` is at `pitchMin + i * span / (n - 1)`.

Horizontal coverage of a level gimbal at pitch `g` radians (0 at the horizon, negative down):

```text
if |cos g| < 1e-4 or the coverage below is ≥ 2π: nYaw = 1
else:
  coverage = 2 * atan(tan(hfov/2) / |cos g|)
  stepMax = 0.70 * coverage
  nYaw = ceil(2π / stepMax)
  yaw step = 2π / nYaw
```

Yaw samples on a ring are `k * yawStep` for `k = 0 .. nYaw-1`, degrees clockwise from north, the same sign as JSBSim `ψ`.

The function's required output:

Mini 2, 4 rings, vertical step `110/3` degrees, 22 shots:

| Ring | Pitch ° | Yaw count | Yaw step ° |
| --- | --- | --- | --- |
| 0 | −90 | 1 | 360 |
| 1 | −53.3333333333 | 6 | 60 |
| 2 | −16.6666666667 | 8 | 45 |
| 3 | +20 | 7 | 51.4285714286 |

Air 2S, 4 rings, vertical step 38°, 20 shots:

| Ring | Pitch ° | Yaw count | Yaw step ° |
| --- | --- | --- | --- |
| 0 | −90 | 1 | 360 |
| 1 | −52 | 5 | 72 |
| 2 | −14 | 7 | 51.4285714286 |
| 3 | +24 | 7 | 51.4285714286 |

A unit test builds the grid from the FOV function and compares these tables to 1e-6 degrees. Nodes are ordered ring 0 first, then increasing yaw.

### 8.2 State machine

`captureSequence.ts`. States: `IDLE`, `SLEWING`, `OPTICAL_FLOW_FINE_ALIGN`, `SETTLING_ZERO_VELOCITY`, `CAPTURING`, `NEXT_NODE`, `ZIP_EXPORT`, `DONE`, `ABORTED`.

`capture360` false from the phone or the HUD button, from any state except `CAPTURING`, goes to `ABORTED` and then `IDLE` without a zip. `CAPTURING` finishes the in-flight readback, then aborts.

| State | Exit |
| --- | --- |
| IDLE | `capture360` becomes true. Copy the grid. `nodeIndex = 0`. Go to SLEWING. Set yaw mode to `angle` and gimbal target to the node. |
| SLEWING | Command `yawAngleDeg` and `gimbalPitchDeg` of the node. The velocity sticks are forced to 0 for the whole sequence. Existing hold anchors stay fixed; pending horizontal and altitude anchors capture independently after braking to the section 4.4 speed thresholds. Exit to FINE_ALIGN when `\|wrap(ψ - target)\| < 3°` and `\|gimbal - target\| < 1°`, or after 8 s, whichever first. On the timeout, still advance; record `slewTimeout: true` on the shot. |
| OPTICAL_FLOW_FINE_ALIGN | Add the fused visual error into the yaw and gimbal targets, clamped to ±2°. Exit after 0.6 s or when both absolute errors are `< 0.15°`. If `flowSource` is `imu`, skip this state in one step. |
| SETTLING_ZERO_VELOCITY | Exit when `ωGate < ωBlur` for 6 consecutive physics steps (50 ms at 120 Hz), or after 2 s. On timeout, capture anyway and set `blurRisk: true`. |
| CAPTURING | `await captureFrame(...)`. Push the JPEG bytes, the pose, and the telemetry. Go to NEXT_NODE. |
| NEXT_NODE | `nodeIndex += 1`. If nodes remain, SLEWING. Else ZIP_EXPORT. |
| ZIP_EXPORT | Build the zip, trigger the download, DONE. |
| DONE | Stay until `capture360` is released, then IDLE. |
| ABORTED | IDLE on the next step. |

`isCapturing` is true from SLEWING through ZIP_EXPORT. `isSettled` is true only in SETTLING when the blur gate has passed, and during CAPTURING. Telemetry `nodeCount` is the grid length.

The sequence commands the controller by replacing `DroneStickCommand.yawMode`, `yawAngleDeg`, and `gimbalPitchDeg`, and by zeroing pitch, roll, and vertical. It does not write JSBSim properties itself.

---

## 9. File-by-file blueprint

| Path | Action | In the startup graph? |
| --- | --- | --- |
| `.github/AGENTS.md` | Phase 0 row | yes, docs |
| `docs/parity-checklist.md` | Phase 0 section | yes, docs |
| `Quadcopter/`, `DJI-Quadcopter/`, `DJI-Mini2/`, `DJI-Air-2S/` `public/0sfs-mod.json` | Declarative manifests (section 3.1; `quad` and `quadcopter` data in section 4.1) | fetched on mod add |
| `DJI-Mini2/public/jsbsim-data/aircraft/dji-mini-2/dji-mini-2.xml` | XML, section 4.3 | fetched on drone boot |
| `DJI-Air-2S/public/jsbsim-data/aircraft/dji-air-2s/dji-air-2s.xml` | XML, section 4.3 | fetched on drone boot |
| `Quadcopter/README.md`, `DJI-Quadcopter/README.md`, `DJI-Mini2/README.md`, `DJI-Air-2S/README.md` | What each mod owns and that the flight chunk stays in fly | no |
| `aircraft/src/aircraftIds.ts` | register/unregister family and aircraft ids | yes, tiny |
| `aircraft/src/aircraftCatalog.ts` | `procedural`, revision, unregister | yes, tiny |
| `aircraft/src/createAircraftModel.ts` | retain procedural LODs across refreshes; load nonempty-path HD LODs | yes, calls the mesh hook |
| `fly/src/flight/jsbsim/fdmProfiles.ts` | `none`, `dataSource`, `kinematic`, unregister | yes |
| `fly/src/flight/jsbsim/bootstrapC172.ts` | skip propulsion when `engine === "none"` | yes |
| `fly/src/flight/jsbsim/resetFlightLocation.ts` | same skip | yes |
| `fly/src/flight/jsbsim/createJsbsimRuntime.ts` | mod hydrate branch | yes |
| `engines/src/engineControl.ts` | `"none"` stub | yes |
| `fly/src/flight/mods/applyDynamicMod.ts` | `onModAdded` only | yes |
| `fly/src/flight/mods/modSession.ts` | `removeMod` | yes |
| `mods/src/aboutTree.ts`, `mods/src/ModsPanel.tsx`, `fly/src/flight/mods/ModsTab.tsx` | Remove button | yes |
| `fly/src/flight/hud/FlightControlPanel.tsx` | refresh families on catalog revision | yes |
| `fly/src/flight/createFlightSimApp.ts` | gate calls only | yes |
| `fly/src/flight/drone/droneGate.ts` | dynamic import boundary | yes |
| `fly/src/flight/drone/droneFeature.ts` | `attach`, owns the rest | **lazy** |
| `fly/src/flight/drone/droneTypes.ts` | constants, FOV | lazy |
| `fly/src/flight/drone/droneController.ts` | section 4.4: commands, holds, arming, landing slowdown | lazy |
| `fly/src/flight/drone/droneFlightModel.ts` | section 4.4: JSBSim reads and wrench writes, the multirotor runtime, WASM load | lazy |
| `fly/src/flight/drone/droneMotorTelemetry.ts` | rotor position and direction helpers | lazy |
| `multirotor/src/airframe.ts`, `layout.ts`, `rotorPlant.ts`, `flightController.ts`, `multirotor.ts` | `@0sfs/multirotor`: airframe buffers, flight controller, rotor plant, runtime (section 4.4) | lazy, imported only by the drone chunk |
| `multirotor/wasm/` | Rust twin of both kernels; `single/multirotor.wasm` fetched when a drone flight model is created | lazy |
| `engines/src/quadEngineMonitor.ts` | engine widget follow-up | lazy, supplied by the drone feature |
| `fly/src/flight/drone/droneGimbal.ts` | section 4.6 | lazy |
| `fly/src/flight/drone/buildDroneProceduralMesh.ts` | section 4.7 | lazy |
| `drone-camera/src/captureFrame.ts` | section 6 | lazy |
| `drone-camera/src/jpegExif.ts` | section 6.5 | lazy |
| `drone-camera/src/zipStore.ts` | section 6.6 | lazy |
| `drone-camera/src/flowWgsl.ts` | section 7.3 | lazy |
| `drone-camera/src/opticalFlow.ts` | device, pipelines, dispatch | lazy |
| `drone-camera/src/flowEstimator.ts` | section 7.4 | lazy |
| `drone-camera/src/sphericalGrid.ts` | section 8.1 | lazy |
| `drone-camera/src/captureSequence.ts` | section 8.2 | lazy |
| `fly/vite.config.ts` | static serve each mod `public/` at `/dev-mods/quadcopter`, `/dev-mods/dji-quadcopter`, `/dev-mods/dji-mini2`, `/dev-mods/dji-air-2s` | dev only |
| `fly/package.json` | `jpeg-js` file dependency | the chunk, not a static import from `main.tsx` |
| `rc/src/protocol.ts` | availability predicate and optional fields | yes, small |
| `rc/src/drone/droneProtocol.ts` | command types and builders | lazy on both ends |
| `rc/src/drone/loadQuadcopter.ts` | dynamic import | yes, one function |
| `rc/src/drone/PhoneQuadcopter.tsx` | section 5.2 | **lazy** |
| `rc/src/createPhoneControlSession.ts` | `getDroneCommand` | yes, stores a small object |
| `rc/src/PhoneController.tsx` | calls `loadQuadcopter` when `status.drone.available` | yes, no static quad import |
| `rc/docs/phone-controller.md` | one paragraph: quad controls appear only after the computer is flying a drone from the installed mod | docs |

`droneFeature.ts` exports `attach` as specified on the gate. It constructs the controller, gimbal, mesh mount, flow, and sequence, and registers them through `setHooks` and `setMesh`.

Tests live next to the pure modules and do not boot Babylon unless they already have a `NullEngine` harness in the repo: `sphericalGrid.test.ts`, `droneController.test.ts` (velocity commands, hold anchors and their signs, yaw mapping, the landing slowdown, the gimbal slew, arming and disarming, the 120 Hz guard, bad samples and resets), `quadAirframeData.test.ts` (XML and manifest agree: CG, wrench properties, feet, mass, inertia, drag), `quadJsbsimLab.test.ts` (the real XML in the WASM JSBSim with the multirotor: settling on the feet, hover RPM, each mode's speed within its tilt and height limits, climb and descent through the vortex ring, a 10 m/s wind, landing), `multirotor/tests/*.test.ts` (rotor plant, allocation, the closed loop on a flat-earth rigid body, and WASM fallback and parity), `jpegExif.test.ts` (SOI then APP1, model string present), `zipStore.test.ts` (local header magic and method 0), `droneGate.test.ts` (source files listed above do not name `droneFeature`).

---

## 10. Verification

Unit tests first, from `fly/`:

```sh
cd ~/gh/0sfs/fly && npx vitest run --configLoader native src/flight/drone ../multirotor/tests src/flight/mods/modSession.test.ts
cd ~/gh/0sfs/multirotor/wasm && node scripts/check.mjs   # WASM receipt and bit-for-bit parity
```

Add `modSession` coverage in the existing mods tests: add a declarative fixture, assert `loadDroneFeature` was not called; remove it, assert the catalog revision dropped the family.

Manual, M5 MacBook Air, Firefox. Start with `cd ~/gh/0sfs/fly && npm run dev`. Do not use a background server left by the agent.

1. Open `/fly/`. DevTools network: no request contains `droneFeature` or `0sfs-mod.json` for drones. The aircraft gallery has no DJI family.
2. Mods tab, choose This computer, add `0SFS/DJI-Mini2`. If a GitHub copy is already installed, remove it first. Network shows `/dev-mods/dji-mini2/0sfs-mod.json`, then `dji-quadcopter`, then `quadcopter`, and no `droneFeature` chunk. Adding Air 2S the same way puts both variants on the DJI family. JSBSim XML is fetched from that same `/dev-mods/` root when the aircraft is applied, not when the mod is added.
3. Apply Mini 2. Network then shows `dji-mini-2.xml` and the drone chunk. The physics log line `Fixed-step rate` reads `physicsHz: 120`. The airframe is the HD model nearby and the procedural silhouette beyond 15 m, never the Cessna. First-person view does not roll with the airframe when the stick is pushed; the horizon stays level and the gimbal pitch changes only from the gimbal control.
   Flight:
   - **On the ground.** A drone applied on the ground sits on its feet with the ARM gauge off and the rotors stopped. Push the climb stick over half to arm: the rotors spin and it lifts off.
   - **Hover.** Centered sticks hold position and height. The engine widget shows four simulated RPMs near hover: about 9,000 for the Mini 2 and 6,200 for the Air 2S at sea level.
   - **Translation.** Full forward stick reaches the mode's speed without exceeding its tilt: 10 m/s at 25° in Normal, 16 m/s at 40° in Sport for the Mini 2.
   - **Landing.** Holding descent slows the drone near the ground and it touches down gently. Holding the stick fully down on the ground for a second disarms it.
   - **Wind.** With weather wind set, the drone leans into the wind while holding position.
4. Phone: `cd ~/gh/0sfs/rc && npm run dev` is not required; the fly dev server already serves `/rc/`. Pair from Remote Control. Before a drone is applied, the phone shows the fixed-wing controls and its network log has no `PhoneQuadcopter` chunk. After Mini 2 is applied, the next heartbeat carries `drone.available: true` and the phone requests the quad chunk. Left stick climbs and yaws. Right stick translates. The gimbal slider stops at +20 on the Mini 2 and +24 on the Air 2S.
5. Camera → Take still (or the phone shutter) downloads one JPEG zip. The JPEG contains `FlightYawDegree` and `GimbalPitchDegree`. The image size is 4000×3000, or an integer divisor when `maxTextureDimension2D` or `maxBufferSize` requires it. The main canvas size does not change.
6. `360` runs 22 shots for the Mini 2 and 20 for the Air 2S, then one zip. The phone tape counts `nodeIndex/nodeCount`. Pointing the camera at the sky sets `flowSource` to `imu` and the sequence still finishes.
7. Remove the mod. The flight returns to the Cessna, the gallery loses the DJI family, and a later reload does not request the drone chunk until the mod is added and a drone is applied again.

Manual, Galaxy Note 20 Ultra 5G, Chrome. Repeat steps 3, 5, and 6 on the Air 2S. In `chrome://gpu` or the renderer diagnostics, confirm the mode is `webgpu`. If `maxTextureDimension2D` is below 5472, `CaptureScale` in the XMP is an integer greater than 1 and both sides of the image are the rounded quotient. The compute pipeline creates successfully with workgroup `8, 8` (`maxComputeInvocationsPerWorkgroup` is at least 256). A WebGL session, if one can be forced with `?renderer=webgl2`, shows the refusal string and does not throw on shutter.

---

## 11. What the coding agent does not decide

- Do not put the drones in the built-in fleet.
- Do not import `droneFeature.ts` from the startup graph.
- Do not prefetch the chunk.
- Do not add a WebGL capture path.
- Do not add a WASM optical-flow module.
- Do not model the rotors in JSBSim XML, FCS or propulsion. Rotors come from `@0sfs/multirotor`, which writes one body-axis wrench at the CG. The XML aerodynamics are body drag only.
- Do not simulate the ESC, motor or battery; the simulation stops at the spinning propeller.
- Do not publish the `drones` repository or the flight site.
