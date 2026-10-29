# Drone realism audit: controls, state transfer and timing

> **Partly history.** The flight-model findings here describe the drone before the 28 October 2026 rebuild. The force controller, the `fcs/drone-*` properties, the estimated RPM and the private wind have since been replaced by the rotor plant and flight controller in [`multirotor/`](../../multirotor/README.md). The [flight-model roadmap](360-drone-flight-model-roadmap-2026-10-28.md)'s stage statuses and placeholder inventory say which findings are fixed and which are still open. Findings outside the flight model are unaffected.

28 October 2026. Read-only implementation audit, supplementary to the main drone realism audit. Findings describe the current source; they are not claims of validation against a physical DJI aircraft. No flight-physics implementation changed during this audit.

## What happens when a C172 becomes a Mini 2 at 100 knots

The switch intentionally preserves motion. `fly/src/flight/createFlightSimApp.ts:462` suspends the old session and captures its snapshot. Although the new SDK initially bootstraps with the new airframe's starting speed (`:512`), the prior snapshot is passed to the replacement session (`:527`) and restored after terrain/model preparation (`:4112`). `fly/src/flight/jsbsim/savedFlight.ts:203` copies the saved initial conditions across different aircraft. The field list in `ground-contact/src/safeFlightState.ts:160` includes position, roll/pitch/heading, N/E/D velocity and body p/q/r angular rates. That file was inspected only.

The Mini therefore retains roughly 51.4 m/s if the preserved horizontal velocity was 100 knots. IAS and ground speed are distinct; this unit conversion alone does not establish the exact restored ground velocity for a particular flight. It also retains the C172's attitude and angular rates. Throttle, trim, fuel and engine state remain those of the newly loaded aircraft (`savedFlight.ts:222`); the C172 engine and trim are not transplanted. Weather and view mode carry over, damage resets when the aircraft ID changes, and the new session uses a new clock.

The switch finishes paused (`createFlightSimApp.ts:3988`). On resume, a newly bound drone controller has no hold anchor. With neutral sticks it anchors position at the current location and immediately requests zero horizontal velocity (`fly/src/flight/drone/droneController.ts:68`). Its translational force is applied in NED independently of actual body tilt. Thus the body can begin braking before its attitude has physically tilted into the braking maneuver. The main physics audit covers that force-model defect; the switch is a reproducible way to expose it, rather than evidence that the new SDK starts at zero speed.

Acceptance: a C172-to-Mini switch at a recorded position/attitude/velocity should explicitly preserve or deliberately reset each documented state component, start paused, and expose those values in diagnostics. A high-speed transition must remain a physical initial-value problem if momentum is retained. A future reset-to-hover choice should be an explicit simulator convenience, not an undocumented speed clamp.

## Findings

### C1 — Phone drone payload bypasses control sharing and pilot-state neutralization

**Confirmed code path; high impact when a phone is flying.** `createFlightSimApp.ts:3638` obtains the lease-checked generic phone controls and blends them with desktop controls. It then neutralizes stick/pedals when the simulated pilot is out (`:3643`). But `:3718` obtains a separate raw drone payload, and `:3726` passes it to `currentDroneCommand`. `fly/src/flight/drone/droneGate.ts:216` prefers that payload wholesale over the blended/neutralized desktop record. Consequently local controls and the configured blend priority do not affect drone flight while a drone payload is present; the phone payload can also bypass the pilot-out neutral stick values. This is a dispatch bug, not a motor-model approximation.

Remedy: produce one authoritative quad command after input authority, sharing and pilot-state handling. Retain auxiliary camera/capture/model fields separately from flight axes. Acceptance: with phone input held, both priority settings produce the specified mixed flight axes; pilot-out sends neutral translational/yaw/climb commands for both input sources; camera commands follow their separately documented policy.

### C2 — Drone payload can outlive the generic phone loss boundary until the watchdog runs

