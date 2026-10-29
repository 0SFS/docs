# Drone simulation realism audit — 28 October 2026

> **Partly history.** The flight-model findings here describe the drone before the 28 October 2026 rebuild. The force controller, the `fcs/drone-*` properties, the estimated RPM and the private wind have since been replaced by the rotor plant and flight controller in [`multirotor/`](../../multirotor/README.md). The [flight-model roadmap](360-drone-flight-model-roadmap-2026-10-28.md)'s stage statuses and placeholder inventory say which findings are fixed and which are still open. Findings outside the flight model are unaffected.

Status: local audit and repair backlog. No publication or commit. This document
does not authorize changing the flight model by itself. The separately requested
3D-propeller presentation correction is recorded below. The
[flight-model roadmap](360-drone-flight-model-roadmap-2026-10-28.md) turns this
audit's repair order into staged work with flight-lab gates.

## Assessment

The current drone is a JSBSim rigid body driven by an idealized velocity/position
controller. JSBSim integrates motion using the configured mass/inertia, gravity,
ground contact and externally supplied forces/moments. There are **no four
physical motor/propeller plants** in the drone XML. The controller decides an
aggregate force and torque; the RPM display works backward from that request.
It cannot report an exact underlying shaft state because that state does not
exist yet.

This is a useful capture/control prototype, but it is not currently a validated
Mini 2 or Air 2S flight-dynamics simulation. A 120 Hz integration rate, plausible
DJI speed settings, an HD model and working RPM circles do not establish that
fidelity. Some limitations are missing detail; several are concrete physical or
algorithmic errors even within the ordinary operating envelope.

This audit covers the local controller, motor estimates, XML/profile data,
input dispatch, session transitions, timing, weather/terrain integration,
sensing, capture, gimbal and presentation paths. It distinguishes source-proven
defects from deliberately simplified models and scenarios that still require
measurement. It cannot establish that every possible physical effect or latent
runtime defect has been found. It makes no numerical claim about proprietary
DJI firmware or unmeasured motor/propeller performance.

## Detailed evidence

- [Propulsion and rigid-body/controller audit](360-drone-realism-propulsion-audit-2026-10-28.md): P1–P5 physical errors, controller geometry/rates, motor/propeller/electrical omissions, mass/inertia/calibration, acceptance checks and the 100-knot calculation.
- [Controls, state and timing audit](360-drone-realism-controls-audit-2026-10-28.md): C1–C12 command authority, stale phone input, autopilot/assist applicability, reset/commit boundaries, gamepads, AGL age, timestep/preference behavior.
- [Sensors, camera and presentation audit](360-drone-realism-sensors-audit-2026-10-28.md): S01–S10 actual optical-flow defects, acquisition timing, motion/blur gates, truth-state sensing, gimbal constraints, camera calibration, remote state and audio.
- Environment/ground/validation findings E1–E8 and V1–V3 below complete the cross-component review. Some appendix findings describe the same underlying cause from different consumers; do not treat overlapping rows as separate fixes.

## The fast-switch experiment

