# Drone flight-model overhaul roadmap — 28 October 2026

Status, 28 October 2026: partly built; nothing is committed or published. The
rotor plant and flight controller exist in [`multirotor/`](../../multirotor/README.md),
and the drones are moving onto them. [What was built](#what-was-built-28-october-2026)
says what changed from this plan, and each stage below has its status. The
rest of the page is the original plan.

The plan builds on the [realism audit](360-drone-realism-audit-2026-10-28.md) and its
appendices ([propulsion](360-drone-realism-propulsion-audit-2026-10-28.md),
[controls](360-drone-realism-controls-audit-2026-10-28.md),
[sensors](360-drone-realism-sensors-audit-2026-10-28.md)). IDs such as P1, M2
and E1 refer to those documents. The propulsion appendix has its own C1–C3
(controller approximations), written here as "propulsion C2"; plain C1–C12 are
the controls appendix. Each stage below, when built, replaces part of
[SPEC-360-DRONE](SPEC-360-DRONE.md) §4.1–§4.5 (see
[Contract changes](#contract-changes)).

## Summary

1. **Goal.** Replace the placeholder flight model with the real causal chain:
   flight controller → four ESCs → four motors → four spinning propellers →
   forces and torques at each hub, along each propeller's actual shaft →
   JSBSim. Body aerodynamics, mass properties, battery, wind and ground contact
   are separate models, each built from geometry, published specifications or
   measurement. **As built, the chain stops at the spinning propeller:** each
   rotor's speed follows its command with a first-order lag, in place of the
   ESC, motor and battery (see [What was built](#what-was-built-28-october-2026)).
2. **It works in our JSBSim build.** A probe flew a Mini 2-sized quad on four
   JSBSim brushless motors and propellers, controlled only through the four
   throttles. It held hover to 0.4 mm at about 8,970 RPM per motor; the motors
   respond with a 25 ms time constant. One JSBSim step took 3.8 µs, 0.05% of one
   CPU core at 120 Hz.
3. **Data is the constraint, not compute.** The strategy is *compile offline,
   look up at runtime*. CFD of the body, blade-element analysis of the
   propellers and mass properties from the 3D model run in offline tools and
   produce tables. At runtime the simulator does table lookups and integrates a
   handful of states. The rotor plant that was built needs no tables: it solves
   blade element and momentum theory analytically every step, in microseconds.
   CFD of the body is still an offline job, and is deferred.
4. **Stock JSBSim leaves gaps.** In a 10 m/s headwind the probe's propellers
   produced zero in-plane force. Nothing models rotor drag (H-force),
   translational lift, ground effect, vortex ring state or rotor-wake
   interaction, and the old drone XML had no body aerodynamics at all. The
   rotor plant now covers all of these except rotor-wake interaction, and the
   XML has interim per-axis body drag. Stage 4 would add the rest.
5. **Several current values are measurably wrong.** From DJI's published
   speed and tilt limits, the Mini 2's total drag area at top speed is at most
   0.0131 m² (manifest 0.02) and the Air 2S's at most 0.0185 m² (manifest
   0.045). With today's values neither can reach its published top speed. The
   inertias are about 3× too high, the Mini 2 should hover near 9,000 RPM rather
   than 6,000, "normal" mode uses Sport-mode limits, and the rotor radii are
   25–42% too small. Details are in [Data errors](#data-errors-found-while-planning).
   All of these were corrected in the manifests and XML on 28 October.
6. **Rust/WASM by default, and off the main thread unless it must be on it.**
   The per-step models become Rust/WASM kernels beside TypeScript references,
   as ground contact's already are. The step itself stays on the main thread:
   moving it to a worker failed its gate on 21 October 2026, worst of all at
   low altitude. Motor sound runs on the audio thread. Recording, flight-lab
   batches and the offline compilers run in workers or across all cores.
   Details are in [Rust/WASM, threads and the main thread](#rustwasm-threads-and-the-main-thread).

## What was built, 28 October 2026

### Down to the spinning propeller

The simulation goes down to the spinning propeller and stops there. Each
rotor's speed follows its command with one time constant: an ideal motor and
ESC. There is no motor, ESC or battery model. This was the user's decision on
28 October.

The reason is data, not compute:

- **No data.** DJI publishes no motor or ESC data, so a motor model's Kv,
  resistance and current limits would be guesses tuned to land hover in range.
- **Cost is not the reason.** The probe ran four JSBSim brushless motors with
  their propellers in 3.8 µs per step, which would have been affordable.

Stage 5 (energy) is therefore dropped, and flight time, voltage sag and
low-battery behavior are not simulated.

### What exists

- **[`multirotor/`](../../multirotor/README.md) (`@0sfs/multirotor`)**, laid out
  like ground-contact: TypeScript kernels in `src/`, their Rust twins in
  `wasm/`.
  - **Rotor plant.** For each rotor and step:
    - the speed lag;
    - blade element and momentum theory with Glauert's inflow in climb and
      forward flight;
    - Leishman's empirical axial curve through the vortex ring state, then the
      windmill brake state, blended so the inflow has no jumps;
    - a correlated vortex-ring buffet;
    - Sanchez-Cuevas et al.'s multirotor ground effect: the rotor's own image,
      its neighbours' and the body's;
    - the H-force and the hub rolling moment;
    - reaction torque and the gyroscopic moment.

    It sums the four rotors into one force and moment about the CG, in body
    axes.
  - **Flight controller.** Four loops:
    - velocity, with jerk-limited setpoint shaping and an integral;
    - tilt-prioritised attitude, then heading;
    - body rates, with an integral and the gyroscopic term;
    - minimum-norm allocation, in which thrust and tilt come first and yaw
      gives way.

    It scales its static-thrust model by the thrust efficiency it measured on
    the previous step. That is incremental dynamic inversion, so climbs and
    fast flight need no gain schedule.
  - **Rust/WASM twin.** It matches the TypeScript bit for bit and allocates
    nothing while stepping. If it fails, the runtime falls back to the
    TypeScript for good and says why.
  - **Tests.** 26 tests, plus the WASM parity check.
- **JSBSim models.** The Mini 2 and Air 2S XMLs have:
  - the rotors' wrench as BODY external reactions at the CG, through
    `multirotor/force-{x,y,z}-lbs` and `multirotor/moment-{l,m,n}-ftlbf`;
  - body drag on JSBSim's body X, Y and Z aero axes, with a drag area per
    axis;
  - the component-estimate mass, CG and inertia;
  - four feet.

  There is no `<propulsion>`. The plant takes the place of FGPropeller and the
  brushless motor.
- **Manifests.** Each aircraft's `quad` section holds:
  - the Normal and Sport mode limits from DJI's specifications;
  - the propeller and the rotor layout (hubs and spins);
  - the CG and the controller gains.

  The shared Quadcopter manifest keeps only the stick mapping, position hold and
  property names.
- **Fly, 28 October.** Handoffs B and C built this, and handoff E integrated
  it:
  - the JSBSim bridge and command layer, including arming and a landing
    slowdown;
  - simulated per-rotor RPM in the engine monitor and props;
  - the procedural model at the real hubs;
  - `planDroneForce`, the private wind and the RPM estimate, deleted.

### Numbers

The propellers' radius and pitch come from the HD meshes and DJI's propeller
names (4726F, 7238F). Chord, section data, inertia, time constant and the speed
limits are estimates.

| | Mini 2 | Air 2S |
| --- | --- | --- |
| Hover, sea level | 8,994 RPM | 6,243 RPM |
| Static C_T (propeller convention) | 0.109 | 0.097 |
| Figure of merit | 0.70 | 0.68 |
| Hover induced velocity | 4.72 m/s | 4.75 m/s |
| Thrust at maximum RPM over weight | 2.19 | 2.17 |

The closed-loop envelope was flown against `multirotor/tests/rigidBody.ts`, a
flat-earth rigid body with the same drag areas:

| Aircraft, mode | Speed (m/s) | Tilt, max / limit | Climb (m/s) | Descent (m/s) | Yaw (°/s) |
| --- | --- | --- | --- | --- | --- |
| Mini 2 Normal | 10.01 | 24.7° / 25° | 3.01 | 3.01 | 60 |
| Mini 2 Sport | 16.00 | 37.9° / 40° | 4.99 | 3.51 | 130 |
| Air 2S Normal | 15.02 | 24.9° / 25° | 5.99 | 6.02 | 90 |
| Air 2S Sport | 18.88 | 34.8° / 35° | 5.99 | 6.02 | 250 |

The Air 2S descends at its published 6 m/s through the vortex ring state, which
is 1.26 × its hover induced velocity, with under 5° of tilt in 20 s of buffet.

`fly/src/flight/drone/quadJsbsimLab.test.ts` flies the mod XMLs in the
installed JSBSim WASM build at 120 Hz. Its run on 28 October, before
integration, measured the following:

| Real JSBSim | Mini 2 | Air 2S |
| --- | --- | --- |
| Hover at 20 m | 9,003 RPM, height within 1.2 cm | 6,249 RPM, height within 1.2 cm |
| Normal: speed, time to 97%, peak tilt / limit | 10.02 m/s, 3.7 s, 24.7° / 25° | 15.02 m/s, 6.0 s, 25.9° / 25° |
| Sport: speed, time to 97%, peak tilt / limit | 16.00 m/s, 4.7 s, 38.7° / 40° | 18.87 m/s, 6.4 s, 35.3° / 35° |
| Climb / descent, Normal | 3.00 / 3.01 m/s | 6.00 / 6.02 m/s |
| Climb / descent, Sport | 5.00 / 3.51 m/s | 6.00 / 6.02 m/s |
| 10 m/s wind: lean, ground speed | 10.6°, 0.009 m/s | 7.9°, 0.004 m/s |
| At rest: CG height, pitch | 10.0 mm, 0.02° | 10.0 mm, 0.04° |
| Whole step, mean | 14.2 µs | 12.2 µs |

The lab sends velocity commands straight to `@0sfs/multirotor`, bypassing the
stick and hold layer.

- **Tilt.** While accelerating, the Air 2S overshoots its tilt limit by under
  1°; the lab allows 1.5°.
- **Hover RPM.** It sits 0.1% above the sea-level figure because the air at
  20 m is thinner.
- **Landing.** That case failed in this run. Integration added a flare, and
  stage 6 has the passing numbers.

One runtime step costs the following for the Mini 2. The step is the
controller plus the plant. The machine was Node 26 on an Apple M5, on
battery:

| Case | TypeScript | WASM |
| --- | --- | --- |
| Hover | 4.7 µs | 9.4 µs |
| Sport, 16 m/s | 3.7 µs | 12.1 µs |
| Vortex ring | 10.9 µs | 16.8 µs |

Both are under 0.2% of the 8.3 ms a 120 Hz step allows. V8's optimised
TypeScript is faster than this WASM. Building without `simd128`, turning off
Liftoff, or calling the kernel directly made no difference. The WASM build
stays for a cost no JIT can deoptimise, and for a later move off the main
thread.

### Out of scope or deferred

- **Stage 4:** CFD body tables and rotor-wake interference. Without them, rear
  rotors do not run faster than front rotors in forward flight, and hover
  download is not modelled.
- **Stage 5:** the battery, ESC and energy. Dropped.
- **Stage 7:** sensors and the EKF. The controller flies on true state, and
  its thrust-efficiency measurement is an ideal accelerometer.
- **Stage 8:** validation against flight logs.
- **Stage 9:** the step in a worker. Blocked on FOSS Earth's steps 6 and 9.
- **The offline blade-element oracle (implementation C).** The plant's
  analytic blade element and momentum theory stands in for it.
- **Cine mode.** Only Normal and Sport exist.

## What "perfect" means here

The unreachable ideal is blade-resolved, time-accurate CFD of rotors and body,
coupled to motor electromagnetics, structural vibration and DJI's own firmware.
That costs hours per simulated second, and the firmware is closed.

The practical target is this: **every quantity that a pilot, camera, gauge or
sensor can observe comes out of a causal physical model.** Each model's
parameters come from geometry, a published specification or a measurement, and
its error against reality is recorded. Nothing observable is computed backward
from something else or added for appearance.

| Observable | Physical cause | Model | Parameters from | Checked against |
| --- | --- | --- | --- | --- |
| Tilting to move, leaning into wind | Thrust acts only along the propeller shafts | Four rotors at the real hubs, integrated by JSBSim | HD model geometry | P1 invariants |
| Top speed at maximum tilt | Body drag and rotor drag versus tilted thrust | Body aero tables; rotor in-plane force | Aero compiler; blade-element tables | DJI speed and tilt limits |
| Climb and descent, wobble in fast descent | Thrust versus axial inflow; vortex ring state | Rotor maps versus inflow; VRS model | Blade-element tables; Johnson 2005 | DJI ascent/descent; VRS literature |
| Cushion near the ground | Ground effect | Multirotor ground-effect model | Geometry | Sanchez-Cuevas et al. 2017 |
| RPM, motor sound, rear motors faster in forward flight | Motor torque against prop load; front-rotor wake on rear rotors | Motor and propeller states; rotor-on CFD interference | Motor identification; CFD | NASA tunnel trends; flight logs |
| Flight time, voltage sag | Electrical power | Battery, ESC and motor | DJI battery specs; identification | DJI flight and hover times |
| Response to sticks, braking | DJI-like flight controller | Cascade controller with DJI mode limits | DJI specs; logs | DJI speed, tilt and angular-rate limits per mode |
| Hover precision, drift | Sensors, estimator, gusts | IMU, GPS, vision, estimator; turbulence | DJI hover-accuracy spec | DJI ±0.1/0.3 m (vision) and ±0.5/1.5 m (GPS) |
| Landing, tip-over | Contact at the feet | JSBSim contacts plus collision probes | Geometry | Landing and slope tests |

Left out until a use case needs them: blade flexing, acoustics beyond RPM- and
load-driven sound, motor magnetics beyond the DC-motor model, propeller damage,
icing, and building or terrain wakes.

## Where the model stood before the rebuild

This section is history: it describes the drone before 28 October. The rotor
plant and flight controller in [What was built](#what-was-built-28-october-2026)
replace all of it except the inertia, which the XML now takes from the
component estimate.

The drone was JSBSim used as a bare rigid body. A controller in
[`droneController.ts`](../../fly/src/flight/drone/droneController.ts#L60)
computes one force in world (north/east/down) axes plus three body moments and
pushes the body with them. Then it separately rotates the body to look right.
That produces these failures:

- **P1:** thrust does not follow the actual tilt. A drone rolled 90° gets the
  same upward force as a level one.
- **P2:** the tilt limit clips aerodynamic drag together with the thrust request.
- **P3:** thrust can point downward or grow without limit.
- **P4:** moments are not tied to any motor.
- **P5:** forces are overridden near the ground.
- **M1–M3:** there are no motors, propeller maps or battery.
  [`estimateDroneMotorRpm`](../../fly/src/flight/drone/droneMotorTelemetry.ts#L26)
  works backward from the force request, assuming 6,000 RPM hover.
- **E1, E2, D2:** the drag is one isotropic number with fixed density, and the
  wind is a private random process that ignores the weather.
- **D1:** the inertia assumes all mass sits at the motors.

### Data errors found while planning

| Item | Manifest / SPEC | Evidence-based value | Basis |
| --- | --- | --- | --- |
| Mini 2 drag area (`cdA`) | 0.02 m² | ≤ 0.0131 m², body plus rotor drag together | 16 m/s at ≤ 40°: W·tan 40° / (½ρV²) |
| Air 2S drag area | 0.045 m² | ≤ 0.0185 m² | 19 m/s at ≤ 35° |
| Top speed at maximum tilt with today's `cdA` | — | Mini 2 12.9 m/s; Air 2S 12.2 m/s | Same formula |
| Mini 2 inertia Ixx / Iyy / Izz | 1.41e-3 / 1.41e-3 / 2.82e-3 kg·m² | ≈ 4.5e-4 / 4.3e-4 / 8.0e-4 kg·m² | Component estimate (motors, arms, body, battery) versus all mass at the motors |
| Iyy | Copied from Ixx ([`droneTypes.ts`](../../fly/src/flight/drone/droneTypes.ts#L78)) | Its own value | Body is longer than wide |
| Hover RPM (`rotorHoverRpm`) | 6,000 for both | Mini 2 ≈ 7,900–10,500; Air 2S ≈ 5,200–6,900 | T = C_T·ρ·n²·D⁴ with static C_T 0.08–0.14; probe 8,974 at C_T 0.11. Body download adds a few percent |
| Rotor radius (`radiusMeters`) | 0.045 / 0.053 m | 0.0597 / 0.0917 m | HD meshes; 4726F (4.7 in) and 7238F (7.2 in) props |
| Normal-mode tilt | 40° / 35° | 25° / 25° (Air 2S up to 35° in strong wind; DJI's pages disagree on detail) | DJI specs |
| Normal-mode yaw rate | 130 / 250 °/s | 60 / 90 °/s (Sport 130 / 250, Cine 30 / 60) | DJI specs |
| Mini 2 normal-mode climb / descent | 5 / 3.5 m/s | 3 / 3 m/s (5 / 3.5 is Sport) | DJI specs |
| Yaw torque per newton of thrust | arm/4 = 26.6 mm (Mini 2) | ≈ 6–9 mm (C_P·D / (2π·C_T)) | Propeller coefficients; yaw authority is about 3× weaker than the estimator assumes |

The drag bound assumes DJI's top speed is reached in still air at or below the
published tilt with negligible body lift. If DJI reaches it at less tilt, drag
is lower still.

## Evidence from the probe

[`probe.mjs`](../../fly/build/drone-flight-model/probe.mjs) is in fly's ignored
`build/`, with results in `probe-results.json` beside it. It ran in Node on the
same JSBSim WASM build the app uses. The airframe was Mini 2-sized:

- 0.249 kg.
- Inertia 4.0e-4 / 4.5e-4 / 7.5e-4 kg·m².
- Hub positions measured from the HD model (front hubs 22 mm higher than rear).
- Four `<brushless_dc_motor>`: 8.4 V, Kv 2,600, 0.25 Ω, 0.25 A no-load.
- Four `<propeller>`: 4.7 in, with C_T and C_P tables versus advance ratio J.
- Thrust pointed up with `<orient>` pitch 90°.
- Four feet.

**The coefficients are plausible, not identified.** The numbers show behavior
and cost, not Mini 2 truth.

| Test | Result |
| --- | --- |
| Load | Four engines, no warnings |
| Hover on throttles only (altitude PI, attitude PD, 4×4 allocation, per-motor RPM loop) | Altitude error 0.4 mm. Attitude level to 1e-6°. Throttle 0.457; 8,974 RPM, 0.610 N and 1.54 A per motor; 23.7 W total |
| Motor step, +0.05 throttle | 63% response in 25 ms |
| Reaction torque | `sense` +1 yaws the body nose-right. This matches [`droneMotorDirection`](../../fly/src/flight/drone/droneMotorTelemetry.ts#L13), where front-left and rear-right are +1 |
| 10 m/s headwind, level hover | In-plane rotor force 0 N; the drone stays level and drifts 0.024 m/s. A real Mini 2 must lean roughly 10–18° into that wind |
| Cost per step | `run()` 3.81 µs; plus batched property I/O 3.91 µs; with unbatched string-property I/O 11.12 µs |
| CPU share | 0.047% of a core at 120 Hz; 0.19% at 480 Hz |
| FGRotor variant (JSBSim's helicopter rotor) | Diverged: RPM 0.4–0.9 million and negative currents. The cause is direct-drive props with almost no engine-side inertia behind FGTransmission's freewheel. Not pursued |

## Target architecture

```text
sticks / phone / autopilot
  → command mapping (kept: deadzone, holds, corrected hold signs, yaw modes)
  → flight controller (Rust/WASM kernel with a TypeScript reference, lazy drone chunk, 120 Hz)
       position → velocity → acceleration and thrust vector → attitude (quaternion)
       → body rates → torque → allocation within motor limits → four RPM commands
  → four ESCs (speed loop, current limit) ← battery (voltage, state of charge)
  → JSBSim, one step:
       4 × brushless motor + propeller: thrust, torque and gyroscopic moment on each real shaft
       rotor in-plane force, translational lift, ground effect, vortex ring state  (stage 3)
       body aerodynamic tables versus angle of attack and sideslip               (stage 4)
       compiled mass, CG and inertia; four feet; collision probes
       atmosphere: density, host weather wind, turbulence
  → state → sensors → estimator (stage 7) → flight controller
  → the same RPM states drive rotating-prop, the engine widget and sound (audio thread)
```

As built, the chain is shorter:

```text
sticks / phone
  → command mapping (kept: deadzone, holds, corrected hold signs; arming; landing slowdown)
  → @0sfs/multirotor flight controller, 120 Hz, TypeScript then Rust/WASM once loaded
       velocity → tilt-prioritised attitude, then heading → body rates
       → minimum-norm allocation, yaw gives way first → four rotor-speed commands
  → @0sfs/multirotor rotor plant, same step, from the pre-step JSBSim state:
       4 × first-order rotor-speed lag (in place of ESC, motor and battery)
       4 × blade element and momentum: thrust, H-force, torque, hub moment,
           vortex ring and windmill states, buffet, ground effect, gyroscopic moment
       → one body-axis force and moment about the CG
  → JSBSim, one step:
       BODY external reactions at the CG: multirotor/force-{x,y,z}-lbs, multirotor/moment-{l,m,n}-ftlbf
       body drag on the body X, Y and Z axes, one drag area each
       component-estimate mass, CG and inertia; four STRUCTURE feet
       atmosphere: density, host weather wind
  → true state → flight controller
  → the same rotor speeds drive rotating-prop and the engine monitor
```

Rules:

- JSBSim stays the only integrator of rigid-body motion, rotor speeds and
  contacts.
- Production drones get no force or moment that does not come from a
  propulsion, aerodynamic, contact or gravity model. JSBSim computes it, or a
  plant kernel computes it at the step boundary from the same pre-step state
  that JSBSim steps from (rotor option R, ground contact), and JSBSim applies it
  where it acts. The external-force
  properties `fcs/drone-f{n,e,d}-lbs` and `fcs/drone-m{x,y,z}-ftlbf` leave
  production. Tests may keep a body-frame disturbance injector. *As built:* the
  rotor plant's wrench goes to JSBSim at the CG, so the hub offsets enter as
  the moment rather than as separate application points.
- The controller sees only what real flight software could: commands, its
  sensors (ideal at first), and its own fixed nominal model. It never reads
  plant coefficients to cancel them, so model mismatch shows up the way it does
  on hardware. Gains are never tuned to hide a wrong force model. *As built:*
  the nominal model is static thrust ∝ ω² from the same propeller
  specification. The thrust efficiency that scales it is the plant's last total
  thrust over that static thrust, filtered with a 0.1 s time constant and
  clamped to 0.3–3. That is an ideal accelerometer, not a read of the
  plant's coefficients.
- One source per quantity. Hub positions, spin directions and radii come from
  one compiled geometry and drive physics, visuals and sound alike. Mass
  properties are read live from JSBSim (`inertia/*`).
- Drone code stays in the lazy chunk; the position-hold signs stay as corrected.
- Properties are read and written in batches. Unbatched string access cost
  about 3× the step itself.

Where things live:

- **Runtime models** (controller, allocation, ESC, battery, rotor option R,
  estimator) go in a new sibling package, provisionally `multirotor`, laid out
  like `ground-contact`: TypeScript references in `src/`, their Rust twins in
  `wasm/`. They are pure, without Babylon, so the flight lab can run them
  headless. Only fly's lazy drone chunk imports the package. Like
  `rotating-prop`, it stays local until publishing is requested. The
  integration (`droneFeature.ts`, the gate, camera, gimbal and telemetry) stays
  in `fly/src/flight/drone/`. *Built:* [`multirotor/`](../../multirotor/README.md)
  holds the controller, allocation and rotor plant. There is no ESC, battery
  or estimator.
- **Generated aircraft data** (propulsion, aerodynamics and mass XML, plus
  provenance JSON) goes in each mod's `jsbsim-data/`, so mods stay declarative.
  JSBSim looks for engine files in the aircraft folder's `Engines/` or
  `engine/` directory, and its model loader accepts `file=` includes, so
  compiled sections can be separate files. The mod's file list must name them.
- **Offline compilers** go in a new sibling repository, provisionally
  `airframe-compiler`, covering mass properties, propeller tables and body
  aerodynamics. They are written in Rust and use every core. Like
  `rotating-prop`, it stays local until publishing is requested. *Not built:*
  the rotor plant needs no propeller tables, and the mass properties and body
  drag areas are hand estimates in the XML.

## Rust/WASM, threads and the main thread

New drone code is Rust compiled to WebAssembly by default, beside a TypeScript
reference, and nothing runs on the page's main thread unless it has to. This
follows conventions that already exist: FOSS Earth's
[WebAssembly cores](https://github.com/foss-earth/docs/blob/main/docs/webassembly-cores.md)
and 0SFS's [ground-contact kernels](../../ground-contact/wasm/README.md).

### What has to stay on the main thread

The fixed step has to stay there. That covers JSBSim's `run()` and its batched
property I/O, plus the controller, ESCs, battery, rotor extras and ground
contact, at 120 Hz inside the frame. Moving the step to a worker was measured
on 21 October 2026 and
[failed its gate](../../fly/docs/performance/2026-10-21/flight-dynamics-worker-results.md):

- The page cannot block on a worker, because
  [`Atomics.wait`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait)
  is not allowed on the main thread. A worker's step is therefore shown one
  frame later.
- Each step's exact terrain and collision queries became a separate
  main-thread task, which cost more than the in-frame work it replaced:
  0.30 ms against 0.12 ms per frame at cruise, and 0.88 ms against 0.49 ms at
  60 m.
- At 60 m the worker path was 6.4% slower at p95, and total CPU rose.

Drones spend their time low, which is that worst case. So the drone's step
stays on the main thread. It is reconsidered only when both of these FOSS Earth
steps exist:

- [Step 6](https://github.com/foss-earth/docs/blob/main/docs/proposals/webassembly.md#6-ground-height-grids-and-bvhs):
  ground grids and BVHs in shared memory, which a physics worker reads in
  place.
- [Step 9](https://github.com/foss-earth/docs/blob/main/docs/proposals/webassembly.md#9-jsbsim-in-a-worker):
  JSBSim in a worker on its own clock, publishing snapshots in shared memory.

[Stage 9](#stage-9--the-step-in-a-worker-conditional) then reruns the gate for
the drones. What makes the main-thread step acceptable until then is its size:
under 60 µs per step, under 0.7% of a core at 120 Hz (see
[Compute budget](#compute-budget)). The controller and rotor plant as built
measure 4–17 µs per step.

### Placement

| Work | Thread | Build | Why there |
| --- | --- | --- | --- |
| JSBSim `run()` and batched property I/O | Main, in the step | The existing JSBSim WASM | The gate above |
| Controller, allocation, ESC, battery, rotor option R, estimator | Main, in the step | Single-threaded Rust/WASM kernels with TypeScript references | Microseconds per step. A thread hand-off every step costs more than the work |
| Ground contact, feet and probes | Main, in the step | ground-contact's existing kernels | Unchanged; no edits to `ground-contact` |
| Props, gauges, HUD | Main, per frame | TypeScript and Babylon | They draw |
| Motor sound | Audio thread | A multirotor voice in sound's existing WASM DSP core, in its AudioWorklet | Already off the main thread. Rewriting that C++ core in Rust is outside this roadmap; a second core would duplicate its limiter and transport |
| Flight recorder at step rate | Sampling in the step; buffering, CSV and storage in a worker | TypeScript | Today's recorder builds its CSV on the main thread |
| Flight lab, identification and calibration batches | Worker pool: Node `worker_threads` in tests, Web Workers in the browser | JSBSim WASM plus the same kernel binaries | Independent runs; each worker owns its JSBSim and kernels |
| Offline compilers | Command line, all cores | Rust built for the host CPU, with `std::thread`; [wgpu](https://github.com/gfx-rs/wgpu) for backend L | Hours of batch work; never part of the app or tests |

Details:

- **Motor sound.** The step's RPM and current reach the worklet through sound's
  [`audioTransport`](../../sound/src/audioTransport.ts). It uses a
  MessagePort, or a shared ring on an isolated page.
- **Flight recorder.** Today's [recorder](../../logging/src/flightRecorder.ts)
  keeps a `Float64Array` ring and builds its CSV on the main thread. Drones
  record per step:
  - The step writes rows into a shared ring, or into transferred chunks when
    the page is not isolated.
  - A worker formats them and writes them to the
    [origin private file system](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system),
    whose synchronous file handles exist only in workers.

### Shared-memory threads

Threads pay off for large independent work, not for per-step kernels:

- **Per-step kernels:** single-threaded. The whole step takes microseconds, and
  waking a thread would cost more.
- **Flight lab and calibration:** a pool of workers, one simulation each.
  Results go to a shared ring when the page is isolated, and to transferred
  buffers otherwise.
- **Offline compilers:** threads across all cores. wgpu runs the LBM's WGSL
  shaders on the laptop's GPU, and the same shaders can run in a browser through
  WebGPU.
- **Browser builds of the compilers:** optional, for example a mass compile in
  the mod editor. They follow the shared build in
  [sky-WASM](https://github.com/foss-earth/sky/blob/main/sky-WASM/README.md):
  - fixed shared memory and atomics;
  - a thread-count setting;
  - a threaded → single → TypeScript ladder that reports what ran.

Isolation is not guaranteed. A frame, a missing service worker or the setting
turned off starts the page unisolated. So every shared path keeps a
transferred-buffer path, and the report says which one ran.

### Rules for each kernel

- **Reference first.** Write the TypeScript reference, then its Rust twin
  beside it in the owning package.
- **Parity.** Before production uses a kernel, it must match its reference
  bit for bit with the core's own trig substituted (ground-contact's
  `?wasm-trig`), and within a stated bound with V8's trig.
- **No allocation while stepping.**
  - Buffers are allocated once.
  - Inputs and outputs are `Float64Array` views over WASM memory.
  - Each kernel is one call per step.
  - A test counts heap allocations.
- **Fallback.** If a kernel throws, the module goes back to its TypeScript
  reference for good, and `status()` says why.
- **Builds.** Offline and locked, with a `build-metadata.json` receipt and no
  wasm-bindgen.
- **Measure, don't assume.** Rust is not automatically faster: FOSS Earth
  measured terrain heights and geometry slower in WASM.
  - Use paired A/B runs with an A/A control, on the same machine in the same
    power state.
  - Production keeps TypeScript for a kernel only where Rust measures slower.
  - *As built:* the multirotor WASM measures slower than its TypeScript (see
    [Numbers](#numbers)), yet the runtime moves to it once it loads. By this
    rule production would stay on TypeScript. That is
    [decision 7](#decisions-for-the-user).
- **Naming.** Call these WASM or Rust, never "native".

## Workstreams and implementation choices

Where two or three implementations are listed, they must pass the same tests,
and production keeps the cheapest one that passes. The others stay as test
oracles. A single implementation is used where alternatives would add nothing:
the rigid body (JSBSim), the contract and the telemetry path.

### 1. Motors and propellers

Each rotor must produce the following per step:

| Effect | Status |
| --- | --- |
| Thrust and shaft torque versus RPM, density and axial inflow | FGPropeller has it |
| Reaction torque and gyroscopic moment | FGPropeller has it |
| Motor current, power and speed dynamics | FGBrushLessDCMotor has it, but with no current limit and no battery |
| In-plane force (H-force or rotor drag), roughly proportional to RPM × in-plane airspeed at low speed ([Martin & Salaün 2010](https://hal.archives-ouvertes.fr/hal-00422423)) | Missing |
| Translational lift: thrust at fixed RPM rises with edgewise speed at small disk angles | Missing; FGPropeller only sees the axial component |
| Hub moments of rigid props in edgewise flow. NASA measured a lift offset of 5–8% of rotor radius at 20 ft/s, up to 13% at 40 ft/s | Missing |
| Ground effect, including partial ground effect ([Sanchez-Cuevas et al. 2017](https://doi.org/10.1155/2017/1823056)) | Missing |
| Vortex ring state in steep descent ([Johnson 2005](https://ntrs.nasa.gov/citations/20060024029)) | Missing. It matters for the Air 2S, whose 6 m/s descent is about 1.26 × its hover induced velocity |
| Interaction effects (detail below) | Missing |

NASA measured these interaction effects on similar quads:

- Rear rotors run in the front rotors' wake. The 3DR Solo trims at −6° pitch
  and 20 ft/s with rear rotors near 5,900 RPM and front rotors near 4,900 RPM.
- Hover download, where rotor wash pushes down on the body, was 5–15% of
  thrust.
- About 5% of rotor lift was lost to rotor–airframe and rotor–rotor interaction
  at 20 ft/s.

Four implementations:

| | A. Stock JSBSim plus XML | R. Rust plant kernel at the step boundary | B. Multirotor propeller class in our JSBSim fork | C. Offline blade-element oracle |
| --- | --- | --- | --- | --- |
| What | FGPropeller and the BLDC motor. Extra effects are hub-located `<external_reactions>` forces whose `<function>`s use RPM, air-relative body velocities and AGL. Translational lift goes through FGPropeller's two-dimensional C_T/C_P table slot: the `blade-angle` property, which is tied directly to the table's second axis, carries the edgewise ratio instead of pitch | FGPropeller and the BLDC motor for the axial part, as in A. A Rust/WASM kernel in `multirotor/wasm/`, called once per step before `run()`, computes H-force, translational lift, hub moments, ground effect and VRS from C's tables (J and μ), RPM, air-relative velocity and AGL. The inputs come from the step's existing property batch. It writes per-rotor magnitudes that hub-located `<external_reactions>` apply | A propeller subclass with C_T, C_P, H-force and hub-moment tables versus axial J and edgewise μ, plus ground-effect and VRS terms and runtime coefficient properties | Explicit blade-element models in the style of [Gill & D'Andrea 2019](https://www.mdpi.com/2504-446X/3/4/77), or a full BEMT, using chord and twist measured off the HD prop meshes and low-Reynolds airfoil data (tip Re ≈ 30–60k). It writes the tables for A, R and B and is a TypeScript reference for tests |
| For | No fork; ships with stage 1 | Rust/WASM; no fork and no pitch-slot reuse; full two-axis tables; about 1 µs per step, judging by ground-contact's kernels | Clean, fast, upstreamable, no property reuse | One physics source for every table |
| Against | Obscure: it reuses the pitch slot and needs many XML functions | Evaluated once per step from the pre-step state, which matches JSBSim only while there is one `run()` per step. Sub-stepping would need a call per sub-step | C++ change, WASM rebuild, fork upkeep | Accuracy depends on low-Re airfoil data |
| Role | First implementation, then R's cross-check | Production once it matches A on the same tables | Only if an effect must act inside JSBSim's own sub-steps | Produces the data and stays as the oracle |

FGRotor, JSBSim's helicopter rotor, has flapping, H-force and a crude
ground-effect term, but no gyroscopic moment, a linear lift curve without
stall, and a transmission built for helicopters. It diverged in the probe and
is a fallback only.

*As built, 28 October:* production uses neither A nor B. It runs R extended to
the whole rotor: [`rotorPlant.ts`](../../multirotor/src/rotorPlant.ts) and its
Rust twin.

- **What it covers.** It computes the axial part too: thrust, torque and
  reaction torque from blade element and momentum theory. It also covers the
  H-force, translational lift, hub moments, ground effect, the vortex ring and
  windmill states, and the gyroscopic moment. Every row of the table above is
  covered except the interaction effects.
- **No JSBSim propulsion.** There is no FGPropeller and no brushless motor.
  The rotor speed is a first-order lag on the controller's command, and the
  rotors' sum is applied at the CG.
- **No tables.** Blade element and momentum theory runs analytically per step,
  so it needs no tables from C. C is not built.
- **Tests.** The plant's tests check that:
  - the H-force grows linearly with edgewise speed;
  - ground effect matches Sanchez-Cuevas et al.;
  - the inflow through the vortex ring state is continuous, and the buffet
    peaks there;
  - the reaction torque's sign is right.

Calibration, in order of preference:

1. Static thrust and torque measurements of the real motor and prop, if the
   aircraft is available.
2. C's predictions, checked against similar published props: the
   [UIUC propeller database](https://m-selig.ae.illinois.edu/props/propDB.html),
   and NASA's Phantom 3 rotor (C_T ≈ 0.11–0.12 in propeller convention, figure
   of merit 0.58–0.67).
3. Closure against DJI's hover power and top speed.

### 2. Body aerodynamics: the aero compiler

This tool turns a 3D model into JSBSim's `<aerodynamics>` section, so drag,
lift or downforce, and aerodynamic moments follow the body's attitude relative
to the air.

What to expect: in NASA's tunnel test, the bare airframes of the small
quadcopters "all generate very little lift". The Phantom 3 airframe's drag/q
stayed between 0.21 and 0.26 ft² (0.020–0.024 m²) across ±40° pitch. The Mini 2
and Air 2S bodies are smaller and smoother, so small values are expected. The
larger vertical load is probably the hover download (5–15% of thrust in NASA's
tests). The compiler measures both instead of assuming downforce.

Pipeline:

1. **Geometry.**
   - Load the flight-configuration GLB with the arms unfolded and transform it
     to JSBSim structural axes.
   - Check its scale against the published dimensions.
   - Drop the propeller blades, which the rotor model covers. Keep motor bells,
     gimbal and feet.
   - Make it watertight with a winding-number voxelization, which tolerates
     holes in visual meshes. Export voxels for backend L and a surface for
     backend F.
2. **Sweep.**
   - Body-axis angles: α from −180° to 180°, β from −90° to 90°.
   - 5° steps inside the flight envelope: forward flight is nose-down, α ≈ −45°
     to +10°; climb and descent are near ±90°. Use 15–30° steps elsewhere.
   - Use left–right symmetry: CX, CZ and Cm are even in β; CY, Cl and Cn are
     odd.
   - Run two airspeeds to check Reynolds sensitivity (body Re ≈ 5×10⁴ to
     2×10⁵).
3. **Solve, rotors off.** Average loads over a converged window and record
   their spread as uncertainty.
4. **Solve, rotors on (stage 4b).** Put actuator disks at the hubs with hover
   and forward-flight thrust. Measure body download versus thrust and μ, and
   the mean inflow through each disk (front versus rear) for the rotor model.
5. **Compile.** Produce:
   - Six body-axis tables (X, Y, Z, roll, pitch, yaw) versus `aero/alpha-rad`
     and `aero/beta-rad`, scaled by `aero/qbar-psf` with unit reference area
     and length, with moments about the compiled CG.
   - A provenance JSON: mesh hash, solver, resolution, convergence and
     uncertainty.
   - A plot report in the tool's ignored `build/`.
6. **Verify.** Run the stage 3 and 4 flight-lab tests with the new tables.

Backends:

| | E. Empirical build-up | L. Lattice Boltzmann on the GPU | F. OpenFOAM RANS |
| --- | --- | --- | --- |
| Method | Hoerner-style component drag for body, arms, motors, gimbal and feet | D3Q19/D3Q27 with LES on voxels, as WebGPU compute (vendor-neutral, no meshing) | k-ω SST with snappyHexMesh. Native macOS build: [openfoam-app](https://github.com/gerlero/openfoam-app), GPL-3.0, run as an external program |
| Cost per attitude | Milliseconds | Minutes on a laptop GPU (estimate) | Tens of minutes on 8 cores (estimate) |
| Role | Stage 1 interim table; sanity check | Production sweeps | Reference on a subset; actuator-disk runs with [`rotorDisk`](https://cpp.openfoam.org/v12/rotorDisk_8H_source.html) or [`actuationDisk`](https://www.openfoam.com/documentation/guides/latest/doc/guide-fvoptions-sources-actuation-disk.html) |

[FluidX3D](https://github.com/ProjectPhysX/FluidX3D), a fast LBM solver, is
excluded because it is free only for non-commercial use.

Selection:

1. Validate L and F on canonical shapes (sphere, cube, Ahmed body) and check
   grid convergence (under 5% change).
2. Require L and F to agree within about 15% on drag area at three attitudes.
3. L then runs the sweeps. If they disagree, investigate before trusting either.

Sweeps are explicit, resumable batch jobs, run overnight on the laptop or on a
rented GPU, and never part of tests. Tests read the committed tables.

License: the HD meshes are CC BY-NC 4.0, so tables derived from them probably
carry the same terms. That is fine while the mods are unpublished. Publishing
needs a decision (see [Decisions](#decisions-for-the-user)).

### 3. Mass properties compiler

- **Inputs:**
  - Mesh parts.
  - Published masses: Mini 2 battery 86.2 g, Air 2S battery 198 g, Mini 2 prop
    about 0.6 g; totals under 249 g and 595 g.
  - Estimated ranges for motors, gimbal and camera, boards and shell.
- **Method:** closed-mesh mass integrals per part, with the unknown mass
  distributed so the total matches. A Monte Carlo run over the ranges gives the
  CG and the full inertia tensor, including products of inertia, with
  uncertainty.
- **Output:** `<mass_balance>` with the CG, the tensor, and `<pointmass>`
  entries for the battery and gimbal. Then a battery or payload change updates
  physics and controller together.
- **Single source:** the controller reads `inertia/*` from JSBSim, fixing D1's
  two sources.
- **Interim (stage 1):** the component estimate above.
- **Optional measurement** if the aircraft is available: a kitchen scale, the
  balance point and a bifilar pendulum, good to about ±5%.

*As built:* the XMLs carry the interim component estimate. The compiler is not
built.

| | Ixx / Iyy / Izz (slug·ft²) |
| --- | --- |
| Mini 2 | 2.360e-4 / 3.202e-4 / 5.113e-4 |
| Air 2S | 1.2118e-3 / 1.6566e-3 / 2.6545e-3 |

The fly runtime passes the live mass, inertia and density to the controller
each step.

### 4. Motor, ESC and battery

**Dropped on 28 October.** The simulation stops at the spinning propeller. The
reason is data, not cost: DJI publishes no motor data, and four JSBSim motors
cost 3.8 µs per step in the probe. A first-order rotor-speed lag in
`@0sfs/multirotor` stands in for this whole workstream. The plan below is kept
as history.

- **Motor:** JSBSim's BLDC model. Voltage is max volts × throttle; current is
  (V − RPM/Kv)/R; torque comes from (I − I₀)/Kv. DJI publishes no motor data.
  Kv, R and I₀ are identified so that hover throttle, hover power and climb
  margin land in range.
- **ESC:** a closed RPM loop (PI plus feed-forward from the motor model) with a
  current limit, as in the probe. A variant uses open-loop throttle with
  PX4-style thrust-curve linearization, for comparison.
- **Battery:**
  - One-RC Thevenin model: open-circuit voltage versus state of charge for 2S
    and 3S high-voltage LiPo, plus series resistance; state of charge
    integrates current.
  - It feeds the ESC by scaling throttle by V_batt / max volts. That is exact
    for an ideal PWM ESC, so JSBSim needs no change.
  - A constant avionics load covers the camera, gimbal and radio.
- **Implementations:** a TypeScript reference and its Rust/WASM twin in
  `multirotor`, run in the step (see
  [Rust/WASM, threads and the main thread](#rustwasm-threads-and-the-main-thread)).
  A JSBSim-side ESC (FCS `<pid>` channels) or battery (fork) comes only if
  sub-stepping needs it.
- **Targets:**
  - Mini 2 about 28–33 W average at 4.7 m/s (31 min on 17.32 Wh).
  - Air 2S about 69–81 W in hover (30 min on 40.42 Wh).
  - Both ranges assume 85–100% of the energy is usable.

### 5. Flight controller

The controller is a cascade like [PX4's](https://docs.px4.io/main/en/flight_stack/controller_diagrams),
in the lazy chunk, tuned to DJI's published envelope. DJI's firmware is closed,
so this matches behavior, not code.

1. **Commands (kept):** stick mapping, deadzone, holds with their corrected
   signs, yaw modes.
2. **Velocity → acceleration command,** with per-mode speed limits and
   acceleration and jerk limits (smooth, DJI-like braking; gentle Cine mode).
   Diagonal sticks respect the scalar speed limit (propulsion C3).
3. **Thrust vector = m·(a_cmd − g),** plus optional drag feed-forward from the
   controller's own nominal model.
   - The tilt limit clips only this commanded thrust. Drag comes from JSBSim
     and is never clipped (P2).
   - Collective thrust is non-negative by construction (P3).
4. **Attitude:** quaternion error, tilt-prioritized
   ([Brescianini & D'Andrea 2020](https://dblp.dagstuhl.de/pid/139/3560.html)),
   replacing Euler differences (propulsion C2).
5. **Body rates → torque,** with gyroscopic compensation and the live inertia.
6. **Allocation:** collective thrust and torques become four rotor thrusts,
   using the actual hub positions about the live CG.
   - Desaturation keeps thrust and roll/pitch and gives up yaw first, the
     weakest axis (see the yaw-torque row in the data table).
   - Per-rotor limits follow battery voltage.
7. **Thrust → RPM** through the controller's nominal propeller curve and its
   density estimate, then to the ESCs.

DJI's N, S and C modes come per aircraft from the specs.

A possible variant is an INDI inner loop (incremental dynamic inversion from
measured angular acceleration and motor speed), which tolerates model error
well. Try it in stage 2 only if the cascade struggles with saturation or wind.

*As built:* [`flightController.ts`](../../multirotor/src/flightController.ts) and
its Rust twin cover items 1–7, with these differences:

- **Velocity.** The velocity loop is PI on jerk-limited setpoints. There is no
  position loop: the kept stick holds supply it.
- **Attitude.** The tilt-prioritised attitude works from the rotation matrix.
- **Thrust.** A thrust-efficiency factor scales the static-thrust curve. That
  factor is the plant's last thrust over the nominal static thrust, which an
  accelerometer would see. This is the incremental part of INDI, applied to
  thrust rather than to angular acceleration. It is what lets the cascade
  hold height in a 5 m/s climb and at Sport speeds without a gain schedule.
- **Rotor limits.** Per-rotor limits are fixed, because there is no battery.
- **Modes.** Normal and Sport only. Cine is not built.

### 6. Environment and contact

- **Wind.** The host already writes the weather wind into JSBSim
  ([`applyWeather`](../../fly/src/flight/createFlightSimApp.ts#L2694)). Once
  the drone has body aerodynamics and propellers, JSBSim applies that wind to
  both automatically, because both use air-relative velocity. The private wind
  in `droneWind.ts` is deleted (E1, C11). Turbulence comes from JSBSim's
  built-in models, driven by the weather settings.
- **Density.** Propellers and aero tables use JSBSim's atmosphere
  automatically (E2). Hover RPM must scale with 1/√ρ up to the 4,000 m and
  5,000 m ceilings.
- **Contact.**
  - Four feet at the measured positions replace the single belly point (E5).
  - Body, arm and rotor probes go through the existing profile seams (E4).
  - No edits to `ground-contact`.

*As built:*

- **Done.**
  - **Wind.** `droneWind.ts` is gone. The weather wind reaches the drone only
    through JSBSim's atmosphere. The plant reads the air-relative body
    velocities `velocities/{u,v,w}-aero-fps`, and the XML's body drag uses the
    same air.
  - **Density.** The live density reaches both the plant and the controller,
    so hover RPM scales with altitude.
  - **Feet.** Four STRUCTURE feet; the runtime reads `contact/unit[0..3]/WOW`.
- **Not built.** Turbulence, the arm and rotor probes, and the service ceiling.

### 7. Sensors and estimator

This workstream covers M4, controls C10 and sensors S05.

- **Sensors:**
  - IMU, whose specific force includes rotor drag. Real multirotors sense
    airspeed this way.
  - Barometer.
  - GPS with latency and noise.
  - Downward vision (Mini 2 0.5–10 m; Air 2S time-of-flight 0.1–8 m).
  - Magnetometer.
- **Estimator:** a compact EKF. The controller then flies on estimates.
- **Gate:** hover scatter matches DJI's hover accuracy. For the Mini 2 that is
  vertical ±0.1 m with vision or ±0.5 m with GPS, and horizontal ±0.3 m or
  ±1.5 m.

### 8. Presentation

- Props, the engine widget and sound read the simulated RPM directly, so the
  "estimated" label goes.
- Visuals use the measured radii.
- Blade-passing frequency drives the sound: 2 blades × 9,000 RPM ÷ 60 ≈ 300 Hz
  for a hovering Mini 2.
- The [`rotating-prop`](../../rotating-prop/README.md) swept-volume look stays.

*As built:*

- **Rotor speed.** Props and the engine monitor read each rotor's simulated
  speed. The telemetry's source is `"simulated"`, and the maximum RPM comes
  from the propeller.
- **Procedural model.** It puts the motors at the real hubs relative to the CG,
  with blades of the measured radius.
- **Not built.** Sound.

## Compute budget

| Per physics step | Cost | Basis |
| --- | ---: | --- |
| JSBSim: four motors and propellers, rigid body, four contacts | 3.8 µs | Probe |
| Batched property I/O | 0.1 µs | Probe |
| Body aero tables (six 2-D lookups) | ~1 µs | Estimate |
| Rotor extras, implementation A (XML functions) | ~2–5 µs | Estimate; measured in stage 3 |
| Rotor extras, implementation R (Rust kernel) | ~1 µs | Estimate: ground-contact's full C172 step takes 1.08 µs; measured in stage 3 |
| Controller, allocation, ESC and battery (TypeScript reference; the Rust twin is measured against it) | ~5–15 µs | Estimate |
| Estimator (stage 7) | ~10–30 µs | Estimate |
| **Total** | **< 60 µs** | **< 0.7% of a core at 120 Hz; < 3% at 480 Hz** |

Every row runs on the main thread inside the fixed step, which is why each must
stay in microseconds.

Measured on 28 October for the design that was built, which replaces the
JSBSim motors, the rotor extras, the ESC and the battery rows. The figures are
`multirotor/wasm/scripts/bench.mjs` for the Mini 2, Node 26 on an Apple M5, on
battery:

| Per physics step | TypeScript | WASM |
| --- | ---: | ---: |
| Controller and rotor plant, hover | 4.7 µs | 9.4 µs |
| Controller and rotor plant, Sport at 16 m/s | 3.7 µs | 12.1 µs |
| Controller and rotor plant, vortex ring | 10.9 µs | 16.8 µs |

That is under 0.2% of the 8.3 ms step at 120 Hz.

The real-JSBSim lab, `fly/src/flight/drone/quadJsbsimLab.test.ts`, also timed the
whole step. That step is one batched read, the TypeScript controller and plant, one
batched write and JSBSim's `run()`. Averaged over 11,281 steps across hover,
speed, vertical, wind and landing runs in Node 26.9, it costs 14.2 µs for the
Mini 2 and 12.2 µs for the Air 2S, under 0.2% of the step.

Offline work takes hours for a CFD sweep, run as a batch job, and seconds for
blade-element tables and mass compilation.

**Physics rate.** Stage 0 decides it:

- Fly the same scripted maneuvers at 120 and 480 Hz in the flight lab. Keep
  120 Hz if the responses agree within tolerance.
- Otherwise there are two options:
  - Let drones run at 240 or 480 Hz. That needs `PHYSICS_RATES_HZ` in
    [`physicsRate.ts`](../../fly/src/flight/physics/physicsRate.ts#L13)
    extended, and the app's own per-step overhead measured; that overhead is
    the unknown.
  - Sub-step JSBSim inside one accepted step, with the ESC loop in JSBSim FCS
    channels.
- Either way, accepted-step semantics (C7) and the terrain-contact seam must
  hold.

## Validation targets

Published by DJI. Sources: [Mini 2 spec copy](https://www.cliftoncameras.co.uk/uploads/specifications/DJI%20Mini%202%20Drone%20%20Specifications.pdf),
[DJI Mini 2 support](https://www.dji.com/support/product/mini-2),
[Air 2S spec copy](https://www.cliftoncameras.co.uk/uploads/specifications/Clifton%20Cameras%20DJI%20Air%202S%20Drone%20Product%20Specifications.pdf).

| Quantity | Mini 2 | Air 2S |
| --- | --- | --- |
| Takeoff mass | < 249 g | 595 g |
| Diagonal | 213 mm | 302 mm |
| Max speed S / N / C | 16 / 10 / 6 m/s | 19 / 15 / 5 m/s |
| Max tilt S / N | 40° / 25° | 35° / 25° (35° in strong wind) |
| Max yaw rate S / N / C | 130 / 60 / 30 °/s (adjustable to 250) | 250 / 90 / 60 °/s |
| Ascent | 5 / 3 / 2 m/s | 6 m/s |
| Descent | 3.5 / 3 / 1.5 m/s | 6 m/s (verify against DJI's sheet) |
| Service ceiling | 4,000 m | 5,000 m |
| Flight time | 31 min at 4.7 m/s, no wind | 31 min; hover 30 min |
| Wind resistance | 8.5–10.5 m/s | 8.5–10.5 m/s |
| Battery | 2S, 7.7 V, 2,250 mAh, 17.32 Wh, 86.2 g | 3S, 11.55 V, 3,500 mAh, 40.42 Wh, 198 g |
| Propeller | 4726F (4.7 × 2.6 in) | 7238F (7.2 in) |

Derived:

| Quantity | Mini 2 | Air 2S |
| --- | --- | --- |
| Total drag area at Sport top speed | ≤ 0.0131 m² | ≤ 0.0185 m² |
| Hover induced velocity v_h | ≈ 4.7 m/s | ≈ 4.8 m/s |
| Max descent / v_h | 0.74 (near vortex-ring onset) | 1.26 (inside the vortex-ring band for vertical descent) |
| Hover RPM, C_T 0.08–0.14 | ≈ 7,900–10,500 | ≈ 5,200–6,900 |
| Average electrical power | ≈ 28–33 W at 4.7 m/s | ≈ 69–81 W in hover |

Outside data. These check model structure and trends, not Mini 2 or Air 2S
numbers:

- **NASA Ames tunnel test** ([Russell et al. 2016](https://rotorcraft.arc.nasa.gov/Publications/files/72-2016-374.pdf);
  [program page](https://rotorcraft.arc.nasa.gov/Research/Programs/UAS_Performance_Test.html)).
  It covers five multicopters including a DJI Phantom 3, and found:
  - Near-zero bare-airframe lift.
  - Drag rising with RPM at nose-down pitch.
  - Rear rotors faster in trim.
  - 5–15% hover download.
  - Hub roll moments.
  - Motor and ESC efficiency of 50–77%, rising with RPM.
- **Martin & Salaün:** rotor drag dominates horizontal accelerometer readings.
- **Sanchez-Cuevas et al.:** ground-effect curves.
- **Johnson:** the vortex-ring model.
- **[NeuroBEM](https://rpg.ifi.uzh.ch/NeuroBEM.html) (optional):** a public
  dataset of a racing quad at up to 65 km/h. It can test the rotor and body
  code path on another vehicle.
- **Flight logs, if available.**
  - What they contain: DJI Fly `.txt` logs hold attitude, velocities, sticks
    and battery data.
  - Parser: [`dji-log-parser`](https://github.com/lvauvillier/dji-log-parser)
    reads them.
  - Encryption: version 13 and later logs are encrypted. Each log needs a
    keychain from DJI's API, which takes a DJI developer key and a request to
    DJI.
  - Motor speeds: reportedly only in the aircraft's `.DAT` logs. That is
    unverified for these two aircraft.

## Stages

Stages 3 and 4 can run in parallel; stage 9 waits on FOSS Earth; everything
else is in order. Each gate is a test in the flight lab, not a visual
impression.

### Stage 0 — Flight lab (no behavior change)

**Status, 28 October: partly done, differently.** There are two flight labs:

- **`multirotor/tests/rigidBody.ts`** closes the loop on a flat-earth rigid body.
- **`fly/src/flight/drone/quadJsbsimLab.test.ts`** loads the real mod XML into
  the WASM JSBSim. It flies the plant and controller through these cases:
  - settling on the feet;
  - hover;
  - each mode's speed;
  - climb and descent;
  - a 10 m/s wind;
  - landing.

The model was replaced instead of first encoding the old model's failures. The
real-JSBSim lab passes all 22 cases, including full-descent and recorded-flight
landing regressions in stage 6. The lab measured the per-step cost; see the
compute budget. The 120 versus 480 Hz comparison is not measured. The labs run in vitest's own workers,
not a custom pool.


- Turn the probe into a headless vitest harness in fly. It loads the real mod
  XML into the WASM JSBSim, runs scripted maneuvers with fixed seeds, and
  records traces.
- Encode P1–P5, M1 and the validation targets as tests. Mark the ones the
  current model fails as known failures.
- Keep the 100-knot C172 → Mini 2 switch as a scenario (E8).
- Run its batches in a worker pool, one simulation per worker, never on the
  page.
- Measure 120 versus 480 Hz, and the app's per-step overhead.
- **Gate:** the suite runs in under about 30 s on the laptop, and the current
  model fails P1–P4 exactly as the audits predict.

### Stage 1 — Four real motors (the core fix)

**Status, 28 October: done differently.** There are four rotor plants with a
first-order speed lag instead of JSBSim BLDC motors, propellers and ESCs, and
no battery. Other differences from the plan below:

- **Profile.** `engine: "none"` stays, with an ARM gauge on `multirotor/armed`.
  The engine monitor shows simulated RPM per rotor.
- **Removed, or being removed in fly:** `planDroneForce`, the `fcs/drone-*`
  forces, the private wind and `estimateDroneMotorRpm`.
- **Data.** The data fixes are in.
- **Gates.** The flight labs cover P1–P5 and M1 structurally: the wrench is
  the sum of the four rotors, there are no ground overrides, and RPM is the
  thrust state. Current and power are not modelled.


- **New Mini 2 and Air 2S XML:**
  - Four BLDC motors with propellers at the measured hubs, with correct radii
    and spin senses.
  - Interim mass (the component estimate).
  - Interim body aero (empirical backend E, fitted inside the drag bounds).
  - Four feet.
- **Minimum new controller:** workstream 5, steps 1–7, with simple gains, plus
  the ESC loop.
- **Kernels:** the controller, allocation, ESC and battery as TypeScript
  references in `multirotor/src/`, then their Rust twins in `multirotor/wasm/`.
- **Removed:** `planDroneForce`, the external forces, the private wind and
  `estimateDroneMotorRpm`.
- **Profile:** a multirotor engine kind in
  [`fdmProfiles.ts`](../../fly/src/flight/jsbsim/fdmProfiles.ts), with
  per-motor RPM and current gauges and an arming state.
- **Data fixes:** radii, the hover-RPM field, Iyy, mode limits.
- **Gate:**
  - **P1:** rolled 90° with every motor at hover RPM, the vertical rotor force
    is about zero.
  - **P2:** body drag at a given airspeed does not depend on the tilt limit or
    controller state.
  - **P3:** every rotor's thrust is ≥ 0 and ≤ its current maximum. Fast
    descent produces no downward rotor force.
  - **P4:** the total moment equals Σ r × F plus reaction and gyroscopic
    moments of the four rotors, checked against JSBSim's outputs.
  - **P5:** no force overrides near the ground; landing goes through contacts.
  - **M1:** a motor step response is bounded. Gauges, props and sound read the
    same RPM states as thrust.
  - Hover RPM, current and power fall inside the derived ranges. Each mode's
    top speed is reached at or under its tilt limit. A 10 m/s wind needs a
    plausible lean.
  - The hold-sign test passes, and the import-boundary test shows no drone code
    at startup.
  - Each Rust kernel matches its TypeScript reference (bit for bit with the
    core's trig) and allocates nothing over 20,000 steps. Its per-step time is
    recorded against the reference's.

### Stage 2 — DJI-like flight controller

**Status, 28 October: partly done.** Done:

- the cascade, with tilt-prioritised attitude from the rotation matrix;
- yaw-last desaturation and jerk-limited setpoints;
- Normal and Sport modes, with each aircraft's limits;
- live mass, inertia and density.

The envelope tests pass in both labs. Not done:

- Cine mode;
- the 10.5 m/s wind and saturation stress tests;
- the 100-knot switch;
- controls C1 and C9.


- **Build:**
  - The full cascade: quaternion attitude, desaturation, jerk-limited braking,
    and N/S/C modes.
  - The controller reads live mass and density.
  - On the way, fix controls C1 (phone payload bypasses blending and pilot
    neutralization) and C9 (stale AGL).
- **Gate:**
  - The per-mode envelope holds: speeds, tilts, yaw rates, climb and descent.
  - Flight stays stable under saturation and in 10.5 m/s wind.
  - In the 100-knot switch, the drone pitches back before it brakes, and its
    deceleration never exceeds what tilt-limited thrust plus drag can supply.

### Stage 3 — Propeller aerodynamics beyond axial flow

**Status, 28 October: built analytically, without the oracle.** The rotor
plant computes all of these effects itself, so C, A and B are not built. Tested:

- H-force linearity;
- ground effect against Sanchez-Cuevas et al.;
- vortex-ring inflow and buffet;
- the Air 2S holding 6 m/s descent under control.

NASA's rear-rotor trend needs rotor-wake interference, which is out of scope.


- **Build:**
  - The blade-element oracle C and its tables.
  - Implementation A, then R: rotor drag, translational lift, hub moments,
    ground effect and VRS. B only if R cannot pass.
- **Gate:**
  - The tables match C within tolerance.
  - R matches A on the same tables, step for step, within a stated bound.
  - H-force is linear in RPM × speed at low speed, and the accelerometer shows
    rotor drag in steady flight.
  - Thrust gain near the ground follows Sanchez-Cuevas et al.
  - Axial descent shows vortex-ring thrust loss and fluctuation between about
    0.5 and 1.5 v_h. The Air 2S must still descend at its published 6 m/s under
    control; if it can't, the model or the DJI-like descent logic must account
    for that.
  - NASA's trends are reproduced in sign and rough size.

### Stage 4 — Aero compiler

**Status, 28 October: deferred.** The XML has interim per-axis body drag
areas, inside the drag bounds. There are no lift or moment terms and no hover
download.


- **Build:**
  - The tool with backends E, L and F, plus solver validation.
  - Mini 2 and Air 2S rotor-off tables.
  - Stage 4b: rotor-on download and per-rotor inflow.
- **Gate:**
  - The solver checks pass.
  - With the compiled body tables plus rotor drag, each drone reaches its
    Sport top speed at or under its tilt limit.
  - Hover download falls in a plausible range: NASA's similar quads showed
    5–15%.
  - Rear rotors spin faster than front rotors in forward flight.

### Stage 5 — Energy

**Status, 28 October: dropped** by the user's decision to stop at the spinning
propeller. The plan below is history.


- **Build:** battery, ESC current limits, avionics load and low-battery
  behavior (E6).
- **Gate:**
  - Flight and hover times fall within ±10% of DJI's figures.
  - Voltage sag reduces climb.
  - Depletion produces DJI-like warnings, return and landing.

### Stage 6 — Environment and contact

**Status, 28 October: partly done.** Done:

- the weather wind through JSBSim;
- live density;
- four feet.

Not done: turbulence, vertical wind, the ceiling checks and the collision
probes.

Landing tests now pass for a 1 m/s approach, full Normal and Sport descents
from 40 m, and the climb, hover, forward-flight and landing sequence through
the shipped controller. The flare, idle on contact and one second of grounded
descent stick to disarm remain. The flare limits descent to
`max(0.1, (agl - touchdownAgl) * 0.5)` m/s;
`touchdownAgl` comes from the loaded XML's CG height above its lowest foot
(0.0232 m Mini 2, 0.0347 m Air 2S). The earlier 1.5 s⁻¹ slope passed the
1 m/s test but braked too late from the Air 2S's 6 m/s descent. The
0.5 s⁻¹ slope starts that slowdown about 12 m above the feet. Both aircraft
now touch down at about 0.06 m/s. All tested landings remain below 1.63 g
and 0.72 mm rebound, against the 3 g and 5 mm limits, with positive CG
clearance. They finish level on all four feet and disarmed. The feet hold
both aircraft level at rest, with the CG about 10 mm up. Touchdown speed
is still a feel decision for user testing.


- **Build:** turbulence from weather, vertical wind, density and ceiling
  checks, feet and collision probes.
- **Gate:** E1, E2, E4 and E5 pass their acceptance tests as written in the
  audit.

### Stage 7 — Sensors and estimator

**Status, 28 October: deferred.** The controller flies on true state.


- **Gate:**
  - Hover scatter matches DJI's accuracy.
  - The controller is stable on estimated state with realistic delays.

### Stage 8 — Validation against flight data

**Status, 28 October: deferred.** No flight logs.


- If logs exist, replay their sticks, compare traces and identify the remaining
  parameters.
- Record an error envelope per maneuver (V1–V3).

### Stage 9 — The step in a worker (conditional)

**Status, 28 October: blocked** on FOSS Earth's steps 6 and 9.


- **Starts only when** FOSS Earth's steps 6 (shared-memory ground) and 9
  (JSBSim in a worker) exist. See
  [What has to stay on the main thread](#what-has-to-stay-on-the-main-thread).
- **Build:** run the drone's whole step in the physics worker, reading the
  ground in place.
- **Gate:**
  - Rerun the 2026-10-21 measurement for the Mini 2 and Air 2S below 60 m,
    with sound, against
    [its shipping gate](../../fly/docs/performance/2026-10-21/flight-dynamics-worker.md):
    at least 1 ms less main-thread work per frame, or at least 10% better p95
    main-thread frame time. Either one must come with a repeatable
    responsiveness improvement and no regression in contact, audio or input
    tests. The worker's extra frame of latency counts against responsiveness,
    and predicting it away would change behavior.
  - A scripted flight, with inputs keyed to steps and its ground recorded, gives
    the same trajectory bit for bit in the worker and on the main thread.
- If it fails, the step stays on the main thread, and the result is published
  like the 21 October one.

## Placeholder inventory

Kept as is: the lazy gate, command mapping, deadzone, hold anchors and their
corrected signs, the camera, gimbal and capture pipeline (its defects stay in
the sensors audit), and `rotating-prop`.

The "Today" column is the state before the rebuild. The last column is the
status on 28 October; "in progress" means fly's integration was still being
written that day.

| Item | Today | Action | Stage | Status, 28 October |
| --- | --- | --- | --- | --- |
| [`droneController.ts`](../../fly/src/flight/drone/droneController.ts) `planDroneForce`, `createDroneController` | World-frame force planner. Drag sits inside the tilt clip. AGL overrides. Euler rate commands. It throws unless dt = 1/120 | Replace with the cascade, allocation and ESC commands. Keep the command and hold code | 1–2 | In progress: `planDroneCommand` turns sticks and holds into a velocity command for `@0sfs/multirotor`, with arming and a landing slowdown. The 120 Hz guard stays |
| [`droneController.test.ts`](../../fly/src/flight/drone/droneController.test.ts) | Two force-plan cases | Replace with flight-lab invariants; keep the hold-sign test | 0–1 | In progress: command, hold-sign and arming tests here; flight invariants in the two labs |
| [`droneMotorTelemetry.ts`](../../fly/src/flight/drone/droneMotorTelemetry.ts) `estimateDroneMotorRpm` | Inverse estimate: 6,000 RPM hover, ideal mixer, arm/4 yaw lever | Delete. Read `propulsion/engine[n]/propeller-rpm`, current and thrust. Keep the position and direction helpers if they match the XML | 1 | Estimate deleted; the position and direction helpers stay. RPM comes from the rotor plant, as there is no JSBSim propulsion |
| `droneWind.ts` | Private random wind | Delete; JSBSim applies weather wind and turbulence | 1, 6 | Deleted; weather wind through JSBSim. No turbulence |
| [`droneTypes.ts`](../../fly/src/flight/drone/droneTypes.ts) | `K_V`, `K_POS`, `K_ATT`, `K_RATE`, `RHO`, `PHYSICS_DT`; profile with Iyy = Ixx | Per-aircraft controller config. Mass and density read from JSBSim. FOV helpers stay | 1–2 | In progress: gains are per aircraft in the manifest; mass, inertia and density are read live |
| [`droneFeature.ts`](../../fly/src/flight/drone/droneFeature.ts) | Steps the controller with the private wind; feeds the estimated RPM to props and widget | Motor commands, batched property I/O, simulated RPM to props and widget | 1 | In progress: [`droneFlightModel.ts`](../../fly/src/flight/drone/droneFlightModel.ts) does the batched I/O |
| [Mini 2 XML](../../DJI-Mini2/public/jsbsim-data/aircraft/dji-mini2/dji-mini2.xml), [Air 2S XML](../../DJI-Air-2S/public/jsbsim-data/aircraft/dji-air2s/dji-air2s.xml) | Zero aero, no propulsion, LOCAL forces, one belly contact, point-mass inertia | Generated propulsion, aerodynamics, mass and four feet | 1, 3, 4 | Rewritten by hand: BODY rotor wrench at the CG, per-axis body drag, component-estimate mass, four feet. No propulsion |
| Aircraft manifests `cdA`, `ixxKg`, `izzKg` | Hand values: drag too high, inertia about 3× high | Remove; the compiled XML holds them | 1 | Removed; the XML holds them |
| Manifest rotor `radiusMeters`, `rotorHoverRpm` | Radii 25–42% short; assumed hover RPM | Measured radii; delete the hover RPM | 1 | Done |
| [`Quadcopter` manifest](../../Quadcopter/public/0sfs-mod.json#L21) gains, `wind`, `airDensity` | Shared constants | Delete wind and density; gains move to per-aircraft controller config. `kinematic` can remain as the controller selector | 1–2 | Done; `kinematic` remains |
| Profiles' `engine: "none"`, `RUN` gauge, external-force `initialProperties` | Placeholder engine and gauge | Multirotor engine kind, per-motor gauges, arming state | 1 | `engine: "none"` stays, with an ARM gauge on `multirotor/armed`. Per-rotor RPM goes to the engine monitor rather than gauges |
| `droneGate.ts` controls C1; `createFlightSimApp.ts` controls C9 | Raw phone payload; previous frame's AGL | Fix with the controller rewrite | 2 | Not addressed |
| Procedural blades, 45 mm | Stand-in size | Measured radii | 1 | In progress: real hubs about the CG, measured radii |
| [`rotating-prop` README](../../rotating-prop/README.md) | "Quadcopter RPM is currently a visual estimate" | Update when stage 1 lands | 1 | Updated to simulated rotor speed |

## Audit traceability

| Finding | Stage |
| --- | --- |
| P1–P5, M1, propulsion C1 | 1 |
| Propulsion C2–C3, controls C1, C9 | 2 |
| M2 | 1 (axial), 3 (full) |
| M3, E6 | 5 (dropped) |
| M4, controls C10, sensors S05 | 7 |
| D1 | 1 (interim), 4 (compiled mass) |
| D2 | 1, 4, 6 |
| E1, controls C11 | 1 (weather wind through JSBSim), 6 (turbulence) |
| E2 | 1 |
| E3 | 3 |
| E4, E5 | 6 |
| E7, controls C7 | 0 (rate measurement), with the rate decision |
| E8 | 0 (scenario kept) |
| Sensors S10 | 1 (RPM source), later (sound) |
| V1–V3 | 0, 8 |

Controls C2–C6, C8, C12 and sensors S01–S04, S06–S09 are outside this
roadmap and stay in their audits.

## Contract changes

Each stage updates [SPEC-360-DRONE](SPEC-360-DRONE.md) as it lands. As updated
on 28 October:

- **§4.1:** the inertia rows and point-mass model became the component estimate
  in the XML. The per-mode tilt, speed, yaw-rate, climb and descent rows come
  from the aircraft's `quad.modes`. Sensor, FOV and camera content stays.
- **§4.2:** `engine: "none"` stays, with an ARM gauge on `multirotor/armed`; the
  engine monitor shows per-rotor simulated RPM. The `fcs/drone-f*`/`m*`
  initial properties became `multirotor/armed`. `requiresPhysicsHz` stays 120;
  stage 0 has not measured otherwise.
- **§4.3:** the zero-aero XML with LOCAL forces became BODY rotor forces and
  moments at the CG, per-axis body drag, the component-estimate mass and four
  feet. There is no propulsion section.
- **§4.4:** command mapping, deadzone, hold anchors and signs, and yaw modes
  stay. The velocity-P law, tilt clip, drag-in-controller, AGL overrides and
  force/moment law became `@0sfs/multirotor`'s controller and rotor plant, plus
  arming and a landing slowdown.
- **§4.5:** the private Ornstein–Uhlenbeck wind and drag law are deleted, in
  favor of JSBSim's weather wind and the XML's body drag. Turbulence is not
  wired.

## Decisions for the user

1. **CFD geometry and license.** The HD meshes are CC BY-NC 4.0. There are
   three options:
   - (a) Use them and keep derived tables under the same terms. This is fine
     while the mods are unpublished.
   - (b) Build owned, simplified geometry from published dimensions. This is
     more work, but publishable.
   - (c) Do both: owned geometry for publishing, HD for cross-checks.
2. **JSBSim fork changes.** Is a multirotor propeller class (implementation B)
   in the fork acceptable, and should it be offered upstream later? Stage 1
   needs no fork. *Moot as built:* the rotors live in `@0sfs/multirotor`, and
   JSBSim needs no change.
3. **Aircraft access.** If a Mini 2 or Air 2S is available, a scale, a pendulum
   and a small thrust stand would replace estimates with measurements.
4. **Flight logs.** Decrypting modern DJI logs means sending a per-log request
   to DJI's API with a developer key. Is that acceptable?
5. **Physics rate.** If stage 0 shows 120 Hz is not enough, choose between
   letting drones run at 240 or 480 Hz and sub-stepping JSBSim. Still open: the
   labs fly well at 120 Hz, but the 480 Hz comparison has not been run.
6. **Motor, ESC and battery.** *Decided on 28 October:* simulate down to the
   spinning propeller only. A first-order rotor-speed lag stands in for the
   ESC, motor and battery. The reason is data, not cost: DJI publishes no motor
   data, and the probe's four motors cost 3.8 µs per step. Flight time,
   voltage sag and low-battery behavior are therefore not simulated (stage 5).
7. **WASM or TypeScript in production.** The multirotor WASM matches its
   TypeScript bit for bit, but measures slower (9.4 against 4.7 µs per step in
   hover). The runtime moves to WASM once it loads, following the instruction
   to use Rust/WASM wherever possible. The [kernel rules](#rules-for-each-kernel)
   would keep TypeScript where Rust measures slower. Both cost under 0.2% of the
   step, so this is a choice of policy, not of performance.

## Sources

- DJI specifications: [Mini 2 spec copy](https://www.cliftoncameras.co.uk/uploads/specifications/DJI%20Mini%202%20Drone%20%20Specifications.pdf), [DJI Mini 2 support](https://www.dji.com/support/product/mini-2), [Air 2S spec copy](https://www.cliftoncameras.co.uk/uploads/specifications/Clifton%20Cameras%20DJI%20Air%202S%20Drone%20Product%20Specifications.pdf).
- C. Russell, J. Jung, G. Willink, B. Glasner, [Wind Tunnel and Hover Performance Test Results for Multicopter UAS Vehicles](https://rotorcraft.arc.nasa.gov/Publications/files/72-2016-374.pdf), AHS Forum 72, 2016 ([NTRS](https://ntrs.nasa.gov/citations/20160007399)).
- R. Gill, R. D'Andrea, [Computationally Efficient Force and Moment Models for Propellers in UAV Forward Flight Applications](https://www.mdpi.com/2504-446X/3/4/77), Drones 3(4):77, 2019.
- L. Bauersfeld et al., [NeuroBEM: Hybrid Aerodynamic Quadrotor Model](https://arxiv.org/abs/2106.08015), RSS 2021.
- P. Martin, E. Salaün, [The True Role of Accelerometer Feedback in Quadrotor Control](https://hal.archives-ouvertes.fr/hal-00422423), ICRA 2010.
- P. J. Sanchez-Cuevas, G. Heredia, A. Ollero, [Characterization of the Aerodynamic Ground Effect and Its Influence in Multirotor Control](https://doi.org/10.1155/2017/1823056), Int. J. Aerospace Eng., 2017.
- W. Johnson, [Model for Vortex Ring State Influence on Rotorcraft Flight Dynamics](https://ntrs.nasa.gov/citations/20060024029), NASA/TP-2005-213477.
- D. Brescianini, R. D'Andrea, Tilt-Prioritized Quadrocopter Attitude Control, IEEE TCST 28(2), 2020 ([dblp](https://dblp.dagstuhl.de/pid/139/3560.html)).
- [PX4 controller diagrams](https://docs.px4.io/main/en/flight_stack/controller_diagrams); [UIUC propeller database](https://m-selig.ae.illinois.edu/props/propDB.html).
- OpenFOAM [`rotorDisk`](https://cpp.openfoam.org/v12/rotorDisk_8H_source.html) and [`actuationDisk`](https://www.openfoam.com/documentation/guides/latest/doc/guide-fvoptions-sources-actuation-disk.html); [openfoam-app](https://github.com/gerlero/openfoam-app); [FluidX3D](https://github.com/ProjectPhysX/FluidX3D).
- [`dji-log-parser`](https://github.com/lvauvillier/dji-log-parser).
- Rust/WASM and threads:
  - FOSS Earth: [WebAssembly cores](https://github.com/foss-earth/docs/blob/main/docs/webassembly-cores.md); [WebAssembly builds](https://github.com/foss-earth/docs/blob/main/docs/proposals/webassembly.md), steps [6](https://github.com/foss-earth/docs/blob/main/docs/proposals/webassembly.md#6-ground-height-grids-and-bvhs) and [9](https://github.com/foss-earth/docs/blob/main/docs/proposals/webassembly.md#9-jsbsim-in-a-worker); [sky-WASM](https://github.com/foss-earth/sky/blob/main/sky-WASM/README.md).
  - 0SFS: [flight dynamics worker plan](../../fly/docs/performance/2026-10-21/flight-dynamics-worker.md) and [result](../../fly/docs/performance/2026-10-21/flight-dynamics-worker-results.md), 21 October 2026; [ground-contact's WASM kernels](../../ground-contact/wasm/README.md); [`audioTransport.ts`](../../sound/src/audioTransport.ts); [`flightRecorder.ts`](../../logging/src/flightRecorder.ts).
  - MDN: [`Atomics.wait`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Atomics/wait); [origin private file system](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system).
  - [wgpu](https://github.com/gfx-rs/wgpu).
- JSBSim fork sources read for this plan, under `~/gh/Felipegalind0/jsbsim/src/`:
  - `models/propulsion/FGPropeller.cpp`: axial inflow only; `blade-angle` tied to the second table axis; reaction and gyroscopic moments.
  - `FGBrushLessDCMotor.cpp`.
  - `FGRotor.cpp`.
  - `models/FGPropulsion.cpp`: the `Engines`/`engine` lookup.
  - `input_output/FGModelLoader.cpp`: `file=` includes.
  - `models/FGAerodynamics.cpp`: X/Y/Z body axes.
