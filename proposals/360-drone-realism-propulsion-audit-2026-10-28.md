# Drone realism audit: propulsion and flight dynamics

> **Partly history.** The flight-model findings here describe the drone before the 28 October 2026 rebuild. The force controller, the `fcs/drone-*` properties, the estimated RPM and the private wind have since been replaced by the rotor plant and flight controller in [`multirotor/`](../../multirotor/README.md). The [flight-model roadmap](360-drone-flight-model-roadmap-2026-10-28.md)'s stage statuses and placeholder inventory say which findings are fixed and which are still open. Findings outside the flight model are unaffected.

28 October 2026. Technical appendix for the broader drone realism audit. This
documents the local implementation inspected on this date; it does not change
the implementation or the existing [360-drone contract](SPEC-360-DRONE.md).

## Assessment

The drone uses JSBSim to integrate rigid-body motion, but the force-generating
plant is an idealized velocity/attitude controller. It does not simulate four
electric motors and propellers. The displayed motor speeds are inferred after
the controller has already applied aggregate forces and moments. They are not
the states from which those forces were generated.

Two distinctions matter:

- Running a force through a rigid-body integrator does not establish that the
  force itself can be produced by the modeled aircraft.
- A 120 Hz integration rate and passing software tests establish neither
  aerodynamic accuracy nor agreement with a real Mini 2 or Air 2S.

Some findings below are shortcuts explicitly required by the current contract.
Here, **physical error** means incompatible with a fixed-rotor aircraft model;
it does not necessarily mean a deviation from that contract. **Missing model**
means an absent physical subsystem. **Uncalibrated data** means a parameter or
assumption whose real-aircraft accuracy has not been established by this audit.
These categories should not be conflated.

The user's C172-to-drone change at approximately 100 knots is a deliberate
stress test. Preserving a high initial velocity is a useful test condition. The
findings below do not depend on removing that condition or resetting the drone
to hover.

## What currently generates motion

The current sequence is:

```text
stick / position error
  → desired NED velocity
  → proportional acceleration request
  → gravity compensation + simplified drag
  → tilt-limited world force and independent body moments
  → JSBSim rigid-body integration

the same force/moment request
  → inverse ideal-X mixer
  → estimated RPM and visible prop rotation
```

