# Drone realism audit: sensing, camera, gimbal and presentation

> **Partly history.** The flight-model findings here describe the drone before the 28 October 2026 rebuild. The force controller, the `fcs/drone-*` properties, the estimated RPM and the private wind have since been replaced by the rotor plant and flight controller in [`multirotor/`](../../multirotor/README.md). The [flight-model roadmap](360-drone-flight-model-roadmap-2026-10-28.md)'s stage statuses and placeholder inventory say which findings are fixed and which are still open. Findings outside the flight model are unaffected.

28 October 2026. Source audit of the current local implementation; no simulation code was changed. This appendix supports the consolidated drone realism audit. It identifies confirmed defects separately from missing physical models, uncalibrated data and intentional presentation choices. It is not a claim that every possible real-world effect has been enumerated or that DJI firmware has been reproduced.

The immediate high-speed flight problem is primarily in the vehicle force model, covered by the propulsion audit. Camera and sensing shortcuts can conceal that problem, make the vehicle look unnaturally stable, or produce misleading capture telemetry. They must not be mistaken for a physically complete navigation stack.

## Findings in repair order

| ID | Finding | Classification | Priority | Confidence |
| --- | --- | --- | --- | --- |
| S01 | Lucas–Kanade shader builds a rank-one gradient matrix | Confirmed algorithm defect | P1 | High: algebraic proof |
| S02 | Visual displacement is integrated repeatedly at physics rate; timestamps use the wrong interval | Confirmed timing defects | P1 | High: source ordering |
| S03 | Capture image pose and reported shutter pose do not share an atomic snapshot | Confirmed ordering gap; visible error needs a moving-scene test | P1 | High for ordering, unmeasured image error |
| S04 | “Settled” omits translation and actual exposure effects | Capture decision defect plus missing imaging model | P1 | High |
| S05 | Flight feedback and fallback “IMU” are exact simulator truth | Model omission | P2 after plant correctness | High |
| S06 | Gimbal can stabilize beyond physical joint stops and without stabilization actuator limits | Model omission/constraint defect | P2 | High |
| S07 | Camera mounting and lens calibration are nominal rather than calibrated per vehicle | Uncalibrated data/model omission | P2 | High for representation, unknown real error |
| S08 | Phone UI has independent camera/airframe state and incomplete model-driven capabilities | UI/state fidelity gap | P2 | High for source; end-to-end stale-state scenario needs a test |
| S09 | Phone timing/loss behavior is generic simulator policy, not a modeled DJI radio/autopilot | Deliberate design/model omission | P2–P3 | High |
| S10 | Propeller visuals and acoustic feedback do not reproduce physical propulsion | Presentation approximation/omission | P3, following motor model | High |

P1 means a concrete correctness issue that should be fixed before using capture/sensing results as evidence of realism. P2 means materially incomplete behavior that should follow the physical plant. P3 is perceptual fidelity that must not delay correcting force generation. These priorities are audit recommendations, not existing project requirements.

## S01 — the optical-flow shader does not form a valid two-dimensional LK system

Evidence: `drone-camera/src/flowWgsl.ts:69` computes `ix` and `iy` once at the center pixel. The 7×7 loop at lines 76–87 reuses those exact two values for every window sample. With 49 samples, the structure tensor is

```text
gxx = 49 ix²;  gxy = 49 ix iy;  gyy = 49 iy²
det = gxx gyy − gxy² = 0
```

Its minimum eigenvalue is also zero in exact arithmetic. A corner or textured window cannot acquire two-dimensional confidence because gradients from neighboring pixels are never accumulated. The solver at lines 90–100 therefore has no valid full-rank system; floating-point roundoff cannot substitute for image structure. `flowEstimator.ts:31` rejects confidence at or below `1e-4`, so the implementation can fall back to the path labeled “imu” even when the scene is well textured. A returned `source: "visual"` from `createOpticalFlow` itself is only a capability label; it is not proof that a valid visual estimate was computed.

