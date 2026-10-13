# Flight VR TODO

Status: deferred feature, 2026-10-08. **Do not implement VR or provision its
repository during the split.** It is excluded from current tab counts and
migration acceptance criteria.

Proposed owner: **`0sfs/vr`**, consuming
[`foss-earth/vr`](../../../foss-earth/docs/proposals/vr.md). Shared immersive
session, tracking, rendering and generic UI behavior stay in FOSS Earth and
are also used by the UMN tour. Flight owns cockpit interactions, seat/view
configuration, and the adapter between tracked poses and aircraft/floating
origin reference frames.

The simulator composes `0sfs/vr` and injects aircraft pose, control and camera
services through public ports. The VR adapter must not import the simulator
runtime, establish a second flight clock, or apply a second copy of flight
controls. Aircraft-specific geometry and interaction definitions belong to
their aircraft packages and are supplied through their public contracts.

The later assignment must design head/vehicle pose composition, recentering,
cockpit input ownership, entry/exit handoff and accessible non-VR alternatives,
then validate actual headset/backend combinations. These are TODOs, not
support claims. Decide the settings/tab presentation then; no new current
tab is implied by reserving this repository name.

Proposed direction: `simulator → 0sfs/vr → foss-earth/vr → renderer/ui`.
Neither shared VR nor the campus tour imports `0sfs/vr`. The dependency graphs
mark these relations as deferred; they must not become required dependencies
of today's migration artifacts.
