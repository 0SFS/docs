# Flight extraction agent

Status: implementation assignment, 2026-10-08. Copy this document into one
agent working in the existing canonical flight checkout. This agent owns the
flight extraction and its newly assigned `0sfs/*` repositories. Run it alongside the FOSS Earth,
UMN and shared-platform agents under the migration coordinator. See the
[implementation index](README.md) and
[coordinator assignment](coordinator-agent.md). It must not edit another
agent's checkout while that agent owns it. This document is an execution
assignment for a future agent; writing it does not initiate extraction.

## Assignment

Implement the flight part of the
[repository split specification](../repository-split.md), preserving existing
behavior while making each extracted repository independently usable. The
[combined tab inventory](../tab-inventory.md) and
[flight graph data](../repository-split-graph.json) define the proposed owners;
they are architecture inputs, not proof that the implementation already has
those boundaries. Recheck the current source before moving it.

Deliver working packages, pinned flight assembly, contribution enforcement,
documentation, and migration evidence. A collection of empty repositories,
copied directories, or packages that still deep-import the old app does not
complete this assignment. New capabilities such as ArduPilot SITL, a broader
weather model, or additional engine audio sources are separate work. The
proposed future Search tab and `foss-earth/search` are deferred TODOs, excluded
from this migration assignment and the current tab counts. Do not implement
Search, create its repository or add its dependency while executing this
assignment. A future flight Search contribution will consume the shared FOSS
Earth owner; preserve the existing Location/search behavior in the meantime.

Likewise, [`0sfs/vr`](../vr.md) and the shared `foss-earth/vr` are deferred
TODOs for a later assignment. Record their ownership without implementing,
provisioning or adding dependencies on them. They do not block this migration.

The coordinator supplies the approved repository manifest: canonical remote,
local checkout path, writer, visibility, package identity, release strategy,
source revision and provision/cutover state for each target. Before writing
any new module, name its owner and use that checkout. Do not invent a new
shared repository, create remote repositories outside that manifest, or put
future shared source in the flight app as a temporary permanent home. Request
missing target provisioning from the coordinator and continue independent
flight work while it is prepared.

Only this agent writes the current flight checkout and its shared integration
files. A delegated child may own a **distinct, provisioned destination
repository**, after receiving an explicit exclusive writer assignment. The
child can use the [component assignment](component-agent.md). It returns an
exact commit/artifact and checks; this primary agent updates
the simulator's imports, lockfiles, settings composition, assembly and graph.
Do not have a child edit `createFlightSimApp.ts`, `FlightControlPanel.tsx`,
`flightParameters.ts` or the simulator manifest concurrently with this agent.

## Read and record before changing source

Read `AGENTS.md` in the current checkout and every destination, then:

