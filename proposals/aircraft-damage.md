# Aircraft damage: useful implementations at different costs

Status, 2026-10-25: Off, whole-aircraft health and host-authorized shared
selection are implemented in the local checkout. Component and physical damage
below are proposals. No runtime cost
frontier or aircraft damage fidelity has been measured yet.

Follow [Choosing the useful peaks of realism and cost](../realism-cost.md).
The goal is to retain a separate implementation for every useful measured
tradeoff. Health remains a useful cheap implementation when richer models
arrive. Candidates below are hypotheses to compare, not predetermined quality
levels that must all ship.

## Candidate models

| Model | State and behavior | Work and useful hypothesis | Status |
| --- | --- | --- | --- |
| Off | Damage has no effect; no health meter or damage evolution. | Explicit no-damage baseline, useful for ordinary flight and comparisons. | Implemented |
| Health | One 100-point pool for the whole aircraft; zero means destroyed. Partial damage does not alter handling. | Constant work per accepted hit, one scalar of damage state, no continuous damage integration. Useful for casual combat and validating the combat loop. | Implemented; uncalibrated gameplay model |
| Components | Separate wings, tail, controls, engine and tanks; discrete intact/degraded/failed states and authored flight effects. | Simple hit volumes and event-driven transitions. Test whether location-dependent failures earn their additional authoring and runtime costs. | Proposed |
| Physical components | Spatial impact traversal, energy deposition, reduced structural loads, leaks, thermal/fire state and asymmetric aerodynamic effects. | More state and continuous work for active processes. Test whether causal failures, delayed separation and system interactions improve outcomes. | Proposed; [implementation design](aircraft-damage-physical.md) |
| Reference | Offline refined impact/structural/aerodynamic cases, or a more detailed implementation for selected mechanisms. | Calibration and comparison evidence; runtime use must earn its place separately. | Proposed tooling, no product option |

Collision representation, physical model, solver detail and TS/WASM backend are
independent axes. Reuse a cheap exact collision path wherever it applies. More
triangles, fragments or timesteps do not establish more credible damage.

## The implemented health model

The owner is `fly/src/flight/damage/`. `createAircraftHealth` implements the
generic `AircraftDamageModel<Impact, State>` lifecycle: `applyImpact`, `reset`
and a copied `snapshot`. Health impacts contain positive gameplay `points`.
They are not energy, penetration depth or material strength. A physical model
can implement this lifecycle with its own impact/state types; it must not be
forced to derive its physics from health points.

- Every aircraft starts with 100 health. Valid hits subtract points and clamp
  at zero. Non-finite, zero or negative damage is ignored; hits cannot heal.
- There is no passive healing and no per-frame health simulation.
- At zero, the aircraft is marked destroyed. The flight application pauses and
  blocks resume through the shared keyboard, controller, phone and panel pause
  boundary. It does not mutate JSBSim into an arbitrary engine/control failure.
- **+ → Damage** shows one meter for the entire local aircraft. **Test damage →
  Apply 25 damage** exercises the damage path. **Repair aircraft** restores 100
  health without automatically resuming.
- Closing the tab, pausing or relocating the aircraft preserves health. A new
  flight, page reload or successful aircraft change starts a fresh health pool.
  Health is not persisted with a saved flight. Failed aircraft switching
  restores the prior aircraft's health.
- **+ → Shared settings → Damage → Implementation** selects Off or Health.
  An explicit implementation change starts a fresh damage life and pauses
  flight. Switching aircraft preserves the implementation choice; reloading
  starts with the default Health model. Off ignores impacts and shows a status
  instead of a health bar.
- Aircraft poses optionally carry `health`, bounded to 0–100. The Multiplayer
  tab shows the other aircraft's health when reported. Missing health means
  unknown, not intact. Damage and repair while held send a new reliable pose
  before `held`; moving aircraft retain the existing 30 Hz pose cadence.
- These poses report owner state. They are not hit claims or authoritative
  adjudication. Weapon/terrain hit detection, scoring, ammunition, respawn rules
  and reliable accepted-impact events remain separate work. Test damage affects
  the local aircraft only; repair is unrestricted in this casual prototype.

The health pool is aircraft state, independent of tab mount, render rate and
multiplayer connection lifecycle. Its snapshot is available through the flight
feedback provider and the dedicated `damage` provider.

## Shared combat boundary for subsequent models

Add the hit pipeline before connecting guns to any damage implementation:

1. Integrate a projectile in simulation time. Sweep each traveled segment
   against time-matched target geometry, including target motion. A drawn tracer
   and an extrapolated render pose are presentation, not collision authority.
2. Produce an accepted impact envelope: schema/model version, authority,
   shooter/target IDs, target life ID, stable projectile/impact ID, simulation
   tick/time and the model-specific impact payload.
3. Deduplicate at the acceptance boundary and queue impacts for one fixed-step
   application. Both a local prediction and a later authority result must not
   subtract health twice. Enforce bounds before doing expensive work.
4. Apply the selected damage model, publish a compact status for presentation,
   and pass only physical effects to the JSBSim adapter when the model has them.
5. Replicate accepted events reliably and publish versioned state checkpoints.
   Checkpoints include life/model IDs and the last included event. Recover
   packet gaps and reconnects without replaying old hits into a repaired life.

For Health, ammunition definitions map accepted hits to gameplay points.
For Physical components, they supply the physical impact data described in its
design. The common lifecycle does not make these different payloads equivalent.