Impact: capture alignment/settling does not have the visual feedback its status suggests. The drone flight controller does not use this optical flow for navigation, so fixing it alone will not fix vehicle dynamics.

Remedy: calculate spatial gradients for each window sample, form the true tensor, use a consistent LK sign/warp convention, and retain a conditioning threshold. Acceptance: on a known textured synthetic image, horizontal, vertical and diagonal subpixel translations produce the correct flow and positive confidence; an edge alone is rejected; a flat image falls back cleanly. Exercise the actual WGSL on WebGPU, not only a hand-constructed list of `FlowSample` values.

## S02 — flow timing and sequence accumulation are not sample based

Evidence:

- `fly/src/flight/drone/droneFeature.ts:285` derives `dt` from `performance.now()` and the latest rendered frame. That is wall time, while the camera pose comes from simulated motion.
- `drone-camera/src/opticalFlow.ts:77` drops pushes while GPU readback is busy. The next accepted image is compared with the previous accepted image, but it is still given the last display-frame interval rather than the elapsed interval between those accepted images.
- `opticalFlow.ts:147` stores one estimate and `latest()` at line 154 returns it without a sample ID, acquisition timestamp or age.
- `droneFeature.ts:188` reads that same estimate on each 120 Hz physics step. `drone-camera/src/captureSequence.ts:137` adds `deltaYawDeg`/`deltaPitchDeg` every such step, rather than once per newly accepted image pair.

Example: one displacement measured at 30 camera frames/s can be added roughly four times by a 120 Hz loop. A delayed/stale image can be added for still more steps. The resulting fine alignment and angular-rate estimate depend on rendering cadence, GPU throughput and simulation speed rather than only vehicle motion. `lastFrame` also remains set across capture inactivity, so the first resumed interval is not a normal frame pair.

Remedy: carry sequence number, simulation acquisition times, camera poses and actual accepted-pair interval with every estimate; consume each displacement once; reject old results and reset image history on pause, scene/profile changes and capture restart. Acceptance: identical simulated motion with 15/30/60 fps, deliberately delayed readback and multiple physics substeps yields the same accumulated angle and approximately the same rate. Repeated `latest()` reads must not generate additional displacement.

## S03 — shutter pose, camera pose and image acquisition are not atomic

Evidence: sequence capture is invoked from `beforeStep` (`droneFeature.ts:194`, callback at lines 204–208), using `result.gimbalDeg` and a fresh native `readFlightState`. The actual sensor camera and visual aircraft root are posed later in `afterView` (lines 270–279). `captureFrame.ts:41` renders immediately when `capture()` is called; it does not first apply the supplied metadata pose or obtain an immutable scene snapshot. The position/orientation used to draw can therefore be the previous view pose while EXIF/XMP and sequence metadata describe a newer physics state. Several physics substeps can occur between views. Ordinary still capture is initiated from `afterView` and has a smaller version of this risk, but still needs an explicit acquisition contract.

`captureSequence.ts:172` additionally records the nominal grid node's pitch/yaw in the manifest, even when `slewTimeout` or fine alignment makes actual capture orientation differ. Actual orientation is placed separately in JPEG metadata; consumers must not treat the manifest's planned node as the measured camera pose.

Impact: inconsistent sensor evidence, exaggerated parallax or stitching error, especially during motion or a timed-out sequence. The exact pixel/angle error has not been measured in this audit.

Remedy: define one simulation acquisition timestamp and freeze position, body attitude, joint angles, exposure parameters and relevant scene transforms for that shot; construct/render the sensor from that snapshot and derive metadata from the same snapshot. Keep planned and actual node orientation as distinct manifest fields. Acceptance: a moving/rotating target marker has the image location predicted by the recorded camera matrix at multiple physics/render-rate ratios, including a capture initiated between rendered frames.