- [Current FOSS Earth boundary](https://github.com/0SFS/fly/blob/main/docs/foss-earth-relationship.md),
  [flight settings](https://github.com/0SFS/fly/blob/main/docs/proposals/flight-settings.md),
  [aircraft assets](https://github.com/0SFS/aircraft/blob/main/docs/aircraft-assets.md),
  [JSBSim integration](https://github.com/0SFS/fly/blob/main/docs/jsbsim.md), and
  [validation layout](../../validation/layout.md).
- [Sound specification](https://github.com/0SFS/sound/blob/main/docs/sound.md) and
  [audio implementation ledger](https://github.com/0SFS/sound/blob/main/docs/validation/audio-implementation-ledger.md).
  Preserve the difference between implemented, tested and qualified behavior.
- The FOSS Earth agent's published contracts for UI/settings, toolbar,
  renderer, surface queries, sky/weather, About and Dev, and the platform
  agent's workspace-manifest, installer and reusable-CI contracts. The
  platform agent owns `foss-earth/ci`, `foss-earth/dev_installer` and generic
  repository scaffolds; this flight agent owns flight adapters and callers.
  Consume public packages at their reported accepted revisions.

Record `git status`, source HEAD, branch, remotes, tracked asset inventories,
existing package links and adopted artifact identities in a dated local
`build/` report. Before installing packages, inspect the current manifests
and lockfile changes. At specification time, `package.json`,
`package-lock.json`, `scripts/check-landing-page.mjs` and `q0.md` already had
uncommitted changes. Reinspect at execution time; these are user/other-session
work, not disposable migration inputs. Preserve them, reconcile necessary
manifest edits deliberately, and stage only this assignment's changes.

Do not reset, clean, stash, overwrite or publish unrelated working changes.
Never delete any `build/` folder or move its contents wholesale. It contains
working sources and retained evidence. Scratch and logs belong under the
owning checkout's `build/`; distributable evidence belongs in that owner's
`validation/evidence/` with rights and provenance intact. Do not publish
personal data, local absolute paths, credentials, or raw private inventory.

## Owned destinations

The following names are repository destinations, not a claim that matching
package names already exist. The coordinator records actual package names and
export maps before publication.

| Target | Required ownership |
| --- | --- |
| `0sfs/simulator` | Accepted fixed-step execution, JSBSim adapter, scene/flight clock, reset/saved-flight lifecycle, flight cameras/floating origin, host adapters, final control application, feature registration and compatible release assembly |
| `0sfs/0sfs.github.io` | Public flight entry and publishing configuration, consuming an exact simulator release; no second simulator implementation |
| `0sfs/aircraft` | Pure aircraft contracts, catalog/loading and installation interfaces, shared visuals/animation infrastructure and Aircraft panel; host registers concrete aircraft |
| `0sfs/engines` | Pure engine contracts, registration/loading, observations/commands, monitoring/history/instruments, Engine panel and test-stand interface |
| `0sfs/F-35B`, `0sfs/SF-50`, `0sfs/C172` | Complete aircraft definitions/assets/installations, variants, provenance, component tests and released evidence |
| `0sfs/F135`, `0sfs/FJ33`, `0sfs/IO320` | Engine-family definitions, geometry/generators, acoustic and optical profiles, calibration, provenance and compatible native-engine requirements |
| `0sfs/autopilot` | Hold modes/controller, engagement/takeover, observations/commands, settings and panel |
| `0sfs/controls` | Flight actions, keyboard response, assists, flight haptic interpretation and panel sections; device handling stays in gamepad-tools |
| `0sfs/remote-control` | Phone app, pairing/transport/protocol, camera transport/playout, sharing and settings, using explicit simulator adapters |
| `0sfs/fuel` | Fuel panel and management/installation interfaces; aircraft defines tanks and plumbing; JSBSim retains mass/fuel dynamics |
| `0sfs/g-forces` | Pilot-load interpretation, consciousness/blackout/redout behavior, overlays, settings and panel, retaining current model limits |
| `0sfs/logging` | Flight recorder, channels, segments/marks, bounded history, exports and Logging panel; shared status log remains FOSS Earth UI |
| `0sfs/sound` | Sound tab, lifecycle/worklet, mixing/propagation, volume/resource controls and pure source-plugin contracts |
| `0sfs/engine-sound` | Engine source synthesis consuming engine profiles and sound's pure contracts; no imports of specific engine-family packs |
| `0sfs/wind-noise` | Aerodynamic/airframe source model consuming accepted observations and aircraft configuration through sound's contracts |
| `0sfs/exhaust` | Plume optics/rendering, smoke, depth handling, exhaust hot surfaces, controls and diagnostics; engine packs supply optical data |
| `0sfs/ground-contact` | Aircraft/world contact adaptation, collision query/response integration and diagnostics; terrain stays FOSS Earth, native force laws stay JSBSim |
| `0sfs/about` | Flight About content, credits/links and resolved-manifest adapter consuming `foss-earth/about` |
| `0sfs/dev` | Flight Dev contributions using `foss-earth/dev` and its `?dev=1` gate |
| `0sfs/logo` | Flight logo source/generation tools, exported assets and provenance |
| `0sfs/docs`, `0sfs/research` | Flight documentation/catalog and flight research, with component contracts/tests/evidence retained beside their implementation |
| `0sfs/dev_installer` | Flight onboarding wrapper and explicit JSBSim development path using FOSS Earth's installer and the simulator-owned workspace manifest |
| `0sfs/ci` | Flight integration/release checks using immutable shared `foss-earth/ci` revisions |
| `0sfs/.github` | Flight organization contribution/agent policy and templates; every child still installs its own actual CI caller |

There is no flight-owned `sky`, `weather`, shared renderer, UI, toolbar, scene
or 360 implementation. Both flight and UMN consume FOSS Earth. FOSS Earth and
UMN must gain no runtime, build or setup dependency on any `0sfs/*` package.
Native dynamics and the WASM SDK stay in JSBSim. Device handling stays in
gamepad-tools. Send those owners a concrete contract/defect request instead
of adding an application workaround or editing their files concurrently.

## Ordered implementation and handoff gates

### 1. Establish the source map and package contracts

Inspect at least these current sources and their adjacent tests:

| Current source | Extraction work |
| --- | --- |
| `src/flight/createFlightSimApp.ts`, `src/flight/hud/FlightControlPanel.tsx`, `src/flight/settings/flightParameters.ts` | Separate host composition from feature implementation/catalogs/panels; preserve all 21 current registered tab IDs and every parameter's current home |
| `src/flight/aircraft/`, `public/aircraft/`, `public/jsbsim-data/` | Classify shared aircraft code, individual airframes, engine data, installation accessories, generated outputs and licenses individually |
| `src/flight/jsbsim/`, `deps/`, `src/flight/bridge/`, `src/flight/physics/` | Keep host/native adaptation separate from common feature contracts and aircraft-specific definitions |
| `src/flight/autopilot/`, `src/flight/input/`, `src/flight/remote/`, `src/remote/` | Remove feature imports of application internals; preserve arbitration, phone protocol and control ownership |
| `src/flight/audio/`, its `dsp/` and `worklet/`, `scripts/build-audio-wasm.mjs`, `scripts/verify-audio-artifact.mjs` | Map each source model, shared DSP primitive, transport/lifecycle concern, profile and artifact to its actual owner |
| `src/flight/aircraft/engine*`, `createEngine*`, `scripts/exhaustOptics/`, `scripts/build-f135-engine.mjs`, `scripts/build-exhaust-optics.mjs` | Split common exhaust rendering from F135 optics/geometry data and aircraft installation; keep generation/provenance with its output |
| `src/flight/hud/engine*`, `src/flight/engineTestStand.ts`, `src/flight/jsbsim/fuelTanks.ts`, `src/flight/physics/gForce.ts`, `src/flight/hud/loggingTab.ts` | Trace feature lifecycles and panel dependencies before selecting exact source moves |
| `vite.config.ts`, entry pages, `public/`, existing build/deploy scripts | Separate simulator assembly from entry-site publishing without breaking flight or phone routes |

Produce a tracked flight extraction map with old path → owner/new path,
public export, tests, assets/provenance and consumer adaptation. Record actual
versioned interfaces: units, coordinate frames, timestamps, reset/generation
identity, missing values, ownership/disposal, commands and errors. Core modules
must load without React, DOM or renderer initialization. Do not create a
contracts package containing the entire mutable application state.

**Gate:** coordinator and affected owners agree the contract revisions and
repository writer assignments. Pure flight contracts and source classification
can proceed while FOSS Earth extracts its foundations. Consumer integration
waits for usable public exports and exact dependency pins; do not substitute
deep imports into an unfinished sibling checkout.

### 2. Consume the shared foundations

Replace shell/application singletons with host-created settings, feature
catalog registrations, panel contributions, toolbar launchers/readouts and
explicit renderer/surface services. Shared package readiness must include the
flight contract tests, not merely the globe build. Preserve the one settings
record, presets and migrations, the existing tab IDs and saved workspace
state. A toolbar action toggles the existing tab; it does not create a second
settings view.

Consume shared Sky and Weather incrementally at their supported capability.
Today's flight Weather supplies wind speed/direction. Adapt the accepted
sample into JSBSim at the host step boundary; extraction does not implement
the entire weather proposal. Keep one scene time, one weather field and one
lighting/exposure result. Opening a panel never creates another physics loop,
weather world, cache, render loop or audio world.

**Gate:** the current flight behavior works through published public shared
interfaces with no reverse dependency from FOSS Earth. Flight remains usable
at this intermediate assembly revision. Report exact consumer pins and
contract-test results to the coordinator.

### 3. Extract common aircraft/engine contracts, then complete packs

Establish `aircraft` and `engines` **contracts first**. Common registries accept
host-provided definitions; they never import a hardcoded fleet or engine list.
Extract complete engine packs, then the aircraft packs that pin them:
F-35B → F135, SF-50 → FJ33, C172 → IO320. Preserve the current engine identity
from `eng_io320.xml`; do not relabel it to another engine family. Preserve
SF50 generations and both current F-35B model lineages and their status.

Define a package manifest with logical asset IDs, exact compatibility/pins,
hashes, package-relative URLs, native model/system/engine paths, licenses and
optional geometry/sound/exhaust capabilities. Resolve the exact transitive
set into the JSBSim virtual filesystem. Reject conflicting files/IDs,
incompatible schemas and missing references before installing anything.
Asset resolution must work with the old `public/` tree unavailable and under
a non-root deployment base path.

Aircraft owns position/orientation/mounting, source positions, tanks, contact
geometry, propulsion installation and aircraft control-law configuration.
Keep `prop_75in2f.xml` and the empirical F-35B liftfan/sidefan definitions with
those installations unless a separate approved accessory owner is introduced.
The generic JSBSim-derived `direct.xml` has one assembly resource identity and
provenance; do not copy it into every engine as independent authored data.

Native algorithms remain in JSBSim. Use the existing adopted `deps/` tarball
and `verify:jsbsim`; the normal split does not justify a native/WASM rebuild.
Its build is not reproducible, so different hashes from repeated builds do
not prove source drift. Preserve known artifact identity and provenance.

**Gate:** independent pack validation plus simulator integration covers all
three aircraft families, engine references, variants, model installation,
selection/reset, missing optional capabilities, license inclusion and package
URL resolution. A deliberately conflicting or incomplete asset manifest fails
before installation. Retire an old source only after the corresponding
consumer uses the accepted pack; leave no live fallback copy.

### 4. Extract flight features in dependency order

Move one feature and its tests/catalog/panel/diagnostics together, update the
host, then validate that vertical slice before extracting the next dependency.
The final order follows actual readiness; common contracts precede dependents.

Autopilot must return intents/status, not write native controls. Keep pilot,
remote, autopilot and aircraft control-law arbitration at one accepted host
boundary and apply each final control once. Remove autopilot settings' import
of the whole flight catalog and output types' dependence on the input manager.
Preserve the existing disconnected/disabled ArduPilot status; do not advertise
a working SITL bridge.

Fuel preserves JSBSim mass/fuel ownership and aircraft tank configuration.
Engine monitors and the phone app receive observations/commands through their
own public contracts. Presentation and engine test-stand views do not advance
physics independently. G-forces reports pilot availability; host control
application enforces it. Logging records accepted state and applied controls,
not speculative/render-interpolated physics as though it were accepted data.

For sound, register `engine-sound` and `wind-noise` from the host. `sound`
defines pure plugin contracts and has no import of those concrete plugins.
Engine profiles are supplied data, not imports from F135/FJ33 inside the
generic synthesizer. The current DSP combines source paths: extract a real
wind/airframe model boundary with measurable output/ABI parity; an empty repo
or wrapper that keeps the model in `engine-sound` does not meet this split.
Extract only genuinely shared DSP primitives into a dependency-safe pure
entry point. Keep tire/contact sound in the ownership map too; no existing
source disappears because the requested new repositories name only engines
and wind.

Preserve or explicitly version and verify snapshot/parameter/worklet/WASM
ABIs. The current engine renderer admits one sound source, the C172 has no
engine-sound installation, and F135 has main-engine sound without a separate
lift-fan source. Do not fabricate capability entries. No audio tier is device
qualified. If DSP source changes, run the owner's migrated `build:audio`
equivalent and commit the WASM with its provenance; source, build scripts and
artifact must have one canonical owner and recorded dependency inputs.

Exhaust owns plume/smoke/hot-surface implementation; engine packs own optical
data and mechanical engine geometry. Afterburner audio synthesis remains in
`engine-sound`, with volume owned by `sound` and its existing single UI home
composed into Exhaust as needed. Verify generated optical/engine assets without
rebaking unchanged outputs solely for a move.

Extract ground contact only after aircraft/native/surface interfaces exist.
Keep native gear/contact force laws in JSBSim and surface height/datum/readiness
in FOSS Earth. Preserve experimental solver labels, reset identity and accepted
step behavior. A new repo is not evidence of physical validation.

**Gate for every feature:** it builds/tests independently, exports its public
API, passes retained behavioral tests plus consumer integration, has no
reverse/deep app import or duplicate live implementation, preserves settings
IDs/tab placement and registers/tears down through host-owned services.

### 5. Complete composition, project tooling and documentation

Finish `simulator` versus `0sfs.github.io`: the site consumes an exact simulator
artifact/release and preserves existing entry/phone routes. Prepare publishing
configuration and a reviewable build; live deployment is the coordinator's
cutover action. Remove copied app source from the website only after the pinned
artifact works independently.

Build `0sfs/about` around `foss-earth/about`. Feed it a resolved manifest from
the same release assembly used for package resolution: actual installed
versions/revisions/artifacts, native WASM, content, third-party/transitive
dependencies, license/provenance and typed relationships. It must not import
the simulator or every dependency to discover its identity. The draft graph
remains clearly labeled proposed architecture; the running About graph reports
the actual build. No local paths or credentials appear in its public data.

Implement flight Dev contributions with shared `?dev=1` activation. Without
the flag, the Dev tab is absent and its optional diagnostics do not start.
With the flag, only flight-relevant contributions are installed. Opening Dev
must not start a server, benchmark, capture or second simulation loop. Keep
the current Debug tab and feature diagnostic homes unless an explicit migration
is agreed; Dev is not silent deletion or renaming of Debug.

Extract logo assets together with available source/generation procedures and
provenance. Inventory what actually exists; do not claim existing logo scripts
if only assets can be found. Any new generator must reproduce the accepted
brand output through documented inputs and checks before replacing it.

Move flight documentation and research by subject, preserve links/redirect
notes, and keep API contracts, regression tools and release evidence with
components. Do not copy FOSS Earth or campus specifications into a competing
flight authority. Preserve the combined tab overview as a cross-project index
that links the owners' inventories. Update the extraction map and graph after
each accepted boundary, distinguishing planned from shipped packages.

Use shared `foss-earth/dev_installer` with the simulator's versioned workspace
manifest. `0sfs/dev_installer` supplies complete flight setup, selectable subsets
and an explicit native-engine-development path. Default setup uses the adopted
JSBSim artifact. Validate init/doctor/sync/check, dependency build ordering,
resume after failure, exact revisions, artifact verification and dirty/diverged
checkout protection. Do not duplicate the repository list inside CLI source.
No implicit system installation, deletion, branch reset or server startup.

`0sfs/ci` consumes an immutable reviewed `foss-earth/ci` pin and adds flight
integration against exact candidate dependency sets. Install a small actual
workflow caller in every flight code/content/docs/research/tool repo; local
checks call the same policy. Coordinate job identities and required checks
with the coordinator. Organization templates alone are not CI enforcement.
Every destination runs the shared consumer-conformance command against its
installed validator entry point and hands off its policy/caller pin and
positive/negative attribution results. This includes docs, research, content,
logo, installer, CI and `.github` repos, with no path-filter skip. The coordinator
verifies each remote's required admission checks; do not duplicate the validator.

Every repository's contribution and agent instructions must contain:

> YOU CANNOT HAVE AI AS A CO-AUTHOR, OR YOUR CONTRIBUTIONS WILL BE REJECTED.

Reject AI authors, committers and identity-bearing attribution trailers using
the shared validator and its configured identity rules. Preserve genuine human
attribution, notices and licenses. Do not add AI trailers to this work. Inspect
the proposed commit metadata/message before committing, without printing
unnecessary personal information. A full-history audit must fail rather than
silently pass on shallow/incomplete history. Existing contaminated ancestry
is a cleanup gate handled by the coordinator; do not weaken the full-history
rule to make existing refs green. Do not fabricate replacement human authors.

### 6. Validate the complete flight assembly and hand off cutover

Use the owning repositories' documented checks. For each source change, run
incremental typecheck/lint and related tests, with meaningful extra boundary
tests where needed. Disk-read asset fixtures must trigger tests when they
change. Run each finished owner's full required suite once and the completed
flight integration suite once after the final pins. Coordinate machine-heavy
checks with the other agents; cap repeated/background workers at half the
cores, and never overlap full suites. Keep logs under `build/` and investigate
failures with the failed test, not repeated full-suite runs. No `--force`,
cache deletion or `tsc -b` to bypass incremental procedures.

The completed assembly must demonstrate:

- All 21 current flight tabs and their conditional sections retain behavior,
  settings homes and saved state; Dev is separately opt-in. Flight still has
  no Scenes/360 tabs. Toolbar actions toggle the same registered panels.
- Aircraft/engine variants load from independently packaged assets with the
  former public asset tree unavailable. Installed engine identity, optional
  capabilities, licenses/provenance and JSBSim references remain accurate.
- Pause/reset/load/location changes preserve fixed-step and reset-generation
  semantics across contacts, autopilot, logging, remote controls and audio.
  Pilot takeover and final control arbitration apply commands once.
- Renderer readiness/disposal, render-on-demand, camera momentum and unused
  input propagation remain intact; no extracted feature creates a second
  host loop or resource budget.
- Sound source registration/disposal, missing profile behavior, accepted
  snapshot transport, ABI compatibility and retained numerical/output tests
  pass. Exhaust/engine generated-asset verification and adopted native/audio
  artifact verification pass with retained provenance.
- Pure contracts/core exports import without DOM/React/renderer initialization;
  package dependency validation catches cycles, old deep imports and a
  FOSS Earth/UMN → 0sfs path. Consumers use accepted public exports only.
- A clean isolated **source checkout/build**, at exact manifest pins and
  without undeclared sibling files, passes the documented commands. Use an
  assigned checkout location; do not disturb or delete existing working trees.
- The About manifest reflects the actual packaged artifacts and graph edges;
  Dev activation works; flight onboarding is repeatable and refuses unsafe
  dirty/diverged updates; every target has the authorship CI caller.

Do not launch dev/preview/watch servers, run GPU benchmarks, phone-layout
qualification or rebuild native WASM merely to prove the split. Use terminal
and relevant existing noninteractive checks. If browser verification is
necessary within the authorized task, use a headless check and close its
browser in `finally`; never kill it after normal close. Report any behavior
that genuinely requires separately scheduled interactive/GPU validation
without declaring it qualified.

Commit only assigned work in the canonical checkout with genuine configured
human metadata. Normal pushes of new audited branches/artifacts follow the
coordinator's publication state and exact ref allowlist. Do not push old tags,
backup refs or a mirror of the contaminated source history. Do not rewrite,
force-push, delete, rename/recreate repositories, switch production routes or
deploy as part of this agent assignment. These are serialized coordinator
cutover operations after all writers have stopped and backup/restoration
evidence is verified.

Return a tracked completion report containing repository/commit/artifact pins,
public exports and contracts, source move map, updated workspace/release
manifest, independent and consumer check results, remaining failures/limits,
asset/license provenance, history-audit status and exact outstanding cutover
actions. Include the last source revision consumed so the coordinator can
detect late source commits before extraction is declared complete. Claim a
repository extracted only when its owner and consumers meet these gates.