The XML has mass, inertia, a belly contact and external reactions. It has no
`<propulsion>` element and its six aerodynamic coefficient functions are zero.
The three external forces use `frame="LOCAL"`; the three direct moments use
`frame="BODY"`. See the [Mini 2 XML, lines 20–71](../../DJI-Mini2/public/jsbsim-data/aircraft/dji-mini-2/dji-mini-2.xml#L20)
and the [Air 2S XML](../../DJI-Air-2S/public/jsbsim-data/aircraft/dji-air-2s/dji-air-2s.xml#L20).
JSBSim defines LOCAL as north/east/down, independently of aircraft orientation;
BODY is forward/right/down. Its external-force facility legitimately accepts
arbitrary forces; the calling aircraft model must decide which forces are
physically possible. See the [official external-force reference](https://jsbsim-team.github.io/jsbsim/classJSBSim_1_1FGExternalForce.html).

The controller writes these six aggregate properties at
[droneController.ts, lines 228–233](../../fly/src/flight/drone/droneController.ts#L228).
The feature calculates motor RPM only afterward at
[droneFeature.ts, lines 183–185](../../fly/src/flight/drone/droneFeature.ts#L183).

## Demonstrated physical errors

### P1 — Thrust direction does not follow actual aircraft attitude

**Impact: critical. Confidence: direct source proof.**

[Force construction, lines 91–120](../../fly/src/flight/drone/droneController.ts#L91)
does not use the actual roll or pitch inputs. Those enter the separate attitude
controller at [lines 143–148](../../fly/src/flight/drone/droneController.ts#L143).
The returned NED force is therefore identical for a level aircraft and one
rolled 90°, if velocity, position, commands and the other inputs are identical.
The XML applies that vector in the world frame.

Consequently, the drone can maintain the same upward support or begin sideways
braking before its propeller axes have physically tilted into the required
direction. Attitude motion can lag without delaying the commanded translational
force. A fixed-axis rotor cannot do that.

**Correction:** make the controller output desired thrust/attitude or motor
commands. Generate actual thrust in each motor's body-fixed axis, transform it
using the current aircraft pose, and give JSBSim the resulting body force and
moments. Keep aerodynamic forces separate. Do not transform rotor force by the
desired future attitude.

**Acceptance:** with identical nonzero motor RPM and no wind, rotate the aircraft
through 0°, 30°, 90° and 180° roll. The world thrust vector must rotate with it.
At 90°, rotor thrust must provide no upward component. An attitude step must
not produce the fully tilted translational force before the body rotates.

### P2 — The controller's tilt limit suppresses aerodynamic drag

**Impact: high, particularly in the 100-knot test. Confidence: direct source proof.**

Drag is added to the acceleration request at
[lines 97–104](../../fly/src/flight/drone/droneController.ts#L97). The horizontal
components of their sum are then reduced to fit a rotor tilt cone at
[lines 105–112](../../fly/src/flight/drone/droneController.ts#L105). The atmosphere's
force is thus constrained by an autopilot actuator limit. At high airspeed,
most of the calculated drag can disappear at this stage.

Drag should be an external aerodynamic load. A flight controller may fail to
overcome it, but its tilt limit cannot reduce that load. The same combined
vector is also treated as rotor thrust by the RPM estimator, so aerodynamic
loading is incorrectly attributed to motors.

**Correction:** calculate aerodynamic force independently from actual air-relative
velocity, density and aircraft pose; add it after the actuator model. Apply
tilt, thrust, motor and power constraints only to the commands the aircraft can
produce. Distinguish rotor force from total external force in the data contract.

**Acceptance:** in a motor-off test at fixed airspeed and pose, changing autopilot
tilt limits must not change drag. Doubling speed under a constant-CdA model
must quadruple drag. The summed rotor and aerodynamic vectors must reproduce
the force sent to JSBSim.

### P3 — The plant can command downward rotor force or unlimited upward force

**Impact: critical, including within ordinary vertical-rate commands. Confidence:
direct source arithmetic.**

The vertical command is `fD = kv * (vDcommand - vDactual) - g`, followed by drag
([lines 81–104](../../fly/src/flight/drone/droneController.ts#L81)). There is no
unidirectional-thrust constraint, maximum collective thrust, motor speed limit
or power limit. The tilt limiter runs only while `-fD > 0`, so a downward vector
also bypasses that limiter.

Two examples use the current parameters, zero wind and otherwise upright
flight, away from ground overrides:

- **Air 2S:** full descent from rest commands 6 m/s down. With `kv=2.5`,
  `fD/m = 15 - 9.80665 = +5.19335 m/s²`, or **3.090 N downward** for 0.595 kg,
  before any aerodynamic force exists. The desired body-Z vector is then up;
  `atan2(yD,bZD)` requests an absolute roll of 180°, despite the configured 35°
  tilt limit.
- **Mini 2:** releasing the vertical stick while climbing at 5 m/s commands
  `fD/m = 12.5 - g = +2.69335 m/s²` downward before drag. The simplified drag
  adds another 1.22992 m/s² downward. Downward drag can be physical; the positive
  controller-generated rotor force is the impossible component for an upright
  fixed-pitch vehicle.

A large descent speed produces the opposite problem: arbitrarily large upward
force can be requested without a corresponding motor/power capacity.

**Correction:** constrain demanded acceleration by feasible collective thrust
and orientation, use the actual motor plant's achieved thrust, and let gravity
and aerodynamic drag brake a climb when motors cannot provide reverse thrust.
Construct a valid attitude target only from a feasible rotor-thrust direction.
Add acceleration and jerk limits to the vertical-rate controller where justified
by the chosen flight-controller model.

**Acceptance:** every motor remains within its modeled nonnegative speed/thrust
range. A descent command from hover cannot create downward thrust in an upright
airframe or an inverted attitude target. Large climb/descent disturbances must
produce bounded actuator effort and the resulting physical recovery trajectory.

### P4 — Display saturation does not constrain the applied wrench

**Impact: high. Confidence: direct source proof.**

The controller computes moments directly from inertia and body-rate error
([lines 144–148](../../fly/src/flight/drone/droneController.ts#L144)). It does not
limit those moments by the forces available from four motors. The later
[RPM estimator, lines 32–47](../../fly/src/flight/drone/droneMotorTelemetry.ts#L32)
rescales all differential motor terms when a motor would need negative thrust.
That changes the forces implied by the four displayed RPMs, but does not change
the aggregate moment already delivered to JSBSim.

Thus four gauges can show a feasible-looking allocation while the rigid body
receives a different, infeasible torque. Showing more RPM digits would not fix
this inconsistency.

**Correction:** actuator allocation must precede force application. Apply the
wrench reconstructed from achieved motor thrust and reaction torque. Feed
achieved-versus-requested effort back to any integrators or command limiters.

**Acceptance:** under saturated roll, pitch, yaw and combined commands, sum
`r × F` and rotor reaction torque from actual motor states. This sum must equal
the moment sent to JSBSim. Reducing motor capability must reduce achievable
angular acceleration rather than only change the gauges.

### P5 — Ground-proximity branches replace forces discontinuously

**Impact: medium/high near contact. Confidence: direct source proof.**

[Lines 89 and 114–119](../../fly/src/flight/drone/droneController.ts#L89) cap a
desired downward rate using AGL/dt and apply separate low-AGL force overrides.
Below 0.15 m, a descent input sets horizontal forces to zero and vertical force
to exactly minus weight, regardless of airspeed and motor state. These are
behavioral protections, not a contact or rotor model. They also remove drag
alongside commanded acceleration.

The XML has a single structural belly spring/contact, not separate landing-leg
contacts ([Mini 2 XML, lines 27–37](../../DJI-Mini2/public/jsbsim-data/aircraft/dji-mini-2/dji-mini-2.xml#L27)).

**Correction:** keep contact reaction, landing logic, aerodynamic loads and
motor effort distinct. A landing controller can reduce descent demand or
disarm after contact, but should not erase physical air loads. Any protection
retained for simulation stability must be explicit and tested as a protection.

**Acceptance:** compare otherwise identical trajectories just above and below
each threshold. Aerodynamic force must remain continuous. Contact forces must
arise from penetration/relative contact motion and contact geometry, not from
the sign of the pilot's vertical stick.

## Controller approximations and checked signs

### C1 — Desired-attitude geometry is consistent in its normal domain

For an upward rotor-force request, the construction `bZ = -normalize(F)`,
`bY = normalize(bZ × desiredHeading)`, `bX = bY × bZ`, followed by
`theta = -asin(bX.down)` and `phi = atan2(bY.down,bZ.down)`, is consistent with
the body's forward/right/down convention. See
[lines 120–145](../../fly/src/flight/drone/droneController.ts#L120).

This audit found no justification for restoring the old position-hold signs or
flipping the pitch/roll signs. The major error is applying the desired force
before the actual attitude realizes it. The downward-force case in P3 lies
outside the intended upright attitude domain and needs a feasibility constraint.

**Acceptance:** independent coordinate-transform tests should cover north/east
acceleration at several headings, positive and negative roll/pitch, and full
rotation matrices. Position error must continue to request velocity back toward
the anchor, preserving the corrected signs at lines 85–88.

### C2 — Euler errors and yaw heading rate are used as body-rate commands

**Category: controller approximation.**

`pCmd`/`qCmd` are proportional Euler-angle errors, while `rCmd` is a desired
heading rate, then all three are compared directly with body angular rates
([lines 128–148](../../fly/src/flight/drone/droneController.ts#L128)). Euler angle
rates and body rates are equal only in special orientations. At significant
roll/pitch, a heading-rate command generally requires coupled p/q/r components.
The roll/pitch rate ceilings also reuse the yaw-slew parameter.

**Correction:** specify whether each command is an Euler rate or body rate and
perform the correct transformation, or use a quaternion/geometric attitude
controller. Give rate and torque limits explicit physical/control meanings.

**Acceptance:** command the same heading rate at level and at combined roll/pitch.
Verify the measured heading rate and bounded attitude error, including wrap
boundaries. Separate yaw-rate specifications from roll/pitch capability.

### C3 — Horizontal stick speed can exceed the declared scalar setting

**Category: command-envelope inconsistency.**

[Lines 79–80](../../fly/src/flight/drone/droneController.ts#L79) rotate independent
pitch and roll components each multiplied by the speed setting. Full diagonal
input therefore requests `sqrt(2) * speed`: 14.14 m/s from a nominal 10 m/s Mini
2 mode, or 22.63 m/s from a nominal 16 m/s sport mode. Drag and actuator limits
may prevent achieving that command; there is no scalar horizontal-speed clamp
here. Similarly, the hold velocity clamps its north and east components
separately.

**Correction:** explicitly declare component limits or normalize the 2D demand
to a circular horizontal-speed envelope, according to the intended controller.
Do not label a component bound as an achieved airspeed limit.

**Acceptance:** sweep stick angle with constant stick magnitude and verify the
desired horizontal speed envelope at every heading. Confirm behavior with
simultaneous hold errors in both north and east.

## Missing physical models

### M1 — No motor/ESC shaft state exists

There are no integrated motor angular velocities, shaft inertias, motor torque
curves, ESC limits, spin-up/down response, friction, arming/disarming dynamics
or actuator faults. The XML has no propulsion subsystem; the force controller
does not read or advance four motor states. The declared `running` property in
[Quadcopter's manifest, line 47](../../Quadcopter/public/0sfs-mod.json#L47) does
not supply such a plant.

The current estimator assumes thrust proportional to RPM² and normalizes it
with the visual hover-RPM reference. Its yaw reaction lever is `arm/4`, an
assumption rather than measured propeller torque data
([droneMotorTelemetry.ts, lines 17–47](../../fly/src/flight/drone/droneMotorTelemetry.ts#L17)).
Both airframe definitions currently set `rotorHoverRpm` to 6000
([Mini 2, line 168](../../DJI-Mini2/public/0sfs-mod.json#L168),
[Air 2S, line 168](../../DJI-Air-2S/public/0sfs-mod.json#L168)); this audit found
no calibration evidence in that data establishing those as real hover speeds.

**Correction:** introduce four actual actuator states before the JSBSim force
sum. A model may live in JSBSim or a host subsystem, provided one authoritative
state supplies forces, gauges and visible rotation. Expose achieved RPM directly.

**Acceptance:** a step command produces a bounded, time-resolved RPM response;
shaft acceleration follows available torque minus load. Disarming removes
motor drive, and any windmilling follows the selected propeller model. The RPM
reported to the widget is exactly the same state used to calculate thrust.

### M2 — No propeller aerodynamic operating map exists

A static `thrust ∝ RPM²` reference cannot resolve thrust/torque versus axial
inflow, crossflow, advance ratio, density or operation far outside hover. This
is a major limitation for the 100-knot test, even after P1–P4 are fixed. Induced
flow, ground effect and descent recirculation are also absent; this audit does
not assign accurate coefficients or claim a specific real-drone failure mode.

**Correction:** supply a documented propeller thrust/power model with its valid
operating envelope, then add higher-order effects where data and intended use
justify them. JSBSim already offers an
[FGPropeller model](https://jsbsim-team.github.io/jsbsim/classJSBSim_1_1FGPropeller.html)
with thrust and power coefficient tables. Its availability does not mean those
tables have been supplied for these drones or establish their real-aircraft
accuracy.

**Acceptance:** validate static thrust and power against data, then vary RPM,
density and inflow, including windmilling/braking where the model supports it.
Report unsupported operating regions rather than silently extrapolating a
hover-only law as calibrated high-speed behavior.

### M3 — No electrical/thermal energy budget constrains propulsion

Battery state, voltage sag, current limits, motor/ESC efficiency and temperature
do not constrain the current force generator. Infinite or instantaneous energy
is effectively available to the aggregate controller.

**Correction:** select and document an energy/power model and connect its limits
to actual motor torque. JSBSim's standard
[FGElectric reference](https://jsbsim-team.github.io/jsbsim/classJSBSim_1_1FGElectric.html)
also documents simplifications, including no battery model; merely selecting
an electric-engine class would not supply those missing effects.

**Acceptance:** delivered shaft work must agree with the modeled electrical
input and losses. Lower voltage/current or thermal limits must reduce achieved
RPM/thrust; depletion must have a defined result.

### M4 — Ideal state feedback substitutes for an aircraft estimator/controller

The controller reads native position, velocity, attitude and angular rate on
each physics step, with no sensor noise, latency, rate differences, estimator
biases or controller scheduling beyond the 120 Hz loop
([lines 203–224](../../fly/src/flight/drone/droneController.ts#L203)). Gains are
shared constants, not identified DJI firmware behavior. The result may be a
useful ideal autopilot but is not evidence that a real Mini 2 responds the same.

**Correction:** identify the target fidelity. Preserve a clean ideal-controller
mode if useful, but add sensor/estimator timing and controller behavior only
with explicit assumptions and calibration. Do not use those additions to hide
force-law errors.

**Acceptance:** validate command-response traces first with ideal feedback, then
test specified delay/noise and control update rates with a bounded stability
envelope. Separate controller errors from actuator and airframe errors.

## Data and parameter calibration

### D1 — Mass/inertia are internally consistent approximations, not measured tensors

The configured inertias reproduce the contract's four equal point masses at a
common X-layout radius: `Ixx=Iyy=m*r²/2`, `Izz=m*r²`. The XML's slug·ft² values
agree with these kg·m² figures within rounding. This is a consistency check,
not proof of the real mass distribution.

| Parameter | Mini 2 | Air 2S |
| --- | ---: | ---: |
| Configured mass | 0.249 kg | 0.595 kg |
| Motor radius used in approximation | 0.1065 m | 0.151 m |
| `m*r²/2` | 0.001412110125 kg·m² | 0.0067832975 kg·m² |
| Configured Ixx and Iyy | 0.00141211 kg·m² | 0.00678330 kg·m² |
| `m*r²` | 0.00282422025 kg·m² | 0.013566595 kg·m² |
| Configured Izz | 0.00282422 kg·m² | 0.01356659 kg·m² |

Sources: [Mini manifest, lines 23–38](../../DJI-Mini2/public/0sfs-mod.json#L23),
[Air manifest, lines 23–38](../../DJI-Air-2S/public/0sfs-mod.json#L23),
[profile assembly, lines 75–80](../../fly/src/flight/drone/droneTypes.ts#L75),
and the contract's explicit inertia approximation (then in [SPEC-360-DRONE section 4.1](SPEC-360-DRONE.md#41-sensor-and-vehicle-constants); now replaced by the component estimate).
The airframe XML and manifest independently carry related mass/inertia values;
the controller uses the manifest mass and diagonal inertia rather than reading
the active JSBSim mass properties. Runtime payload/configuration changes could
therefore make these sources diverge.

**Correction:** obtain or identify a defensible mass distribution/tensor, retain
its provenance and uncertainty, and ensure controller/plant use the same current
mass properties. A mesh's visual detail does not establish its inertia.

**Acceptance:** automate XML/manifest unit consistency; validate current mass/CG
under supported payload configurations. Apply known torques and compare angular
acceleration against the configured tensor, including coupled-axis cases.

### D2 — Drag, density, wind and gains are not a calibrated atmosphere/airframe model

The shared manifest supplies fixed air density 1.225 kg/m³ and shared gains
([Quadcopter, lines 22–35](../../Quadcopter/public/0sfs-mod.json#L22)); CdA is a
single scalar per drone. The controller's disturbance wind is a private seeded
two-component Ornstein–Uhlenbeck process
(`droneWind.ts`, lines 22–40, since deleted),
initialized independently by the feature and passed to the force calculation.
This path reads neither native atmospheric density nor native wind properties.
The zero-aerodynamics XML does not supply the ordinary airframe aero response
to native weather. A full weather audit should also check other host paths
before concluding how every weather control affects the drone.

The constant scalar CdA omits attitude-dependent projected area, aerodynamic
moments and calibrated body drag. Ground contact friction, spring and damping
are likewise fixed modeling choices without validation established here.

**Correction:** define one authoritative ambient-air input, add optional gusts
to it, use local density, and calibrate body forces/moments separately from the
rotor plant. Document provenance, uncertainty and the valid speed/attitude range
for each coefficient. Keep contact calibration separate.

**Acceptance:** changing host mean wind must change relative airflow and the
expected force, while disabling gusts leaves that mean wind intact. Density
changes must affect the selected rotor/aerodynamic model. Compare coast-down,
tilted-body drag, angular damping and landing response with chosen references.

## The 100-knot arithmetic example

This is a one-step calculation of the existing equations, **not a replay of the
user's flight**, and not a prediction of a real Mini 2 trajectory. It assumes
level flight at 100 knots, zero vertical velocity, zero wind, neutral sticks,
hold initialized at the current location and no near-ground branch.

Using Mini 2's current `m=0.249`, `CdA=0.020`, `rho=1.225`, `kv=2.5` and 40°
tilt limit:

| Calculation | Value |
| --- | ---: |
| Ground/air speed | 51.44444 m/s |
| Velocity-feedback horizontal acceleration | −128.61111 m/s² |
| Quadratic-drag horizontal acceleration | −130.20082 m/s² |
| Combined horizontal request before clipping | −258.81193 m/s² |
| Vertical specific force before gravity integration | −9.80665 m/s² |
| Horizontal specific force after tilt clipping | −8.22876 m/s² |
| Horizontal force delivered | −2.04896 N |
| Upward force delivered | 2.44186 N |
| Delivered force magnitude | 3.18762 N |
| Simplified collective RPM reference for that magnitude | 6855.27 RPM |

The pre-clamp drag alone is 32.4200 N, yet the final horizontal force is only
2.0490 N because the controller clips the combined vector. The final world
braking force also acts immediately, independently of the actual tilt. These
two mechanisms are sufficient to explain why the stress response can violate
the expected physical coupling. They do not establish the exact cause of every
feature of the user's observed maneuver.

## What would justify exact per-prop RPM

A physically coherent implementation needs this causal direction:

```text
pilot / autopilot demand
  → feasible motor commands
  → motor torque and rotor angular-velocity states
  → propeller thrust and reaction torque at actual inflow
  → body force + r×F + rotor reaction moments
  → separate airframe aerodynamic/contact loads
  → JSBSim rigid-body integration
```

The widget can then show the exact **simulated** RPM state for each propeller.
That is distinct from claiming those RPM values accurately reproduce the real
hardware. Hardware fidelity additionally needs calibrated motor/propeller data
and flight-response validation. At present the gauge is an honest estimate of
an ideal allocation, and hiding its label would misrepresent the model.

The first acceptance gate should be physical invariants P1–P4. The second should
validate actuator and aerodynamic operating curves. The third should compare
airframe/controller flight responses against documented data. Existing hover,
wind-tilt, mixer, UI, pause and import-boundary tests do not substitute for these
gates; the current controller tests at
[droneController.test.ts, lines 24–44](../../fly/src/flight/drone/droneController.test.ts#L24)
cover only two limited force-plan cases.

No implementation changes, new server, commit or publication were made for this
appendix. No renderer or ground-contact code was edited.
