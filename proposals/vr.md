# Flight VR

Status, 2026-10-21: 0SFS flies in a VR headset from its VR tab, through
FOSS Earth's `foss-earth/immersive` ([shared VR](https://github.com/foss-earth/docs/blob/main/docs/proposals/vr.md)).
It is tested on an emulated headset and not yet on a real one. The 2026-10-08
note deferring VR until after the split no longer holds. The adapter is in
`fly/src/flight/createFlightSimApp.ts`, and the `0sfs/vr` repository below is
still to come.

## What there is

- **Seated in the cockpit.** Entering VR puts the headset at the cockpit
  camera's eye, facing the nose, with the space turning with the aircraft. There
  is no snap turn. The view switches to the cockpit for the session and back to
  what it was after.
- **The interface on the cockpit screen.** By default the toolbar, the windows
  and the instruments are on the touchscreen, which a hand points at and
  presses ([cockpit screen](https://github.com/0SFS/fly/blob/main/docs/proposals/cockpit-screen.md)).
  Aircraft → Cockpit screen moves each part to panels around the head instead.
  In an aircraft without a screen, or with its own displays chosen, every part
  is on panels.
- **Tests.** `npm run dev` gives a page opened with `?headsetEmulator=quest3`
  an emulated headset. [Testing as an agent](https://github.com/foss-earth/docs/blob/main/docs/agent-testing.md)
  has the harness and recipes.

## Still to do

- **Flying with the headset's controllers.** A keyboard, a game controller or a
  paired phone flies the aircraft in VR. The controllers' sticks and triggers
  are not yet mapped to the flight's controls. Map them through the same
  control path as a gamepad, not a second copy of the flight controls.
- **Recentering** the seat to the pilot's head, and a seated or standing
  height.
- **The aircraft's own displays and cockpit controls**, as aircraft packages
  define them.
- **A real headset.** Comfort in turns and under G, text sharpness on the
  screen, and frame rate with the cockpit and the world drawn for both eyes.

## Ownership

Proposed owner: **`0sfs/vr`**, consuming
[`foss-earth/vr`](https://github.com/foss-earth/docs/blob/main/docs/proposals/vr.md). Shared immersive
session, tracking, rendering and generic UI behaviour stay in FOSS Earth and
are also used by the UMN tour. Flight owns cockpit interactions, seat and view
configuration, and the adapter between tracked poses and the aircraft and
floating-origin reference frames.

The simulator composes `0sfs/vr` and injects aircraft pose, control and camera
services through public ports. The VR adapter must not import the simulator
runtime, establish a second flight clock, or apply a second copy of flight
controls. Aircraft-specific geometry and interaction definitions belong to
their aircraft packages and are supplied through their public contracts.

Proposed direction: `simulator → 0sfs/vr → foss-earth/vr → renderer/ui`.
Neither shared VR nor the campus tour imports `0sfs/vr`.