Switching aircraft preserves position, attitude, translational velocity and
angular rates, while retaining the new airframe's own controls/engine. See
[`restoreSavedFlight`](../../fly/src/flight/jsbsim/savedFlight.ts#L195) and the
switch resume at
[`createFlightSimApp.ts`](../../fly/src/flight/createFlightSimApp.ts#L525).
A Mini 2 can therefore start with the C172's approximately 100-knot motion.
That is a legitimate deliberately unusual initial condition for testing the
new aircraft. The editor's instantaneous change of airframe/mass is not itself
a modeled physical event; its purpose is to establish the new initial state.
Resetting velocity to hover would conceal the flight-model weaknesses.

Three concrete defects explain why the response can feel physically wrong:

1. **Force follows the controller's desired direction immediately, rather than
   the aircraft's actual rotor axes.** A level drone and a drone rolled 90°
   receive identical world-directed force for otherwise identical inputs. The
   body is separately told to tilt toward that force. It can begin braking
   before tilting, or keep upward support in a mechanically impossible pose.
2. **The tilt limit clips drag along with the requested rotor force.** At high
   airspeed the model suppresses its own aerodynamic resistance. A thrust
   constraint must not remove a passive aerodynamic force.
3. **The requested force/torque does not have to be producible by four motors.**
   There are no shaft dynamics or power limits. The RPM display desaturates its
   estimated motor allocation, but the unconstrained force/moment request has
   already been delivered to JSBSim.

For an idealized level, neutral-control Mini 2 at 100 knots, zero wind and zero
vertical velocity, the current constants produce 128.61 m/s² of requested
velocity braking plus 130.20 m/s² of modeled drag. The combined 258.81 m/s² is
then clipped to 8.23 m/s² by the 40° tilt rule. These are calculations of the
current implementation, **not measured real Mini 2 drag or a replay of the
pilot's exact maneuver**. See the propulsion appendix for inputs and units.

This is not exclusively an extreme-speed problem: Air 2S full descent from rest
can request downward rotor force and derive a 180° roll target despite its
configured 35° tilt limit. The unilateral rotor-force constraint is missing.

## Repair catalogue

Priority 0: physically impossible or incoherent core behavior. Priority 1:
material command/sensing/collision correctness. Priority 2: calibration and
fidelity extensions after the relevant core path is coherent. These are an
implementation order, not estimates of effort or a claim that all effects
need equally detailed modeling.

| Priority | Source of unrealism | Evidence / detailed remedy |
| --- | --- | --- |
| 0 | World thrust is independent of actual roll/pitch; attitude follows afterward | Propulsion P1 |
| 0 | Tilt limiter removes part of aerodynamic drag | Propulsion P2 |
| 0 | Negative/downward rotor thrust and unbounded upward thrust are permitted | Propulsion P3; Air 2S descent and Mini climb-release examples |
| 0 | Torque/collective sent to JSBSim bypass feasible motor allocation; only displayed RPM saturates | Propulsion P4 |
| 0–1 | Landing/near-ground branches replace forces abruptly rather than modeling contact/thrust | Propulsion P5; E5 |
| 1 | No per-motor RPM, acceleration, propeller load or ESC state; assumed hover RPM is inverted afterward | Propulsion M1 |
| 1–2 | No validated rotor thrust/torque map versus RPM, density and inflow; no windmilling/rotor interactions/ground-effect model | Propulsion M2; E3 |
| 1–2 | No shared battery voltage/current/energy budget, motor thermal derating or failure constraints | Propulsion M3 |
| 1 | Weather wind and private drone wind disagree; density is fixed | E1–E2; Controls C11 |
| 1 | Euler attitude errors/heading rate approximate body-rate commands outside small angles | Propulsion controller C2 |
| 1 | Diagonal translation requests exceed nominal scalar speed by up to √2; rates and acceleration/jerk have no calibrated trajectory model | Propulsion controller C3 and D2 |
| 1 | Raw phone drone commands bypass the shared authority/blend/pilot-neutralization result | Controls C1 |
| 1 | Command expiry is not enforced at the same boundary for generic and drone phone commands | Controls C2 |
| 1 | Generic plane autopilot can appear engaged while its output does not drive the drone | Controls C3 |
| 1 | Plane keyboard body-rate assist is layered over a quad velocity controller; surface auto-trim has no corresponding authority | Controls C4 |
| 1 | Legacy gamepad throttle path does not map to the quad climb command | Controls C5 |
| 1 | Hold/capture/controller state is not reset coherently on all teleports, same-model sessions and aircraft transitions | Controls C6 |
| 1 | External controller/sequence clocks can advance when a native physics step is rejected | Controls C7 |
| 1–2 | Input filtering, flow timing and physical integration mix render, wall and simulation clocks | Controls C8; Sensors S02; E7 |
| 1 | Controller AGL comes from the preceding render frame rather than the current accepted native state | Controls C9 |
| 1 | No drone hull/rotor swept collision probes; one belly ground contact | E4–E5 |
| 1 | Optical-flow shader repeats one center gradient over its window, producing a rank-one structure tensor | Sensors S01 |
| 1 | Flow image intervals can be wrong and one displacement can be added repeatedly at 120 Hz | Sensors S02 |
| 1 | Sequence shutter image and metadata do not share an atomic native-pose acquisition snapshot | Sensors S03; actual error magnitude needs a moving-scene test |
| 1 | “Settled” uses angular rate without a complete translation/depth/exposure test | Sensors S04 |
| 1–2 | Ideal gimbal stabilization can exceed the combined mechanical stops and actuator capabilities | Sensors S06 |
| 2 | Flight control uses perfect native state; no GPS/IMU/range sensor noise, delay, estimator or loss behavior | Controls C10; Sensors S05 |
| 2 | Inertia, CG, drag, gains and motor calibration are assumed rather than experimentally identified | Propulsion D1–D2; E2 |
| 2 | Camera/gimbal origin, lens calibration and image formation are nominal | Sensors S07 |
| 1–2 | Phone-selected airframe/gimbal/capture state and capabilities can diverge from the host | Sensors S08 |
| 2 | Generic remote latency/loss policy does not reproduce an aircraft-specific link/autopilot failsafe | Sensors S09; E6 |
| 2 | No load/RPM-driven four-prop acoustic installation; HUD spin is deliberately slowed | Sensors S10 |
| 2 | Prior non-drone physics-rate preference can be lost after reload while flying a drone | Controls C12; integration-preference issue, not a force error |
| 1–2 | No traceable validation envelope or physical acceptance suite for these aircraft | V1–V3 |

## Environment, terrain and transition details

### E1 — Weather wind is not the wind used for drone force generation

**Confirmed integration defect.** Weather writes native atmosphere wind in
[`applyWeather`](../../fly/src/flight/createFlightSimApp.ts#L2694). The quad
controller receives a separate seeded OU process initialized at zero and uses
it for drag (`droneController.ts:200`, `droneWind.ts:34`). Native aero coefficients
are zero. Native wind may affect instruments while ordinary drone force uses
another wind. Vertical gusts, spatial gradients and a common weather/turbulence
observation are not represented by this two-component private process.

Remedy: one authoritative environmental velocity field, with optional gust
model added once and sampled at physical time/position. Acceptance: prescribed
N/E/down wind changes air-relative velocity, force and instruments coherently;
ground-relative and air-relative quantities remain distinguishable. Do not
silently apply the same drag or gust twice.

### E2 — Density and airframe resistance are fixed, isotropic and uncalibrated

**Confirmed model simplification.** `Quadcopter/public/0sfs-mod.json` supplies
`airDensity: 1.225`; `droneController.ts:101` uses that rather than the active
atmosphere. Mini/Air profiles supply one `cdA` each for every attitude and flow
direction. There are no body aerodynamic moments or rotor/body aerodynamic
coupling in the zero-aero XML.

Remedy: use the authoritative atmospheric state and a separately applied
body-axis aerodynamic model with documented coefficients. Acceptance: rotate
the body in a known airflow, vary density/altitude and compare expected force
direction and magnitude; fit drag/thrust parameters jointly rather than tuning
controller gains to conceal bad aerodynamics. Exact coefficient accuracy remains
unknown until data is available.

### E3 — Ground effect and demanding rotor-flow regimes are absent

**Confirmed model omission.** The controller's AGL branches only limit descent
commands/replace aggregate force; no rotor aerodynamic inflow, wake, airspeed
operating map, ground effect or descent recirculation model exists. That absence
matters for low hover, fast flight and descent. It does not justify inventing a
manufacturer-specific threshold or claiming vortex-ring behavior is validated.

Remedy: choose a rotor model appropriate to the intended envelope and validate
its low-altitude, forward-flight and descent behavior. Add higher-order wake
interactions only when required. Acceptance: reproducible thrust/power curves
versus height, inflow and rotor speed, with the tested domain recorded.

### E4 — Drone geometry has no swept hull/rotor collision coverage

**Confirmed integration omission.**
[`registerFdmProfile`](../../fly/src/flight/jsbsim/fdmProfiles.ts#L138) defaults
unknown airframe body probes to `[]`; neither drone manifest supplies them.
The host passes that list into the visible-mesh collision system at
[`createFlightSimApp.ts:1326`](../../fly/src/flight/createFlightSimApp.ts#L1326).
The HD mesh is presentation geometry and is deliberately not a pick/collision
target. Its appearance does not provide physical collision coverage. Native
belly contact still exists, so this is not a claim that all ground collision
is missing.

Remedy: declare drone-sized body/arm/rotor collision geometry through the
existing host/profile seams. Acceptance: nose, side, top and propeller tips
approaching known planes produce the chosen collision/failure behavior without
tunneling. Respect the current prohibition on editing the shared ground-contact
package; any change requiring that package needs a separate scope decision.

### E5 — Ground support and landing are a coarse model

**Confirmed model simplification and controller shortcut.** Each drone XML has
one central BELLY structure contact, with generic spring/damping/friction. It
does not reproduce the distributed feet/arms, tipping, compliant structure or
prop strikes. Terrain support feeds JSBSim a local horizontal support plane;
surface refinement/placement can move that plane and occasionally the aircraft
(`ground-contact/src/terrainContact.ts:807`). Those corrections are intentional
streamed-world continuity policies, not physical forces.

Remedy: first separate contact forces from rotor commands, then provide
airframe-specific support geometry and plausible coefficients through allowed
data/host seams. Acceptance: level/sloped landings, one-foot contact, lateral
slides, bounce and overturn are stable and consistent with the chosen model;
terrain refinement does not create an unexplained flight impulse. Do not
misclassify a deliberate placement repair as a force-integrator defect.

### E6 — No modeled operational state machine or energy-dependent failsafes

**Confirmed omission at the drone-feature boundary.** The quad has no actual
motor arming/disarming/spin-up state, battery depletion, low-energy landing,
aircraft-specific link-loss response or obstacle-sensor behavior in this stack.
Generic simulator ownership/damage/pause handling is not an implementation of
those onboard behaviors. An always-running hover controller cannot reproduce
a stopped, failed or partially degraded vehicle.

Remedy: define an explicit vehicle state machine and connect it to the motor,
energy, observation and command-age models. Implement only supported per-airframe
capabilities with primary data; do not assume Mini 2 and Air 2S have identical
obstacle or failsafe features. Acceptance: arm/disarm, link loss, motor failure
and depleted-energy scenarios have deterministic, documented outcomes.

### E7 — The simulation can fall behind wall time

**Intentional numerical budget with cross-clock consequences.**
[`physicsRate.ts`](../../fly/src/flight/physics/physicsRate.ts#L30) caps catch-up
to six steps/50 ms at 120 Hz; excess backlog is discarded by
[`fixedStepLoop.ts`](../../fly/src/flight/physics/fixedStepLoop.ts#L110).
This protects the laptop from a catch-up spiral. It is not a reason to remove
the fixed timestep. It becomes physically incoherent when other dynamics,
sensor intervals or feedback advance by the wall/render time discarded here.

Remedy: retain bounded computation, expose actual simulation speed/time loss,
and use accepted simulation timestamps for all physical subsystems. Acceptance:
the same recorded inputs over the same accepted simulation duration produce
equivalent states at several render rates and across a forced stall; physical
sensors, motors and capture clocks do not secretly run ahead.

### E8 — Saved/switch state has a defined meaning, not a conservation claim

**Intentional scenario setup; reset-boundary defects are in Controls C6.**
Swapping airframes preserves velocities/rates while changing mass/inertia. That
editor operation cannot conserve all momentum/energy of the old aircraft; it
sets a new initial condition. The new vehicle should nevertheless obey its
own force/energy constraints afterward. The 100-knot experiment should remain
available as a repeatable scenario. Momentum-follow camera framing/smoothing
can change perceived motion independently of physics; it should be compared
with recorded native state or a fixed-world view when assessing behavior.

Acceptance: document and test each preserved/reset field, including pause,
weather, camera and controller anchors. Compare velocity and acceleration traces
before/after resume; never repair an implausible transient by silently deleting
the starting velocity.

## Validation gaps

### V1 — Internal consistency has been mistaken for physical validation

Existing controller tests cover balanced hover and a small wind/tilt case.
Motor tests validate the estimator's own mixer and RPM² assumptions. Browser
checks validate loading, UI, independent readings, pause and presentation.
These tests are useful but do not establish thrust limits, real stopping
distance, energy consumption or DJI-equivalent control response.

Add invariants before performance targets: rotor force direction follows the
actual body, individual thrust is unilateral/bounded, summed motor wrench
equals the wrench delivered to JSBSim, drag opposes relative airflow and is
not erased by control limits, motor/vehicle energy has a declared source, and
rejected steps roll back every affected state.

### V2 — No calibrated aircraft dataset or declared accuracy envelope

There is no checked-in, traceable dataset fitting per-motor speed/thrust/torque,
motor response, body drag, CG/inertia and battery behavior for each variant.
Nominal speed/weight numbers alone cannot determine these. A proper motor state
would provide exact **simulated** RPM, but real-world accuracy would still
depend on these data and the applicable operating envelope.

Create a parameter/provenance ledger: measured, manufacturer nominal, inferred,
or provisional; uncertainty; source; calibration scenario; and valid domain.
Keep any non-redistributable local reference files out of public artifacts.
Compare against independent flight/bench data rather than only fitting the
same samples used for acceptance. Do not invent proprietary DJI values.

### V3 — Repeatable stress/transition scenarios are missing

Add recorded scenarios for: hover; step climb/descent and release; bank/yaw
recovery; inverted/90° initial attitudes; 100-knot velocity-preserving switch;
wind/density changes; constrained motors; low-power/dead-motor flight; sloped
landing/prop strike; teleport/reload; link loss; changing render cadence; and
moving-camera acquisition. Log actual body pose, velocity, applied forces,
per-motor state, constraints and accepted simulation time.

The realistic result at the edge of an envelope need not be successful hover
or recovery. It must be an explainable outcome of a physically constrained
model. Detailed material fracture or photometric sensor effects are optional
extensions until a use case requires them.

## Repair order and module boundaries

1. **Correct the force boundary.** Separate body aerodynamics from rotor thrust;
   transform thrust through actual attitude; enforce nonnegative/bounded rotor
   thrust and feasible total moments; remove force overrides that bypass those
   constraints. Keep JSBSim as the one rigid-body integrator and preserve the
   corrected position-hold signs.
2. **Make motors causal.** Controller requests → constrained allocator → ESC/motor
   states → four rotor forces/torques → JSBSim. Feed the exact same shaft states
   to props, RPM gauges and sound. Remove reverse-estimated RPM from the physical
   path. This can remain modular around JSBSim; it does not require rewriting
   the globe renderer or adding another motion integrator.
3. **Make command/state/time coherent.** Address Controls C1–C9 and E1–E2, with
   reset/accepted-step boundaries, common atmosphere and clear supported assists.
   Fix optical flow and shutter acquisition timing before treating them as
   validation instruments. Add drone contact geometry through permitted seams.
4. **Calibrate and bound claims.** Identify airframe/motor/rotor/electrical
   parameters and validate normal flight, stress cases and energy budgets.
   Add gimbal mechanical constraints and calibrated camera geometry.
5. **Add mission-specific fidelity.** Sensor/estimator errors, radio/failsafes,
   acoustic feedback, wake effects and image exposure effects become separate
   components with explicit requirements and tests.

The current [contract](SPEC-360-DRONE.md) explicitly describes ideal externally
applied forces and zero native propulsion. The force/propulsion repair must
update that contract alongside implementation, rather than present the existing
shortcut as hardware-accurate. Do not move drone implementation onto startup,
restore old hold signs, introduce a WebGL capture path, or publish these mods.

JSBSim itself has propeller/brushless-motor implementations: its
[propeller source](https://raw.githubusercontent.com/JSBSim-Team/jsbsim/master/src/models/propulsion/FGPropeller.cpp)
uses aerodynamic load and shaft dynamics, while its
[brushless motor source](https://raw.githubusercontent.com/JSBSim-Team/jsbsim/master/src/models/propulsion/FGBrushLessDCMotor.cpp)
uses electrical motor parameters. Those upstream capabilities do not mean our
current zero-propulsion XML uses them, or that the bundled build/model geometry
is already suitable and calibrated for these quadcopters. The
[PX4 controller architecture](https://docs.px4.io/main/en/flight_stack/controller_diagrams)
is a useful public reference for separating position/velocity/attitude/rate
control and constrained allocation; it is not evidence of DJI firmware behavior.

## Verification against current code (28 October 2026, later pass)

After the propeller correction, the central findings were rechecked against the
working tree. None was found fixed or withdrawn; nothing in this pass changed
flight forces, controller gains or the position-hold signs.

- **P1–P5** (`fly/src/flight/drone/droneController.ts`, `planDroneForce`):
  force is computed and written in local north/east/down regardless of actual
  attitude. Drag is added before the tilt clip and scaled with it. Nothing bounds
  `fD` below zero or above a thrust limit. Moments are bounded only by the
  rate-command clamps. The AGL branches overwrite the force vector.
- **100-knot example**: reproduced from `Quadcopter` `kv` 2.5 and Mini 2
  `cdA` 0.02 m², 0.249 kg, ρ 1.225, 40° tilt: 128.61 + 130.20 m/s², clipped to
  8.23 m/s². **Air 2S descent**: `kv`·6 m/s = 15 m/s² > g, so the
  requested rotor force points downward.
- **E1**: `droneFeature.ts` still owns a private OU wind initialized to zero
  and passed to the controller; native weather wind is not read.
- **C1**: `currentDroneCommand` (`droneGate.ts:216`) still returns the raw phone
  payload wholesale, bypassing the blended and pilot-out-neutralized record
  (`createFlightSimApp.ts:3638`–`3643`, `3718`–`3726`). Citations in the
  controls appendix were updated; they had drifted by 1–5 lines.
- **C9**: the controller receives `latestTerrainMeters`
  (`createFlightSimApp.ts:3727`), which is assigned later in the same frame
  (`:3801`), so physics uses the previous frame's AGL.
- **S01**: `drone-camera/src/flowWgsl.ts:69`–`87` still computes `ix`/`iy` once
  at the centre pixel and reuses them for all 49 window samples (rank-one tensor).

Other line citations in the appendices were not individually re-resolved and
may have drifted by a few lines as `createFlightSimApp.ts` changes; the
described code paths were not seen to change.

## Separately requested 3D-propeller correction

**Superseded and corrected on 28 October 2026.** An earlier pass misread the
request. It made individual sharp 3D blades the high-RPM default, with a flat
disc as the only alternative. The request referred to the C172's behavior:
above the display alias limit, the blades are replaced by a
**semi-transparent 3D swept propeller volume**, the visible prop cover while
it rotates. The C172's authored `Propeller_Disc` is exactly that solid, generated
from its blade sections.

Current behavior, shared by C172 and drones in `@0sfs/rotating-prop`:

- Below π / blades / frameSeconds (0.8 hysteresis): the authored 3D blades turn.
- Above it, by default: the blades hide and a generated (or, for the C172,
  authored) semi-transparent 3D swept volume is shown. The generator samples
  every blade triangle under the rotor pivot, records the forward/aft reach at
  each radius, and revolves that profile. Pitch and coning therefore give it
  real thickness edge-on (Mini 2 HD ≈ 8–9 mm, Air 2S HD ≈ 13 mm).
- **Aircraft → Flat propeller discs**, default **Off**, is a live,
  display-only opt-in that substitutes a flat disc. Turning it off restores the
  sweep. Disposing a binding re-enables the authored blades.

This is presentation only. It does not create motor physics; drone RPM is still
the inverse estimate described in M1. A high-RPM sweep is not evidence that a
shaft state exists. Verification: fly's ignored `build/prop-sweep/` (browser
report, edge-on screenshots of sweep vs optional flat disc).

### Rotor radius data found while verifying

The HD meshes measure ≈ 59.5 mm (Mini 2) and ≈ 91.7 mm (Air 2S) blade radius,
consistent with the published 4726F (4.7 in) and 7238F (7.2 in) propeller
diameters. Both manifests declare smaller `radiusMeters` (0.045 and 0.053), and
the procedural stand-ins are 45 mm. Display sizes now come from the measured
blades, so the declared values no longer shrink the visuals. Nothing in the
flight model uses rotor radius today. Any future rotor/motor model must not
take these declared radii as calibrated without correcting them (see M2, V2).
