# 360° drone session — 28 October 2026

Session note, not the contract. The contract is [SPEC-360-DRONE.md](SPEC-360-DRONE.md). This page is what that session learned after the spec was written, which of those lessons are now in the spec, and what is still open. Nothing here was committed or published.

## Latest: flight model down to the spinning propeller

The placeholder force controller is being replaced by a real rotor model and flight controller. The simulation goes down to the spinning propeller and stops: each rotor's speed follows its command with a first-order lag in place of the ESC, motor and battery. That was the user's decision on 28 October. The reason is data, not cost: DJI publishes no motor data, and the probe's four JSBSim motors cost only 3.8 µs per step. This supersedes the estimated-RPM notes in the follow-ups below.

What was built:

- **[`multirotor/`](../../multirotor/README.md)** (`@0sfs/multirotor`). TypeScript kernels with a Rust/WASM twin that matches them bit for bit.
  - **Rotor plant.** For each rotor:
    - the speed lag;
    - blade element and momentum theory, with Glauert's inflow and the empirical axial curve through the vortex ring and windmill states;
    - vortex-ring buffet;
    - Sanchez-Cuevas et al.'s multirotor ground effect;
    - the H-force and the hub moment;
    - reaction torque and the gyroscopic moment.

    It sums them into one body-axis force and moment about the CG.
  - **Flight controller.** Four loops:
    - velocity PI with jerk-limited shaping;
    - tilt-prioritised attitude, then heading;
    - a rate PI with the gyroscopic term;
    - minimum-norm allocation in which yaw gives way first.

    A thrust-efficiency factor, the plant's last thrust over static thrust, stands in for an accelerometer.
  - **Runtime.** It runs the TypeScript until the WASM loads, and falls back to TypeScript for good if a kernel fails.
  - **Tests.** 26 tests and the WASM parity check.
- **JSBSim XMLs** (Mini 2 and Air 2S):
  - the rotor wrench as BODY external reactions at the CG, via `multirotor/force-{x,y,z}-lbs` and `multirotor/moment-{l,m,n}-ftlbf`;
  - per-axis body drag;
  - the component-estimate mass and inertia;
  - four STRUCTURE feet;
  - no propulsion.

  `quadJsbsimLab.test.ts` flies the real XML in the WASM JSBSim.
- **Manifests.** Normal and Sport mode limits from DJI's specs, the propeller, the rotor layout, the CG and the gains are per aircraft in `quad`. The shared Quadcopter manifest keeps `kPos`, the deadzone, position-hold speed and the property names. `cdA`, the inertia fields, `rotorHoverRpm` and the private wind are gone.
- **Fly.**
  - **Flight model and commands.** `droneFlightModel.ts` reads one JSBSim batch and writes the wrench and `multirotor/armed`. `droneController.ts` `planDroneCommand` turns sticks and holds into a velocity command, with the same hold signs. It adds arming: climb stick over half on the ground arms, and full descent held for 1 s on the ground disarms. It also adds a landing flare. Near the ground the descent limit is `max(0.1, 0.5·(AGL − touchdown AGL))` m/s, where touchdown AGL is the CG's height above the lowest foot. The roadmap's stage 6 has the results.
  - **Rotor speed.** The engine monitor and props show each rotor's simulated RPM.
  - **Procedural model.** It puts the motors at the real hubs.
  - **Removed.** `planDroneForce`, `droneWind.ts` and the RPM estimate.

  Handoffs B and C built this wiring on 28 October, and handoff E integrated it. The handoffs are described below.

Numbers, from `multirotor`'s flat-earth lab:

| | Mini 2 | Air 2S |
| --- | --- | --- |
| Hover at sea level | 8,994 RPM | 6,243 RPM |
| Thrust at maximum RPM over weight | 2.19 | 2.17 |
| Normal: speed, peak tilt | 10.0 m/s at 24.7° | 15.0 m/s at 24.9° |
| Sport: speed, peak tilt | 16.0 m/s at 37.9° | 18.9 m/s at 34.8° |

The Air 2S holds its 6 m/s descent through the vortex ring state. A step of controller plus plant costs 3.7–10.9 µs in TypeScript and 9.4–16.8 µs in WASM, under 0.2% of a 120 Hz step.

In the real-JSBSim lab, `quadJsbsimLab.test.ts`, speeds and hover RPM match these to within 0.2%, and peak tilts are up to 1° higher. Its table is in the roadmap's Numbers section, and its landing results are in stage 6. The whole step, JSBSim included, costs 14.2 µs for the Mini 2 and 12.2 µs for the Air 2S.

Out of scope or deferred:

- **Rotor-wake interference and CFD body tables.** Without them, rear rotors don't run faster and there is no hover download.
- **Battery, ESC and flight time.**
- **Sensors and the EKF.**
- **Flight-log validation.**
- **A worker.** Blocked on FOSS Earth.
- **The offline blade-element oracle.**
- **Cine mode.**
- **Turbulence and collision probes.**

The [roadmap](360-drone-flight-model-roadmap-2026-10-28.md) has each stage's status. [SPEC-360-DRONE](SPEC-360-DRONE.md) sections 0, 4.1–4.5, 4.7, 4.8 and 9–11 describe the new contract.

The fly wiring was split into five handoff prompts, each run by a separate agent:

- **A:** the JSBSim XMLs and the real-JSBSim lab.
- **B:** the fly runtime: flight model, commands, arming and telemetry.
- **C:** the engine monitor and the procedural mesh.
- **D:** these docs.
- **E:** integration, the browser check and the user's test checklist, after the other four.

A–C ran at the same time, each owning its own files. They reported every cross-file fix instead of making it, and E made those fixes. The prompts, and each agent's report and evidence, are in the gitignored `build/multirotor-handoff/` on the machine that ran them, not in Git. Next, the user tests with `cd ~/gh/0sfs/fly && npm run dev`, following SPEC section 10 step 3.

Open:

- **WASM or TypeScript in production.** The WASM measured slower than the TypeScript. This is decision 7 in the roadmap.
- **120 versus 480 Hz.** Not measured. This is decision 5.
- **Touchdown speed.** The flare sets down at about 0.06 m/s. How that feels is for user testing.
- **Spawn height (not checked).** Reset and saved-flight placement put the CG at `stance.staticMeters` + 0.17 m above the terrain (`resetFlightLocation.ts`, `savedFlight.ts`). That margin suits planes. For the drones, with `staticMeters` 0.04, the feet start about 18 cm up. Arming starts as `!onGround` on the first sample. So a drone placed there may start armed and hover, instead of resting disarmed on its feet. If it does, give the drones their own stance: the CG is 23 mm (Mini 2) and 35 mm (Air 2S) above the feet, or about 10 mm at rest.
- **The lab's output path.** `quadJsbsimLab.test.ts` writes its results JSON into a scratch folder under `build/multirotor-handoff/`. Give it a stable path, or make the write opt-in.

Fixed during integration:

- **The drone was drawn mirrored.** fly's model frame has +X left (glTF's). The procedural mesh and `droneMotorPosition` both treated +X as right, so each rotor's speed and spin showed on the other side's prop. Every test passed, because the two agreed. Both now use +X left, and SPEC section 4.8 has the frames. Prop `direction` stays `-spin`.
- **Landing.** A constant 1 m/s touchdown peaked at 4.5 g on the Mini 2. Stopping 1 m/s within 3 g needs about 25 mm of travel, more than the feet have. The command layer now flares instead; the roadmap's stage 6 has the law and the results.

## Earlier plan: drone flight-model roadmap

The [flight-model roadmap](360-drone-flight-model-roadmap-2026-10-28.md) planned the replacement of the placeholder force controller with four real motors and propellers in JSBSim, in stages 0–9, each gated by a flight-lab test. What was built differs; see the section above. New per-step models are Rust/WASM kernels beside TypeScript references, as ground contact's are. The step itself stays on the main thread: the 21 October worker experiment failed its gate, worst of all at low altitude. Sound, recording, flight-lab batches and the offline compilers run off the main thread. See [Rust/WASM, threads and the main thread](360-drone-flight-model-roadmap-2026-10-28.md#rustwasm-threads-and-the-main-thread).

## Latest correction: high-RPM propellers show a 3D swept volume (C172 behavior)

The previous propeller pass misread the request and made sharp, individually drawn 3D blades the high-RPM default for drones. That is reverted. The intended behavior is the C172's: above the display alias limit (π / blades / frameSeconds, 0.8 hysteresis), the blades hide and a **semi-transparent 3D swept propeller volume** covers the rotor. The flat disc is an opt-in performance setting, **Aircraft → Flat propeller discs**, Off by default. Slow propellers still show their turning 3D blades.