## S04 — “settling-zero-velocity” is an angular gate, not a physical sharpness test

Evidence: `captureSequence.ts:158` checks only `omegaGate < omegaBlur` for six physics steps, or captures after a two-second timeout. No linear velocity is provided to the sequence interface. The feature supplies a manual roll/yaw mount rate and either a visual rate or the norm of exact body angular rates (`droneFeature.ts:180`, 188–203). The camera's own pitching slew rate is not part of `mountRate`. Body rate is not necessarily the stabilized sensor rate.

`blurLimitRadPerSec()` (`flowEstimator.ts:45`) uses a half-pixel rotational-smear approximation from fixed exposure. Translation produces depth-dependent image movement even at zero body angular velocity. Fallback “imu” cannot detect that translation. `captureFrame.ts:41` then takes one instantaneous render; it does not integrate image irradiance over the metadata exposure time. A photo flagged `blurRisk` can remain perfectly crisp.

Impact: “settled” can be reported while traveling, especially with texture-poor imagery/fallback flow; gimbal/body compensation can also make the gate over-conservative. The result cannot validate real sharpness at high speed.

Remedy: rename the current status to describe the actual gate until fixed. Evaluate sensor-camera angular velocity plus translation/scene-depth flow over a real exposure interval. Choose whether blur is to be physically rendered or only predicted, and make that contract explicit. Acceptance: an unrotating camera translating past near geometry does not pass a sharpness gate solely because body gyro rate is zero; manual pitch slew contributes to the rate; stabilized body rotation is evaluated at the sensor; timeout/risk flags agree with expected pixels of smear.

## S05 — there is no navigation sensor or state-estimation model

Evidence: `droneController.ts:203`–224 reads JSBSim position, attitude, angular rates and north/east/down velocities directly every 1/120 s. No IMU noise/bias, sample clock, GPS/barometer/range noise or measurement delay separates the controller from those state variables. The capture fallback named “imu” likewise reads exact JSBSim p/q/r (`droneFeature.ts:189`–203), not simulated inertial measurements. Optical flow is requested only for capture (`droneFeature.ts:283`), not used by the flight controller to estimate ground motion or hold position.

Impact: navigation has perfect state knowledge and cannot exhibit sensor drift, poor GNSS/visual positioning, estimator lag or loss-of-sensor behavior. A deterministic truth-feedback mode remains useful for testing a physical motor/airframe model, but should be labeled as ideal navigation rather than a reconstructed DJI sensor stack.

Remedy: first keep a deliberate “ideal state” mode to isolate plant bugs; then introduce a shared timestamped sensor interface and estimator as a separate layer, with controlled noise/bias/latency/dropout and reproducible seeds. Do not tune a noise model to conceal an incorrect force model. Acceptance: controller reads the configured estimator output rather than plant truth; delayed/biased/dropout scenarios have defined hold/fallback behavior, while zero-error mode remains reproducible.

## S06 — gimbal stabilization is ideal and can exceed joint capabilities

Evidence: `camera-gimbal/src/index.ts:34` cancels exact body roll/pitch algebraically. The articulated solver at lines 70–83 decomposes the desired pose into yaw/roll/pitch without receiving mechanical joint limits, rate, acceleration or torque limits. `droneFeature.ts:176` and 178 limit the manual roll/yaw offsets; those limits do not bound the combined mechanical angle needed for body compensation. Manual pitch has a simple slew limit in `droneController.ts:149`. Changes in body attitude can therefore be canceled instantly by the compensation path, independent of that manual slew.

Impact: at large rolls/pitches the camera can stay level when a real bounded gimbal could hit a stop or lag. The display can look calmer than the actual airframe dynamics. The same ideal pose drives the visible HD assembly and sensor, so their agreement does not establish physical feasibility.