**Confirmed mismatch; bounded normal-window impact.** `rc/src/createPhoneControlSession.ts:711` neutralizes generic phone controls when the five-second hold expires. Its `getDroneCommand` (`:730`) checks only ownership, not `holding()`. The 50 ms watchdog eventually revokes ownership and clears the drone payload (`:593`, `:282`), but before that timer runs the flight loop can still consume the old drone command. The larger five-second last-command hold is itself an explicit existing policy (`rc/src/protocol.ts:12`), not a newly discovered implementation accident.

Remedy: apply the same immediate freshness/hold result to generic and drone command accessors. Separately decide whether five seconds of continued climb/translation is the intended DJI link-loss simulation. Acceptance: an expired frame cannot drive any quad axis even when physics executes before a delayed heartbeat callback; reconnection needs the documented fresh authority state.

### C3 — The generic master autopilot can appear engaged while quad flight ignores its output

**Confirmed dispatch mismatch; high impact if enabled.** `createFlightSimApp.ts:3647` runs the arbiter and forms `commanded` from `ap.controls` (`:3653`), but the quad command at `:3725` uses `selected`, which precedes arbitration. The resulting aerodynamic surface/throttle writes do not control the zero-aero, zero-propulsion drone. There is no drone-specific engagement block in the local `autopilotBlockReason` (`:1125`).

Remedy: either provide an explicitly supported quad autopilot translation into velocity/climb/yaw targets, or mark the generic plane autopilot unavailable for this vehicle. Acceptance: an engaged mode must demonstrably control the intended quad axis and report its owner, or engagement must be rejected with a clear reason.

### C4 — Plane auto-trim and keyboard rate assist are not quad flight-controller models

**Confirmed applicability issue; control-feel effect depends on selected settings.** Plane auto-trim still runs at `createFlightSimApp.ts:3672`/`:3692` using aerodynamic surface authority estimates in `controls/src/autoTrim.ts:50`. Its trim writes do not alter quad force commands. It can change visible trim/UI without performing the implied stabilization on the quad. Keyboard Assist (`controls/src/keyboardStickResponse.ts:60`, `:137`) closes a body-rate PID around commands that the drone subsequently interprets as translation-speed commands. This layers a plane stick-assist controller over the drone's own velocity/attitude loops and can fight autonomous braking/leveling. Default Smooth mode is only a configurable input filter and does not have that body-rate feedback conflict.

Remedy: define applicability and units per vehicle control mode. Keep useful normalized deadzone/expo/filtering; disable or replace surface auto-trim and plane body-rate keyboard assist for DJI velocity mode. Acceptance: neutral released keys reach a zero velocity request under every offered mode; unsupported assists are unavailable; direct/gamepad/phone paths have documented response and sign parity.

### C5 — Legacy gamepad throttle does not command climb/descent

**Confirmed branch mismatch; medium impact for legacy input.** The profile-driven path maps `flight.climbRate` and converts an absolute throttle axis to a centered climb command (`controls/src/flightInputManager.ts:629`). The legacy path computes climb from keyboard/lever (`:533`), then reads a gamepad throttle into `persistent.throttle` (`:779`). It does not convert that legacy axis into `climbCommand`, and the drone gate ignores throttle in favor of `desktop.climb` (`droneGate.ts:242`). Thus the legacy gamepad throttle can visibly move a value that does not drive the drone vertically.

Remedy: route both gamepad implementations through one centered vertical-axis semantic; decide explicitly whether absolute hardware levers are supported. Acceptance: equivalent axis samples on standard, legacy and custom profiles request the same bounded climb/descent, with center and disconnect commanding zero.

### C6 — Aircraft/teleport controller state is not tied to every native state reset