- `rotating-prop/` is now its own local Git repository (`main`, no commits, no remote; intended `0SFS/rotating-prop`), split into `motion.ts`, `sweep.ts`, `disc.ts` and `binding.ts`, with its tests in `rotating-prop/tests/` (moved out of fly's `sharedPresentation.test.ts`, which keeps only gimbal tests). `src/sweep.ts` is the shared generator. It samples the blade triangles under a pivot, records the forward/aft reach at each radius, and revolves that into a closed, double-sided, semi-transparent lens. Pivots without geometry get a C172-proportioned lens.
- The C172 keeps its authored `Propeller_Disc`. `aircraftAnimation.ts` falls back to the generated sweep for propeller planes without one, and the flat disc only on opt-in. Drones use `createRotatingProp` from the lazy drone feature; nothing new is on the startup graph.
- `stepRotatingProp` no longer takes an `allowDisc` argument; blurring is automatic, and the caller chooses sweep or flat.
- Both looks now take their radius from the measured blades. The HD blades measure ≈ 59.5 mm (Mini 2) and ≈ 91.7 mm (Air 2S); the manifests declare 45 and 53 mm. Manifest data was not changed. See the realism audit.

Verification: 6 rotating-prop tests, plus 314 passing across rotating-prop, aircraft (C172 rig), fly aircraft, drone and gimbal suites (1 existing expected-fail). Typechecking retains only the three pre-existing errors. A real WebGPU browser check on the existing 7470 server (isolated headless profile, removed afterwards) passed. Setting Off by default. C172 running: blades hidden, authored sweep shown, no flat disc. Adding both mods fetched no feature, HD model or JPEG. Mini 2 HD, Mini 2 procedural and Air 2S HD each showed four sweeps, no discs and hidden blade pivots, and held while paused. The flat opt-in swapped four sweeps for four discs live while paused, and turning it off restored the sweeps. No runtime exceptions or console errors. Screenshots, including edge-on sweep vs flat comparisons, are in fly's ignored `build/prop-sweep/shots/`, with `report.json` and `check-browser.mjs`. The older `build/drone-props/` artifacts verified the superseded blades-by-default behavior.

No commit, publication, renderer/ground-contact edit or server start/stop occurred.

## Latest follow-up: four-motor engine widget

Quadcopters now have four Cessna-style RPM circles in a top-view X layout, with FL/FR above RL/RR and a forward arrow. The compact engine widget opens the existing Engine tab, which shows a larger quartet. `engines/quadEngineMonitor` composes the existing RPM face, shaft renderer and simulation-time animation; its input is plain per-position readings. A type-only factory hook in the gate lets the lazy drone feature supply this monitor without putting the quad widget or estimator on the startup graph. Plane monitor behavior is retained, and the quad Engine tab omits unrelated combustion-engine controls.

*Superseded on 28 October: the monitor now shows each rotor's simulated speed from `@0sfs/multirotor`, marked Simulated; see the top section.* RPM was marked **Estimated**. The controller applied aggregate force/moments, so a presentation-only ideal X mixer apportions thrust, normalizes RPM² to the airframe's hover reference, and desaturates moments together to preserve collective thrust without negative motor force. Yaw uses a nominal reaction lever, not calibrated motor data. No flight physics or position-hold signs changed. Actual prop pivot positions select the corresponding motor reading across HD/procedural rotor orderings; adjacent corners counter-rotate. The same per-motor values drive visible props. The simulation clock and readings hold on pause, start at zero on a paused apply, and clear on aircraft changes/removal. Local motor readings do not enlarge the heartbeat.

Quad circles borrow the current WebGPU device. If drawing is unavailable or off, numbers remain; this path never attempts WebGL or requests another device. The shared summary now exposes cleanup for its resize observer; both native and quad monitors call it when destroyed.

Verification: 103 focused tests pass (88 widget/shared engine tests and 15 drone feature/controller/gate/motor tests). Typechecking retains only the same three pre-existing errors. Real WebGPU browser checks passed Mini 2 compact/detail layout and readings, motor response and pause, Air 2S profile changes, returning to Cessna's original single RPM ring, cleared drone telemetry, and no JPEG import. New quad implementation resources are absent at startup/mod add. Cropped screenshots for both drones and the Cessna comparison are in ignored `fly/build/quad-engine/shots/index.html`, with browser reports, test logs and a feedback launcher in that build folder. Shared engine test output is `fly/build/quad-engine-widget-tests.log`.

A focused A-held check changed the diagonal RPM difference from -36 to 3,849 RPM; its evidence is in `yaw-evidence.json`. All new/changed screenshots were inspected and the actual Test tab/checklist was verified. Temporary browsers/profiles were removed and Chrome clone cleanup found none. No commit, publication, renderer/ground-contact edit or additional server occurred. The user's existing development server remains running.

## Latest follow-up: surrounding gimbal sliders and interactive preview

The Camera tab now surrounds its optional preview with Roll above, Yaw below and vertical Pitch on the right. Capture commands follow. The preview stays a replaceable host-owned slot; no camera/airframe dependency on a display provider was added.

The backwards-compatible optional `attachInteraction(canvas)` provider port lets the host supply the exact shared 360 image viewer controls. Fly's lazy `dronePreviewLook` adapter applies Photograph → Looking and input-method preferences, bounded yaw/pitch targets, slider resynchronization and preview-only FOV zoom. Roll remains independently controlled. Closing the tab cancels held input and glide; a held or active 360 locks preview gestures, consumes those events and prevents propagation to the world camera. Provider replacement/removal releases input too. The provider bundle remains self-contained, with no renderer or viewer imports.

Preview magnification narrows the shared sensor render projection temporarily. The original camera FOV/mode/projection are restored synchronously before asynchronous readback; capture keeps the native camera model lens. Tests verify restoration even while GPU mapping is pending. Existing integer slider steps can round a fractional gesture target; browser assertions read the target/actual telemetry instead of treating that rounded thumb value as the command.

Verification: 33 focused tests pass across preview/provider, sensor transport, capture and Camera tab integration. Typechecking retains the same three pre-existing errors. The real WebGPU browser check passed the surrounding layout at 319/435-pixel panel widths, trusted drag, trackpad swipe, pinch, mouse-wheel zoom, hidden/Off lifecycle, capture lock, Air 2S profile swap and optional mod removal. The main chase FOV stayed unchanged; residual paused chase smoothing was below 0.000061 rad and 0.000051 m. A zoomed-preview still exported a 3,039,859-byte zip at the native lens FOV, with JPEG first requested at shutter. Artifacts and the feedback launcher are under ignored `fly/build/preview-controls/`; screenshot review is in its `shots/index.html`.

Four cropped screenshots were inspected, and the actual Test tab/checklist was verified. Temporary browsers and profiles were removed; Chrome clone cleanup found none. No commit, publication, renderer/ground-contact edit or additional server occurred. The pilot's existing 7470 server remains running. Physical trackpad/Safari/phone behavior remains a useful manual follow-up alongside the earlier full-360/device checks.

## Latest follow-up: optional, interchangeable live camera preview

`camera-preview/` is now its own local Git repository, with no commits or remote. Add **0SFS/camera-preview** in **Mods → This computer**, then select it under **Camera → Preview**. No aircraft or camera mod depends on this optional display. Installed alternatives appear in the same selector; Off and mod removal release preview resources without switching aircraft. Closing the tab stops preview rendering; reopening resumes it.

The shared mod API (`@0sfs/mods/cameraPreview`) defines a DOM provider consuming upright RGBA frames. The self-contained provider bundle owns presentation and pacing, with no renderer, aircraft, sensor-profile or JPEG imports. Fly stores approved module descriptors on mod add and imports the selected provider only when its Camera tab is visible. The host compiles without the optional provider checkout. Camera model mods continue to own sensor/lens/metadata data, selected by the airframe's camera dependency and `quad.cameraId`; variants can change those together without duplicating preview or capture code.

`drone-camera/sensorRender` now shares offscreen rendering, mesh filtering, sensor projection, WebGPU submission, padded-row removal and upright conversion between stills, optical flow and preview. Preview has its own target/readback, bounded to 640 pixels wide and 15 fps; it uses the same stabilized sensor camera as capture. It never loads `jpeg-js`. Provider errors and malformed replacements are isolated from flight updates and can be retried. Visibility tests cover the no-provider case, which caught and fixed an attribute-observer loop during browser verification.

Verification: 40 focused host/capture/schema tests and four provider lifecycle tests pass. Typechecking retains only the same three pre-existing errors. Real WebGPU browser checks passed Mini 2 FC7303 (640×480), Air 2S FC3411 (640×427), gimbal-driven image changes, hidden-tab rendering stop/reopen, Off, and mod removal retaining the aircraft and shutter. A paused Mini still exported a 3,039,986-byte zip with preview enabled, with JPEG first requested at shutter. Three cropped screenshots were inspected and the actual Test tab/checklist was verified. Artifacts, screenshots and the feedback launcher live under ignored `fly/build/camera-preview/`.

No commit, publication, renderer/ground-contact edit or additional server occurred. The pilot's existing 7470 server remains running; temporary browsers were closed. Full 360 and physical-phone capture checks remain open from earlier sessions.

## Latest follow-up: articulated camera housing and roll/yaw overrides

Both downloaded HD assets merged the camera housing and support structure into a single mesh. HD preparation now partitions disconnected authored shells without changing vertex attributes or losing triangles. Fixed mounting geometry stays fixed; yaw arms, roll yokes and camera housing/lens attach to separate authored pivots. Only housing/lens pitch. Rest-geometry checks retain all 109,470 Mini 2 / 142,400 Air 2S triangles, with maximum numerical rest displacement below 1.3e-8 m.

`camera-gimbal` now owns a fixed base plus yaw(Y) → roll(Z) → pitch(X) joints. It decomposes the stabilized desired camera pose into those joints, with the same math for visible meshes and sensor/cockpit view. The existing two-node API remains compatible. Camera adds Roll and Yaw sliders with target/actual readouts, per-airframe ranges, 5° nudges and zero resets. They are simulator overrides bounded from the manufacturer mechanical ranges; pitch ranges stay unchanged. A sequence temporarily centers the roll/yaw offsets and locks the sliders; manual targets return after stopping. Still XMP records actual gimbal heading/roll. A paused aircraft change initializes the new binding before posing, preventing stale out-of-range angles.

Focused tests: 34 passed across 11 files. Typechecking retains the same three pre-existing errors. The real WebGPU browser check passed both aircraft, all three sliders, sensor/model joint equality, and zero support-stage movement when pitching the generated model. A paused Mini 2 still exported a 3,441,562-byte zip with roll 20° in its XMP and JPEG requested only at shutter. Six focused screenshots cover both Camera tabs and level/down housing views; they were inspected. Artifacts and the feedback launcher live under ignored `fly/build/gimbal-axes/`.

No commit, publication, renderer/ground-contact edit or server start/stop occurred; the pilot's existing 7470 server remains running. Full 360 and physical-phone capture checks remain open from the earlier sessions.

## Latest follow-up: Camera tab replaces drone toolbar actions

The host's gimbal and capture actions now live in **+ → Camera**, shown only for an applied drone. The old `mountButtons` toolbar insertion and refusal span are removed. The lazy feature owns the DOM panel and passes it through `droneGate` to the normal tab shell; closing a tab only detaches it. `drone-camera/cameraControls` is the shared UI module, with lazy CSS, pitch slider/5° nudges, target/actual angles, still, 360 start/stop, progress, errors and compact help. Paused still capture works; starting a 360 requires resume and the panel says why.

A browser check on the existing WebGPU server passed Mini 2 gimbal movement, tab close/reopen during a sequence without resetting it, Stop, a real 3,477,112-byte paused still zip, JPEG loading only at shutter, and Air 2S's +24° limit. No drone actions remain in the toolbar, no camera UI/code loads on startup or mod add, and no runtime exceptions occurred. Screenshot review corrected the active paused hint and automatic gimbal target readout. The focused module/panel suite passes 21 tests; typechecking has only the same three existing errors. Artifacts and the feedback launcher are in ignored `fly/build/drone-camera-tab/`.

The UI move exposed an existing stop bug: `sequence.command()` forces capture true while live. The host now sends the original requested capture level into `sequence.step()` so Stop works, including a phone-held capture until it is released. Tests cover that cancellation and paused still metadata.

Separate existing follow-up: an active sequence can retain state across aircraft changes because the feature outlives its selected aircraft. Closing/reopening only a Camera tab is intentionally safe and tested; changing aircraft during a 360 has not been redesigned here.

No commit, publication, renderer/ground-contact edit or server start/stop occurred.

## Latest correction: reload retains a mod aircraft and its saved location

`createFlightSimSession` now awaits `ensureModsSession()` before reading the saved flight and saved aircraft choice. Both validate IDs against the live catalogue; previously, an installed drone had not yet been re-registered when its save was checked. That rejected the snapshot and could normalize the selection to Cessna, restarting at the configured start location.

The regression first failed on the old ordering, then passed after the change. The focused resume/store/mod/lazy-gate suite passes 29 tests. A broader run passes 87 and has two unrelated failures (the bug-report field query and a combat scene mock); a separate in-memory baseline transform removing only this fix reproduces both failures. Typechecking retains the same three pre-existing errors.

Actual WebGPU browser reloads passed for Mini 2 and Air 2S in an isolated profile. Each retained its selected aircraft, saved latitude/longitude and heading, paused state, HD model and 120 Hz physics. The configured start position was deliberately changed to a different location before reloading. Altitude changed only by the existing terrain/native restoration adjustment, about 0.223 m. Manifest requests preceded feature application, and neither JPEG nor capture was requested. Browser logs, the validated custom checklist and feedback launcher are in ignored `fly/build/drone-reload/`.

Nothing was committed or published. The pilot's existing server on 7470 stayed running; no server, renderer or ground-contact changes were made.

## Latest correction: Momentum camera roll and stopping

- `aircraft/src/cameraFollow.ts` now eases wrapped heading and pitch separately, then constructs a zero-roll frame. Full quaternion slerp could bank during combined heading/pitch changes. The travel-pitch limit is exactly ±85°.
- Direction confidence fades between 1.2 and 0.9 m/s. Below 0.9 m/s the target holds while the view finishes easing toward it, including when the attitude indicator hides velocity markers at 3 fps. Horizontal speed controls heading confidence independently, so vertical motion keeps the last reliable heading. Mode entry eases from level instead of snapping to the velocity frame.
- `createPlaceholderAircraft.ts` keeps the orbit parent level in heading, combines follow/orbit pitch in the camera offset, and limits the final view to ±85°. This removes the separate sideways-orbit roll and past-the-pole inversion that a tilted parent could cause.
- 67 focused tests passed, including actual Babylon camera transforms in both scene handednesses, vertical reversals, zero-roll intermediate frames, hover noise, finishing a turn after stopping, phone integration and the lazy gate. Typechecking has only the same three pre-existing errors.
- Live WebGPU Mini 2 climb, descent and both stops: 1,124 observed frames, maximum camera pitch 85°, right-vector vertical component below 6e-17, and maximum per-frame direction change near stopping about 0.016°. The combined browser run's subsequent Test-tab navigation timed out; a fresh-profile check separately verified the actual Test tab and Start button. The shared screenshot runner also captured the final climb/descent views; both were inspected and had zero camera roll. Logs, screenshots and the custom checklist are in ignored `fly/build/momentum-camera/`.

No commit, publication, renderer edit, ground-contact edit or server start/stop occurred.

## Latest follow-up: modular cameras and DJI inputs

Everything below remains local and uncommitted. No server was started; the pilot's server on 7470 remained running.

- The Mini 2 checkout is now `DJI-Mini2` / `0SFS/DJI-Mini2`, with mod ID `0sfs/dji-mini2`. The JSBSim aircraft ID/XML remain `dji-mini-2`; the old dev-root slug remains an alias. Both downloaded HD models live in `3d_model/HD`; generated runtime GLBs remain ignored local reference files. Nearby HD ↔ procedural zoom transitions, aircraft-sized zoom limits, Momentum defaults, shared `rotating-prop` and `camera-gimbal` components were verified in the earlier follow-up.
- New local packages/mods: `drone-camera`, `DJI-FC7303`, `DJI-FC3411`. The shared package owns WebGPU capture, EXIF/XMP, zip, optical flow and spherical sequencing. Model mods own sensor/lens/exposure/metadata data. Each airframe selects its camera ID and retains its motion/gimbal mount limits. The data registry imports only profile types; the feature supplies the runtime builder on apply.
- DJI aircraft declare centered climb-rate control and W/S pitch/translation, Q/E lateral, A/D yaw, Shift/Ctrl climb/descent. Controls → Keyboard axes edits all four key pairs live and swaps occupied assignments. The runtime, HUD hints and stored profile normalization share those mappings. Planes retain throttle control. The new CLIMB lever shows m/s, returns to altitude hold on release/cancel/blur and never integrates motor throttle. Phone sticks say CLIMB/DESCEND and FORWARD/BACK.
- A real still exposed two earlier defects: the CommonJS JPEG encoder tried to use Node Buffer in a browser, and the render target had no mesh list or submitted draw commands at readback. The local `jpeg-js` encoder now returns Uint8Array when Buffer is absent. Camera capture selects the loaded scene, flushes WebGPU draws before copy/flow, strips GPU row padding, flips render-target rows upright and uses the sensor aspect ratio. The host excludes its own airframe and orbit-compass helpers through a generic visibility callback. Renderer and ground-contact source were not edited.
- Final browser verification: WebGPU on Chrome, isolated profile, both local camera dependency trees, A/D yaw and Q/E lateral, centered Shift/Ctrl, live pair reassignment/swapping, no camera runtime at startup/add and no JPEG encoder until shutter. The exported Mini 2 still is 4000×3000 with FC7303 metadata and visible scene detail, inspected upright. `jpeg-js` remains in optimizeDeps.include. Air 2S apply and its +6.0 m/s lever were verified by the screenshot run.
- Final focused suite: 174 tests passed. Browser and Node encoder round trips passed. Typechecking still reports the same three existing errors in aircraftAnimation, terrainLightPlugin and engagement. Control/phone screenshots and a verified custom Test tab with downloadable feedback are in the fly checkout's ignored `build/drone-controls/` artifacts.

Still open: full 360 sequences and actual Firefox/Galaxy Note 20 Ultra phone capture tests. Only a Mini 2 still was captured during this follow-up. No commit, push, publication, renderer edit or ground-contact edit occurred.

## Follow-up verification, 28 October 2026

The three tasks in the original resume prompt are complete. The stopping point and open-defect list below describe the earlier session.

- Browser verification used Chrome 156 with WebGPU and an isolated profile against the pilot's existing server on port 7470. A saved GitHub Mini 2 was removed through Mods, then This computer added `local:0SFS/DJI-Mini-2`. The three local manifests loaded without the feature or JPEG encoder. Confirming Mini 2 fetched `droneFeature.ts` and `/dev-mods/dji-mini-2/jsbsim-data/aircraft/dji-mini-2/dji-mini-2.xml`. Native physics logged 120 Hz, all nine procedural meshes were present, and the flight resumed without a reload or runtime exception. No `jpeg-js` request occurred. This check did not change the pilot's browser storage.
- Resuming first exposed a missing HUD instrument-band entry for mod aircraft. `readingTints` now leaves readings untinted when no limits are declared. Ten instrument-band tests pass.
- The existing shared error viewer was preserved. A browser-injected JSBSim download failure showed the actual “Could not change aircraft” dialog, its file tabs, and a working settings search. Download all contained all 22 displayed files. The panel and switch overlay use the same viewer; 72 focused panel, overlay, parser, zip and viewer tests pass. Aircraft imports no fly drone zip helper.
- Sequence capture now shares the still pose builder and reads flight state at the shutter: real MSL, AGL and airframe attitude. Unknown terrain retains the zero-AGL fallback. The regression checks 609.6 m MSL over 200 m terrain produces 409.6 m AGL, plus unknown terrain.
- The host removes `°` before truncating `engine.blocked`. The phone integration regression validates the refusal through the protocol parser. The complete outgoing heartbeat is measured as UTF-8 JSON; above 1800 bytes it drops only `engine.orbs`, without mutating host status. Boundary regressions cover 1799, 1800 and 1801 bytes. `MAX_MESSAGE_BYTES` remains 2048.
- Final drone, mod-session, instrument-band and aircraft-setting checks: 33 passed. Phone integration and lazy-gate checks: 24 passed. RC session checks: 64 passed. Typechecking still reports only the three previously recorded errors in `aircraftAnimation.ts`, `terrainLightPlugin.ts` and `engagement.ts`.

No still or complete 360 sequence was taken. `jpeg-js` remains in `optimizeDeps.include`. Firefox and Galaxy Note 20 Ultra manual capture checks remain open. No commit, publication, second dev server, renderer edit or ground-contact edit was made by this follow-up.

## Model and zoom follow-up

The checkout is now `DJI-Mini2`, added as `0SFS/DJI-Mini2`; its internal aircraft/JSBSim ID remains `dji-mini-2`. Downloaded models for Mini 2 and Air 2S are organized under their respective `3d_model/HD/` directories. Each `npm run prepare:hd` creates a centered local runtime GLB in metres. Both models are by yigeshichen, CC BY-NC 4.0, credited on the HD choice and ignored by Git.

Auto no longer remounts procedural geometry on every camera refresh. The model loader owns its disposer and switches cleanly between HD below 15 m and procedural geometry farther away. Model paths resolve relative to the installed mod's base URL. Camera defaults now come from each aircraft: Mini 2 minimum 0.25 m; Air 2S minimum 0.35 m; both near clip 0.01 m. Explicit Camera settings take precedence.

WebGPU browser verification used the existing server: added both mods without feature, model or JPEG fetches, applied each HD model, zoomed past 15 m to nine procedural meshes, retained the same procedural mesh through repeated wheel gestures, and zoomed back to HD at the declared minimum. Mini 2 showed 59 HD meshes and Air 2S showed 34, without runtime exceptions or JPEG requests. Camera/model/settings/lazy-gate tests: 60 passed.

## Camera follow, props and gimbal follow-up

Aircraft → Camera now offers Aircraft / Momentum / Planet through `osfs.camera.chaseFrame`. Defaults are declared on aircraft data: planes use Aircraft; Mini 2 and Air 2S use Momentum. Momentum follows actual ground-relative travel heading and climb/descent angle without roll, smooths direction changes, holds its last reliable direction at hover, and retains heading during nearly vertical motion. Planet stays level and north-oriented. Explicit pilot overrides remain effective.

`rotating-prop/` and `camera-gimbal/` are local shared component packages with declarative mod manifests, not published repositories. C172 declares a rotating-prop dependency; Quadcopter declares both. C172 and drones share the same anti-strobing threshold, blur hysteresis, and rotation step. Both HD assets now have declared rotor pivots; the procedural models have blades too. Drone hover RPM was then a visual estimate from thrust, explicitly not motor telemetry; it is now each rotor's simulated speed (see the top section). Rotor phase stops with simulated time.

The shared gimbal component handles sensor stabilization, slew limits, visible HD camera pivots, and restoring an adopted cockpit camera to its original parent, position and rotation. `npm run prepare:hd` reads the rotor and gimbal mapping JSON beside each original model. Keep those mapping files; generated GLBs remain local reference files.

WebGPU browser checks passed for both drones: Momentum default, all three live follow controls, four spinning/blurred rotors while running, frozen phase when paused, visible gimbal pitch movement, and HD/procedural/HD zoom transitions. Adding the mods still fetches no drone feature, HD GLB or JPEG encoder. Focused regression suite: 148 tests passed. The only typecheck errors remain the three previously recorded errors. Screenshots and a validated Test-tab feedback link are under fly's ignored `build/camera-components/`.

No commit, push, publication, new dev server, renderer edit or ground-contact edit was made for these changes.

## Where the work stopped

The four declarative mods exist on disk and are not published. The lazy fly chunk exists. Unit tests for the drone modules, the mod session, aircraft setting choices, and local mod resolution passed. Nobody has flown a drone in the browser.

The last failure the pilot saw was:

`Failed to load JSBSim asset aircraft/dji-mini-2/dji-mini-2.xml from https://0sfs.github.io/DJI-Mini-2.`

The Mods tab now has GitHub and This computer. This computer was not tried in the browser. The installed copy, if it is still there, is the GitHub one and will keep asking `0sfs.github.io` until it is removed and added again with This computer.

The pilot’s dev server was already running from `cd ~/gh/0sfs/fly && npm run dev` on port 7470. Do not start a second one. Do not leave a server you started. Do not copy API keys out of that server’s local URL.

## Architecture that stays

- Four repos, none in `BUNDLED_REPOS` or the official fleet: `0SFS/Quadcopter` (`0sfs/quadcopter`), `0SFS/DJI-Quadcopter` (`0sfs/dji-quadcopter`), `0SFS/DJI-Mini-2` (`0sfs/dji-mini-2`), `0SFS/DJI-Air-2S` (`0sfs/dji-air-2s`). Data only. WebGPU, Babylon, the controller, capture, and optical flow stay in the lazy fly chunk. A mod cannot touch WebGPU, and remote JavaScript is unsandboxed.
- `applyDynamicMod` runs only on the root record. It walks dependencies first. Two aircraft mods share family id `dji-quadcopter`; `registerAircraftMod` appends variants.
- `sdk.run()` is the only integrator. Physics is 120 Hz only while a drone is selected, then the previous `osfs.physics.rate` (`"30" | "60" | "120"`) is restored. `droneRateRestore` is that union; both `parameters.get` sites are cast.
- Do not import `droneGate` from `@0sfs/aircraft`. The mesh hook is `aircraft/src/droneMeshHook.ts`; the lazy feature registers it with `setDroneMeshHook`.
- Removing the last quad aircraft disposes the feature. If that aircraft is selected, switch to `cessna-172`. `isDroneAircraftId` stays true for the hardcoded `dji-mini-2` and `dji-air-2s` after uninstall, so removal compares the removed id.
- WebGPU only for this pipeline. Do not edit `foss-earth` renderer WebGL fallbacks or `@0sfs/ground-contact`.
- Engine kind `"none"` is not an `EngineKind` (`"piston" | "turbine"`). The engine monitor uses an IIFE so `"none"` becomes `"piston"`. A second `getFdmProfile().engine` in a ternary does not narrow.
- `AircraftFamilyId` stays the built-in union. Gallery state, `onFamilyChange`, and `FuelPanel` `familyId` are `string`. Catalog variant assignments use `as unknown as`.
- Phone stick Y is down-positive. Quad vertical and pitch use `-Y`. The phone quad UI loads through `loadQuadcopter()` only when `status.drone.available`.

## Decisions that look like bugs

**Vite must see `droneFeature.ts` even when the mod is off.** `droneGate.ts` is on the startup path. Vite resolves `import("./droneFeature")` at compile time. The browser fetches the chunk only when `loadDroneFeature()` runs. A missing file stops `/fly/` from compiling. The gate’s source scan is case-sensitive and does not match the name `loadDroneFeature`. Scanned startup files must not contain the substrings `droneFeature`, `flowWgsl`, `captureFrame`, `droneController`, or `PhoneQuadcopter`.

**The model's +X is left.** Aircraft models use glTF's right-handed frame: +X left, +Y up, +Z forward. A structural hub converts as X = cg.y − y. Code that assumes +X right draws the aircraft mirrored. If both sides of a test share that assumption, the test still passes. SPEC section 4.8 has every frame and conversion.

**Props turn by `-spin`.** A rotor's `spin` is about body +Z, which points down, so +1 is clockwise seen from above. rotating-prop turns by `+direction` about its axis, which for the drones points up. About an upward axis, a positive turn is counter-clockwise from above. So `droneFeature.ts` passes `-spin`, and the manifests' visual `direction` is `[-1, 1, 1, -1]` for front-left, front-right, rear-left and rear-right. Making `direction` equal `spin` mirrors the props' turning.

**Position hold signs pull toward the anchor.** `planDroneCommand` in `droneController.ts` uses `vN = -kPos * northError`, `vE = -kPos * eastError`, `vD = +kPos * (alt - anchor)`. The spec’s first draft had the opposite signs. Those are positive feedback. Do not restore them. The spec’s section 4.4 now matches the code.

**`jpeg-js` encodes a still and nothing else.** It is CommonJS (`module.exports = { encode, decode }`) at `jpeg-js/`, linked from `fly/package.json` as `file:../jpeg-js`. A static named import was rewritten to raw `@fs` CommonJS the browser cannot execute, and it also pulled the library in when the drone chunk loaded. `capture()` now does `await import("jpeg-js")` and calls `encoder.default.encode`. `optimizeDeps.include` still lists `jpeg-js` so the dev server prebundles it and the first shutter does not full-reload the page. The browser does not fetch it until encode. Taking it out of `optimizeDeps.include` is closer to just-in-time and can reload the flight on the first picture. Leave that until a still is tested. `proxy()` does not encode JPEG.

**Settings choices are a snapshot.** `osfs.aircraft.id` and `osfs.aircraft.generation` have `dynamicChoices: true`. That only relaxes stored, URL, and force validation. `setMany` still checks the live choice list, which is why apply said `"dji-mini-2" is not one of Cessna 172 Skyhawk, G1, G2, G3, F-35B` after the catalog already had the aircraft. `syncAircraftSettingChoices` in `registerFlightSettings.ts` rebuilds both lists from `AIRCRAFT_CATALOG` and `AIRCRAFT_FAMILIES` and subscribes to `subscribeAircraftCatalog`. Do not call `setChoices` with an empty catalog. `registerAircraftFleet` does not bump the catalog; `registerAircraftMod` and `unregisterAircraftMod` do. Variant labels are the choice labels, so Mini 2’s generation id is `mini-2` and its aircraft id is `dji-mini-2`. `getAircraftDefinition` returns the Cessna when the id is missing, so an unknown id can be rewritten quietly. Tests that use `createSettingsRegistry` without FOSS Earth parameters throw on `map.focus.refineFrom`; use `getAppSettings()`.

**A fetch rewrite does not change the stored base URL.** `devQuadFetch` can redirect a GitHub Pages GET to `/dev-mods/<slug>/`. `loader.ts` `baseUrlOf` is the directory of the URL recorded on the source, and `downloadModJsbsimFiles` uses global `fetch` of `${baseUrl}jsbsim-data/${file}`. That is why the manifest could come from disk while the XML error named `https://0sfs.github.io/DJI-Mini-2`. The fix is a real source choice, not a wider rewrite.

## GitHub or this computer

Implemented 28 October 2026. Not browser-tested.

- The Add mod form shows GitHub and This computer when `import.meta.env.DEV` (`ModsTab` passes `allowLocal`).
- This computer submits `local:<owner/repo>`. `normalizeModSource` maps that to canonical key `local:<owner>/<repo>@<ref>` and manifest URL `/dev-mods/<repo-lower>/0sfs-mod.json`.
- A mod loaded that way loads its dependency shorthands with the same `local:` prefix (`localDependency` in `mods/src/loader.ts`). Mini 2 then pulls DJI-Quadcopter and Quadcopter from `/dev-mods/` too.
- `baseUrl` becomes `/dev-mods/dji-mini-2/`. JSBSim then requests `/dev-mods/dji-mini-2/jsbsim-data/aircraft/dji-mini-2/dji-mini-2.xml`. That path returned HTTP 200 from the running dev server.
- GitHub stays `https://0sfs.github.io/<Repo>/0sfs-mod.json`. Those four repos 404 until published. Do not publish them from this task.
- A saved source string is the identity. `0SFS/DJI-Mini-2` and `local:0SFS/DJI-Mini-2` are different mods. Remove the GitHub copy, then add the local one.
- Tests: `mods/tests/normalizeModSource.test.ts` and `mods/tests/loader.test.ts` (7 passed).

## Open defects

1. **Browser apply of Mini 2 from This computer.** Reload `/fly/` if the dev server picked up the Mods tab change. Remove the GitHub Mini 2. Choose This computer. Add `0SFS/DJI-Mini-2`. Aircraft tab, apply Mini 2. Expect 120 Hz, the procedural mesh, `dji-mini-2.xml` from `/dev-mods/`, no `droneFeature` request until apply, and no `jpeg-js` request until a still or a 360. Verify the click path in the browser, not a screenshot of the form.

2. **Aircraft-change error files.** Built in the debug session on 28 October 2026. Do not rebuild it. `aircraft/src/issueReport.ts` splits the report. `IssueReportView` is the tabs, the line search, Copy index, Download this file, and Download all. The panel and `createAircraftSwitchOverlay.ts` both use it. `index.txt` is the only file to read first. `index.json` is the same facts as data: `failure.file` and `failure.line`. A file longer than 24 lines stays closed until a needle or a search names a line. A zip of the report still cannot include a Vite transform error; that line exists only in the dev-server log.

3. **Sequence capture AGL is always 0.** In `droneFeature.ts` the sequence callback sets `aglM: step.terrainMeters === null ? 0 : 0`. The still path uses real AGL.

4. **Refusal text can fail the phone protocol.** `360° capture requires WebGPU.` contains `°`. `engine.blocked` must match `/^[A-Za-z0-9 ,.'-]{1,60}$/`. Strip the degree sign before that field.

5. **Heartbeat size.** If the encoded status would exceed 1800 bytes, omit `engine.orbs` first. `MAX_MESSAGE_BYTES` stays 2048. Not implemented.

## Optical flow and capture notes

Coarse and fine need separate uniform buffers. Coarse `flow_in` is a zero buffer. The fine shader must not index `flow_in[fullIndex]` when `scale > 1`; sample the coarse result at `pixel/2` and multiply by 2. Babylon’s `GPUDevice` bind-group layout is branded; cast `getBindGroupLayout(0) as unknown as GPUBindGroupLayout`. `UniversalCamera` has no `aspect`; set it through a cast. A render-target constructor pushes onto `scene.customRenderTargets`; the capture rig splices that target back out. Capture cannot `await` inside the synchronous physics callback. The sequence stays in `capturing` until the promise resolves. HUD buttons mount lazily: `mountButtons` returns null until `[data-drone-actions]` and the flight bar exist, and `afterView` retries because a later HUD recreate drops them.

## Checks already run

From `fly/`: `npx vitest run --configLoader native src/flight/drone src/flight/mods/modSession.test.ts` (6 files, 7 tests) and `flightParameters.test.ts` (14 tests, including a mod-added aircraft). From `mods/`: normalize and loader tests, 7 passed. `npx esbuild` parsed `rc/src/PhoneController.tsx`. Drone-related `tsc -p tsconfig.app.json` was clean after the casts. Pre-existing errors left alone: `aircraft/src/aircraftAnimation.ts` nullable quaternion, foss-earth `terrainLightPlugin.ts` `_currentEffect`, `engagement.ts` unused `Place`. Piping `tsc` to `head` hides the compiler’s exit code.

Manual checks that remain the pilot’s: M5 MacBook Air Firefox, and Galaxy Note 20 Ultra 5G Chrome, as section 10 of the spec. Browser verification of the flight UI was never finished.

## Do not

- Commit or publish unless asked. Do not publish the four drone repos.
- Start a second dev server, or leave one running.
- Edit the foss-earth renderer or `@0sfs/ground-contact`.
- Put the drones in the built-in fleet or `BUNDLED_REPOS`.
- Import `droneFeature.ts` from the startup graph, or prefetch it.
- Import `jpeg-js` except inside `capture()`.
- Restore the unstable position-hold signs.
- Treat `dynamicChoices` as permission for `setMany` to accept an id that is not in the live choice list.

---

## Resume prompt

Copy from here.

I am continuing the 360° drone work in `/Users/felg/gh/0sfs`. Read these two files first and follow them:

- `docs/proposals/360-drone-session-2026-10-28.md` — what the last session learned and what is still open
- `docs/proposals/SPEC-360-DRONE.md` — the contract, including the corrected position-hold signs, the JPEG import, and the GitHub versus This computer add path

Do not commit or publish. Do not publish `Quadcopter`, `DJI-Quadcopter`, `DJI-Mini-2`, or `DJI-Air-2S`. Do not edit the foss-earth renderer or `@0sfs/ground-contact`. Do not start a second dev server: I already run `cd ~/gh/0sfs/fly && npm run dev`. Do not leave a server you started. WebGPU is the only backend for this pipeline.

Work in this order. Finish and verify one before the next.

1. Confirm a local Mini 2 can be applied. If `0SFS/DJI-Mini-2` is already installed, that copy is GitHub and still asks `https://0sfs.github.io/`. Remove it. In the Mods tab choose This computer and add `0SFS/DJI-Mini-2`. Apply Mini 2 from the Aircraft tab. The JSBSim XML must come from `/dev-mods/dji-mini-2/jsbsim-data/aircraft/dji-mini-2/dji-mini-2.xml`. Physics goes to 120 Hz. The airframe is the procedural mesh. The network does not request `droneFeature` until apply, and does not request `jpeg-js` until a still or a 360. Verify that path in the browser.

2. Skip the error screen. It already splits the report into tabs. Do not edit `aircraft/src/issueReport.ts`, `aircraft/src/IssueReportView.tsx`, `AircraftSelectionPanel.tsx`, or `createAircraftSwitchOverlay.ts`.

3. Fix sequence capture AGL, which is hardcoded to 0 in `fly/src/flight/drone/droneFeature.ts`. Strip `°` from the WebGPU refusal before it is placed in `engine.blocked`. If the encoded heartbeat would exceed 1800 bytes, omit `engine.orbs` first.

Leave `jpeg-js` in `optimizeDeps.include` until a still is actually taken. Do not restore the old position-hold signs. Do not put drone code on the startup graph.