Remedy: apply an inverse-kinematic target to an explicit per-airframe joint-state model with actual stops, angular rate/acceleration limits and defined stop behavior. Render/capture from achieved joint angles. Acceptance: body rolls beyond the compensation envelope cause a bounded joint and residual image roll; commanded offsets consume the same mechanical travel as stabilization; sensor and housing remain consistent through stops and return from them.

## S07 — camera geometry and imaging parameters are nominal

Evidence: `fly/src/flight/drone/droneGimbal.ts:6` uses the same `{x:0,y:-0.02,z:0.05}` sensor mount for both airframes. It does not derive the optical center from the HD lens/joint geometry or an airframe mount profile. `drone-camera/src/profile.ts:18` assumes a centered ideal pinhole from diagonal FOV and image aspect. `jpegExif.ts:158`–160 writes this computed focal length and the image midpoint under calibration-named XMP fields. There is no lens distortion, calibrated principal point or per-mode crop model in `CameraProfile`.

The camera manifests supply nominal dimensions/FOV and fixed exposure denominators (Mini 2 1/1000 s; Air 2S 1/800 s: `DJI-FC7303/public/0sfs-mod.json:19` and `DJI-FC3411/public/0sfs-mod.json:19`). The specification derives focal values from assumed sensor dimensions/FOV (`SPEC-360-DRONE.md:426`), rather than a measured camera calibration. The render path contains no auto-exposure, readout timing, rolling shutter, sensor noise or optical point-spread model. A WebGPU image is a rendered scene sample, not a complete camera simulation.

Impact: image proportions and metadata can be internally consistent while differing from a real lens or actual gimbal optical center. Low-altitude parallax and stitching checks are especially sensitive to the mount origin. Fixed exposure cannot be interpreted as a real exposure controller. Photorealistic noise is lower priority than geometric/timing correctness for the current task.

Remedy: add per-airframe mount/optical-center data and provenance; distinguish assumed intrinsics from measured calibration; verify FOV/crop/exposure against the intended camera mode. Add distortion/readout/photometric effects only for identified training or capture requirements. Acceptance: rendered fiducials match a specified camera matrix across the image; optical-center movement agrees with joint geometry; metadata identifies nominal versus calibrated values; known exposure modes have explicit behavior.

## S08 — phone camera state can diverge from the applied aircraft and host controls

Evidence: `rc/src/drone/PhoneQuadcopter.tsx:8` hardcodes the two pitch limits. Lines 21–24 initialize local `droneId`, gimbal, capture and Sport state once; subsequent host telemetry does not synchronize those values in this component. Commands always carry that local ID/target (lines 27–37). The host may request an aircraft switch when the incoming drone ID differs (`createFlightSimApp.ts:3718`). Desktop gimbal changes or Stop can therefore disagree with the phone's displayed local state; a stale local model selection may request a different model on later phone input, subject to session ownership/reconnection behavior.

The phone declares `yawMode: "angularVelocity"` and has no UI for the desktop's new roll/yaw gimbal offsets. That is a capability difference, not by itself a physics error. It should be deliberate and discoverable rather than a second independent definition of camera capabilities.

Remedy: advertise camera/airframe capabilities and actual/target state from the installed profile; distinguish explicit model-change actions from ordinary stick packets; reconcile host state without moving an actively held control under the finger. Acceptance: desktop model/gimbal/Stop changes synchronize on the phone, later stick input does not silently switch aircraft, profile replacement updates bounds, and handoff/reconnect preserves the defined state.

## S09 — remote command timing and failsafes are simulator policy

Evidence: control frames are scheduled at 60 Hz with event sends capped at 120 Hz (`rc/src/protocol.ts:14`, `phoneControllerClient.ts:387`, 759–783), applied on arrival rather than through a DJI link model. The receiver stores the last accepted drone command (`createPhoneControlSession.ts:457`, accessor at 730). It intentionally holds a silent phone's last command for up to five seconds (`protocol.ts:12`, session lines 264–267, 585–605), then returns authority to the computer. This is generic shared-control behavior, not a modeled radio/autopilot link-loss procedure. There is real network latency and jitter, but no calibrated additional DJI processing/transmission delay or virtual sensor/radio dropout model.