**Confirmed lifetime mismatch; medium/high impact on resets.** `createDroneController` owns hold/gimbal state and exposes `reset` (`droneController.ts:183`), but `droneFeature.ts:126` recreates the controller only when the aircraft ID changes. The feature is retained behind its loaded gate. Session teardown and teleport native resets do not call that controller reset. `createFlightSimApp.ts:2769` resets plane auto-trim after teleport, but no drone reset is called. A same-model session rebuild or teleport can retain the old geographic hold anchor until non-neutral commands modify it; neutral hold then commands travel toward the pre-reset anchor, bounded by the position-hold velocity setting. Capture-sequence state similarly survives airframe changes because its reset is only in feature disposal (`droneFeature.ts:304`).

Remedy: add an explicit vehicle/session/reset generation boundary that resets or deliberately restores all TS flight-controller, wind, motor, gimbal and capture state alongside the native integrator. Acceptance: teleporting a neutral hovering drone does not command return to the old coordinates; reapplying the same drone produces the documented state; changing aircraft during a sequence aborts/finishes it according to an explicit policy and never captures with mixed camera profiles.

### C7 — Controller and capture clocks advance on attempted, not necessarily accepted, physics steps

**Confirmed ordering; fault-path impact.** The fixed loop calls `applyInputs()` before its pre-step validity checks and `sdk.run()` (`fly/src/flight/physics/fixedStepLoop.ts:132`). That callback invokes `droneFeature.beforeStep`, which mutates hold, gimbal, wind, sequence elapsed time and `motorSimTime` (`droneFeature.ts:166`–`:217`). On a later invalid/refused native step, `fixedStepLoop.ts:184` can restore the JSBSim snapshot, but those TypeScript states are not rolled back. The ordinary 120 Hz cadence is real, but the promise that only accepted physics time advances all subsystem state is incomplete.

Remedy: separate planning from accepted-step state commit or snapshot/revert the external controller state along with native state. Defer capture side effects until the step boundary is accepted. Acceptance: force a failed run/invalid candidate and verify identical native state, controller anchor, wind RNG, mount angles, motor clock and capture counters before/after rejection; resuming must not replay or skip a capture.

### C8 — Input filters run in render time, not accepted simulation time

**Confirmed design approximation; medium control-feel impact under stalls.** `createFlightSimApp.ts:3557` caps render delta at 100 ms and calls `inputManager.poll(deltaSeconds)` once per render (`:3605`), including while the simulation is paused. Physics catches up at most six 120 Hz steps/50 ms per update (`fly/src/flight/physics/physicsRate.ts:51`), discarding excess time. All catch-up steps use the same already-filtered input. Keyboard Assist integrals and smoothing therefore depend on render timing, and can advance during paused render requests, while the native vehicle is held.

Remedy: specify which filters are user-interface wall-time behavior and which are flight-controller dynamics. Run the latter per accepted fixed step or rebase them when paused. Acceptance: compare recorded input at 30/60/120 render FPS and after a stall with identical accepted physics duration; bound differences in commanded axes and vehicle response. Pause/resume must not introduce an undocumented assisted-control jump.

### C9 — Controller AGL is one render frame old and uses a separate terrain sample

**Confirmed data-age issue; landing effect needs targeted measurement.** `latestTerrainMeters` starts null (`createFlightSimApp.ts:961`), is passed to each physics step (`:3726`), and is refreshed only after the loop from the interpolated display position (`:3800`). All catch-up steps use the preceding frame's sample even if the aircraft moves across terrain. Native terrain/contact correction runs earlier through a separate path. The controller's landing-specific AGL logic can therefore disagree with the native ground state, especially just after loading, fast motion or terrain discontinuities.

Remedy: make the step consume a terrain observation at the current native state with explicit age/reference-frame metadata, or intentionally share the authoritative contact terrain. Acceptance: above flat terrain and a step/slope, log controller AGL and native contact height at each accepted step; verify no spurious climb/descent transition when the previous render position crosses an edge.

### C10 — Flight control uses perfect native state, not simulated GPS/IMU observations