The existing [multiplayer proposal](multiplayer.md#rounds)
assigns serverless hit decisions to the shooter. Preserve that casual authority
rule when adding hits; server/hybrid modes use server decisions. A host owning
the invitation does not already make it an authoritative flight server. The
current health reporting adds no anti-cheat or hit-authority guarantee.

## Shared selection and permissions

The selection lives in **Shared settings**; the health display and test actions
live in **Damage**. Only the implemented Off and Health choices are exposed.
Their versioned wire identities are `off-v1` and `health-v1`. A change in model
equations/data semantics needs an explicit version change; an implementation
name alone does not establish compatibility. [Protocol and authority details](https://github.com/0SFS/multiplayer/blob/main/docs/implementation.md#shared-settings-and-permissions).

The session host owns the canonical setting and a monotonic revision. By
default, **Settings editors: Host** allows only the host to change it. The host
can grant **Everyone**, allowing the current guest to request changes. Only
the host can change permissions, even under Everyone. This is enforced on the
receiving host, independently of disabled UI controls. New sessions reset the
editing policy to Host. The two-pilot implementation has no per-pilot grant UI;
a later mesh can add explicit member grants without transferring publisher
authority to guests.

The protocol advertises supported model versions during authentication. Before
the session becomes connected, the guest installs the host's settings and
acknowledges their revision. The host confirms readiness. Missing capability
support or a missing acknowledgement prevents a synchronized session; it never
silently substitutes a cheaper model for one peer. v1 clients must update to
the shared-settings v2 protocol.

Changes follow the same barrier. Each client applies the chosen model before
acknowledging it. Flight pauses; resume and aircraft publication are blocked
while synchronization is pending. A model change resets both aircraft's damage
state, whereas a permission change preserves health. Flights stay paused after
confirmation until pilots resume. Every transmitted pose carries model ID and
settings revision, and older or mismatched revision poses are ignored. Settings
snapshots and requests use the authenticated reliable channel, not pose traffic.

Guest requests include their base revision. The host rechecks current
permissions, revision and every peer's capabilities at receipt, then publishes
the accepted state or rejects the request. A stale in-flight request cannot
bypass a later permission revocation. The UI shows accepted values and pending
or rejected state; it does not optimistically run a requested model. Only the
host publishes settings, and clients reject a guest impersonating that role.

Before hosting/joining, model selection is local. Changing it does not import
PeerJS or open a connection. Disconnect retains the last model for local flight
and restores the Host editing default. Permission to change settings is
separate from hit authority, test damage, repair policy and anti-cheat. The
casual peer architecture cannot prove that a modified client executes the
equations it claims to support.

When another model earns a runtime place, keep the same selection/authority
boundary and common presentation. Choose it at a new flight/combat life
boundary, or via an explicit synchronized reset as implemented here. A physical
model must restore its own complete state; Health's reset is not a live state
conversion.

Do not switch a stateful damage model because a frame was slow. There is no
credible mapping from 40 health to a unique punctured tank, jammed surface or
broken spar. An explicit reset can change models; a live migration needs a
separately specified and validated transfer. A model can bound internal detail
without changing its identity, with its approximation reported.

## Establishing the useful peaks

Use shared invariants and independent behavior evidence. Do not require the
physical model to reproduce Health's handling or make Health pretend it
predicts material failure.

| Comparison | What to record |
| --- | --- |
| Flight outcomes | Left versus right damage, control availability, response to load, leak duration, thermal progression, time to loss of control, survivable landing where supported |
| Numerical behavior | Finite state, units/signs, accepted-step time, timestep convergence, impact ordering, no repeated impulses, no render-rate dependence |
| Collision | Missed/false hits, location and time error, swept moving-target coverage, representation preparation cost |
| Runtime cost | CPU per impact and per accepted step, p50/p95/p99 and catch-up tails, allocations/GC, resident and peak bytes, serialized bytes and latency |
| Confidence | Input provenance, calibrated range, independent validation, unsupported configurations and uncertainty |

Start with one local aircraft/two peers, then four aircraft; isolated hits and
bursts of 10/100 accepted impacts; intact, leaking and burning cases. Declare
component/fragment counts, hardware, build, background activity and simulated
duration. These are proposed workloads, not performance results or permission
to run long benchmarks on the laptop. Measure model plus collision, adapters
and replication, as well as each piece in isolation.

Keep 60 Hz as the normal flight target. Compare the existing 30 Hz candidate
and explicitly opted-in 120 Hz separately; internal damage substeps must account
for their work and never silently raise the entire aircraft's physics rate.

Retain each model whose capability/cost tradeoff serves a measured workload.
Record dominated/rejected candidates too. A detailed reference can stay offline.

## Local verification

From `~/gh/0sfs/fly`:

```sh
npm test -- src/flight/damage src/flight/multiplayer/SharedSettingsPanel.test.tsx ../multiplayer/tests/protocol.test.ts ../multiplayer/tests/serverlessSession.test.ts
npm test -- src/flight/createFlightSimApp.test.ts -t 'one aircraft health pool|local damage implementation'
npm run typecheck
npm run dev
```

In **+ → Damage**, apply four test hits, confirm 100 → 75 → 50 → 25 → 0,
confirm Resume is blocked, repair, then resume. Closing/reopening the tab must
preserve damage. With a second connected pilot, repeat while paused and confirm
the other pilot sees damage and repair. An older peer that sends no health
must show no invented health meter. In Shared settings, verify guests cannot
edit until the host grants Everyone, and cannot change permissions themselves.
Change Off/Health from either allowed pilot and check that both reset, pause and
agree before Resume is allowed. Revoke access and retry a guest edit. These checks establish software behavior,
not physical aircraft damage validation.