Impact: smartphone control feel and loss behavior should not be used to infer real DJI controller/link behavior. Holding a climb/yaw command is materially different from an autopilot loss-of-link policy. The phone stick does already cancel on blur/visibility loss (`PhoneStick.tsx:64`–83); do not incorrectly classify all normal release handling as missing.

Remedy: retain the current simulator control-sharing policy as a separate mode, add an aircraft/autopilot-specific command-age and link-loss layer if realistic remote operations are a goal, and expose measured input-to-application timings. Acceptance: deterministic latency/jitter/drop scenarios give the documented per-aircraft response without stale commands being replayed as fresh pilot intent.

## S10 — propeller display and audio are perceptual approximations

**Presentation follow-up (corrected):** an intermediate change made sharp 3D blades the high-RPM default. That was a misreading and has been reverted. Above the alias limit the shared component now shows a semi-transparent **3D swept propeller volume** by default, measured from the blade meshes, as the C172's authored `Propeller_Disc` does. **Aircraft → Flat propeller discs** (Off by default) is a live opt-in for a cheaper flat disc. Slow propellers still show their turning 3D blades. RPM remains an inverse estimate until the physical motor model is implemented. See the consolidated audit's propeller section.

Evidence (current code): `rotating-prop/src/motion.ts` integrates the supplied estimate for visible phase and flags blades above the display alias threshold. `rotating-prop/src/binding.ts` then substitutes a constant-alpha 3D swept volume (`src/sweep.ts`), or the opt-in flat disc. That is an intentional anti-aliasing presentation technique. It is not a physical exposure, motion-blur or blade-deformation model: opacity does not vary with exposure time, blade solidity or RPM. The HUD shaft circles also deliberately slow the rotation for readability. Neither changes thrust, and both will remain approximate even after RPM becomes a physical state.

No drone acoustic installation is declared in either airframe manifest. The default sound installation registry (`sound/src/aircraftAudioProfiles.ts:26`–32) covers the C172, Vision Jet and F-35B; dynamic aircraft need explicit audio data (`fly/src/flight/mods/applyDynamicMod.ts:65`). The host only attaches an engine audio adapter when an installation exists (`createFlightSimApp.ts:1255`–1263). There is consequently no four-propeller RPM/load-driven acoustic model for these drones.

Impact: the absence of expected sound, actual blade/rotor-phase detail and camera exposure cues can make the vehicle feel strange, but adding audio or realistic-looking prop blur will not repair the force-model defects. Correct opposite-corner rotation and consistent gauge/prop estimates are useful internal consistency checks, not validation of real motor physics.

Remedy: feed visuals and a future shared quad sound source from the actual motor-state model once it exists; retain a clearly defined readable display scale; use exposure-driven blade integration only where required. Acceptance: pause/slow simulation holds phases consistently; four physical motor RPM/load changes drive both visuals and sound coherently; readable HUD scaling is documented separately from physical RPM.

## Validation limits and dependencies

No browser, physical phone, GPU shader or sound-device run was performed for this appendix. S01's rank defect is an algebraic source proof; S02 follows the explicit render/physics clocks and accumulation paths. S03's magnitude, S07's calibration error and S08's complete remote-session scenario need targeted tests. Manufacturer-specific sensor errors, gimbal dynamics and link-loss policies require primary data before numerical fidelity can be claimed.

Repair the force/motor plant first, then S01–S04 and geometric gimbal constraints. Preserve a truth-feedback mode while validating the plant. Add noisy estimation, calibrated sensor/link behavior and acoustic/photometric refinements as separate modules, so realistic-looking outputs cannot conceal an incorrect underlying simulation.