**Confirmed realism limitation, not a numeric integration defect.** `droneController.ts:202` reads native attitude, rates, N/E/D velocity, latitude/longitude and altitude directly. There is no sensor observation latency, noise, bias, dropout or estimator in that path. Optical flow supplies capture alignment/gating (`droneFeature.ts:188`, `:197`) rather than a fused flight-state estimate. The DJI-like behavior is therefore a controller around ground truth, not a validated DJI sensor/autopilot stack.

Remedy: retain ground truth for diagnosis but expose a separate sensor/estimator boundary before the flight controller. Parameterize observations and validate their units, timing and failure behavior. Acceptance: deterministic seeded sensor replay; GPS loss/velocity bias/IMU drift exercises; controller reacts to observations while diagnostics retain the truth comparison. Do not imply manufacturer-equivalent estimator accuracy without evidence.

### C11 — Weather controls and drone wind are separate inputs

**Confirmed source split; parent environment audit owns the physical remedy.** The Weather tab writes native atmosphere/wind properties (`createFlightSimApp.ts:2695`). The drone controller receives a separate seeded OU wind object (`droneController.ts:200`, `droneWind.ts:34`), initialized/reset to zero on profile binding. The controller's drag calculation uses that object rather than reading native wind. Native atmosphere may still affect reported airspeed, so the displayed airspeed and drone drag can respond to different winds.

Acceptance: a known steady weather wind must appear consistently in native air-relative velocity, drag, controller observations and logged ground-relative velocity; adding gusts must not double-count two unrelated sources.

### C12 — The prior plane physics-rate preference is lost across a drone reload

**Confirmed persistence/lifetime mismatch; numerical-policy impact.** `createFlightSimApp.ts:409` stores the prior physics rate in a module-local `droneRateRestore`. Applying a drone writes the saved settings parameter to 120 (`:495`); normal settings writes persist it. Reloading clears that module-local backup. The restored drone session then records the now-saved 120 as the prior rate (`:700`). Leaving the drone restores 120, even if the pilot had chosen 60 before first applying it. The native timestep and loop still agree, so this is not an integrator timestep mismatch; it loses the intended per-flight override/pilot preference.

Remedy: separate the vehicle-required active rate from the saved pilot rate, or persist the original override state with explicit scope. Acceptance: select 60, apply drone (active 120), reload drone (active 120), switch to C172 (active/saved 60); repeat starting at 30 and 120 and with a failed drone apply.

## Behavior that is implemented as intended

- A/D maps yaw and Q/E lateral translation through aircraft-specific defaults; choosing another key pair is supported (`controls/src/keyboardAxisBindings.ts:27`). Positive W/S pitch input becomes forward velocity, positive roll becomes right translation, positive vertical becomes climb (`droneController.ts:79`). This is DJI-style velocity mode rather than FPV direct motor throttle.
- The desktop climb lever springs to zero; blur and input ownership transitions clear it (`flightInputManager.ts:733`, `:870`). Zero vertical requests altitude/vertical hold rather than zero motor power.
- The fixed-step loop checks the native SDK timestep, runs JSBSim as the integrator, interpolates display state separately, discards pause backlog and skips the first resume delta (`fixedStepLoop.ts:69`, `:226`; `createFlightSimApp.ts:3557`). Six catch-up steps at 120 Hz is an explicit numerical budget, not proof of physical model accuracy.
- Native invalid-state and gross-contact guards pause/restore the simulation. They protect numerical continuity; their broad bounds (for example the absolute 1,500-knot guard) are not Mini 2 structural or performance limits.

## Suggested order

First make the command boundary authoritative (C1–C5), reset/commit state coherently (C6–C9), and document high-speed transition semantics. Those changes prevent interface/dispatch artifacts from being mistaken for aerodynamic behavior. Then validate the physical propulsion/force model and the observation model together. Keep current signs that correctly restore toward the hold anchor; none of these remedies requires returning to the old position-hold signs.
