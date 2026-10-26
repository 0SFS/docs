# Aircraft health and shared settings: local acceptance

Checked 2026-10-25 in the local `fly`, `multiplayer` and `docs` checkouts.

## Implemented behavior

One whole-aircraft Health pool starts at 100, accepts test hits, pauses and
blocks resume at zero, and repairs without automatically resuming. Off is an
explicit alternative. Changing implementations resets damage and pauses flight;
permission changes preserve health.

Shared settings have one host publisher and versioned model IDs. Host-only
editing is the default; Everyone lets the guest submit setting requests. Only
the host changes permissions. Capabilities, settings revisions, acknowledgement
barriers and pose revision checks keep connected clients on the accepted
implementation. This is protocol v2; both peers need updated clients.

## Checks

- 148 targeted tests pass across 21 files. Coverage includes health, model
  switching, app pause/resume, UI permissions, the existing phone controller,
  multiplayer startup boundaries, session/protocol behavior, forged publishers,
  stale requests, revocation races, unsupported implementations and timeouts.
- Focused ESLint passes for changed application and multiplayer code/tests.
- TypeScript passes. `npm run build` passes, including JSBSim, audio, exhaust
  and engine asset verification and emitted JSBSim/audio artifact checks.
- Nine browser review cases pass: shared settings in local Health/Off and phone
  layouts; intact, damaged, destroyed and repaired health; phone health; and an
  actual downloadable feedback report. New/changed images were inspected.
  Both shared-settings layouts report no horizontal content overflow.
- The generated feedback bundle has schema `foss-earth-test-feedback`, captures
  damage/multiplayer/diagnostics, and reports zero capture failures.
- Whitespace checks pass in the three affected repositories.

The networking tests use a controlled in-memory transport. These checks do not
claim a new two-device WebRTC qualification, physically calibrated damage or a
measured realism/cost frontier. Gun/terrain hit detection is not connected to
health. Component and physical damage remain [proposed](../proposals/aircraft-damage.md).

## Reproduce and review

From `~/gh/0sfs/fly`:

```sh
npm test -- src/flight/damage src/flight/multiplayer src/flight/multiplayerStartup.test.ts src/flight/createFlightSimApp.test.ts src/flight/createFlightSimApp.phone.test.ts src/flight/hud/FlightControlPanel.aircraft.test.tsx src/flight/hud/createFlightControlPanel.test.tsx ../multiplayer/tests
npm run build
npm run dev -- --port 7472
```

With the server running, open the [health/shared-settings test checklist](https://localhost:7472/build/damage-test.html).
It opens the Test tab. Press Start, exercise the checklist, then Finish and
Download feedback. The checklist includes the real two-pilot permission and
synchronization checks. For another device, use the network address printed by
startup and open the same `/build/damage-test.html` path. The launcher retains
the origin it was opened on, so it also works from a phone on the local network.

The local [screenshot review](../../fly/build/shots/index.html),
`fly/build/damage-review.mjs`, `damage-test-plan.json`, `damage-test.html` and
`damage-test-link.txt` are ignored local artifacts. They are not deployment
assets. Regenerate the link/launcher with `node build/create-damage-test-link.mjs`.
Every temporary server/browser started for these checks was stopped.

## Source handoff

All three inspected branches are `main`, with their corresponding
`https://github.com/0SFS/<repository>.git` push remotes. Existing uncommitted
multiplayer, invitation, stores and aircraft work was preserved. Review the
intended paths and commit them explicitly rather than staging a whole checkout.

After the reviewed changes are committed, the source push commands, if requested,
are:

```sh
git -C ~/gh/0sfs/multiplayer push origin main
git -C ~/gh/0sfs/fly push origin main
git -C ~/gh/0sfs/docs push origin main
```

These commands are recorded for handoff and were not run. Publishing the site
is a separate action.
