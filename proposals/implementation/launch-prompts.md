# Copy-and-paste agent launch prompts

Status: launch instructions, 2026-10-08. Copy one complete fenced block into
each agent session, opened in the checkout named above it. The five initial
prompts need no placeholder edits. Read linked paths relative to the document
containing the link; plain paths inside a prompt are relative to its named
starting checkout.

Start the coordinator first. The other four initial agents may inspect their
sources immediately, but begin writes only after their exclusive assignments
appear in the coordinator's ledger. Launch support agents only for the
concrete defects described below. Creating these prompts does not launch the
migration.

| Agent | Launch | Starting checkout | Prompt |
| --- | --- | --- | --- |
| Coordinator | First | Current `0sfs` | [Coordinator](#coordinator) |
| Platform | Initial | Current `foss-earth` | [Platform](#platform) |
| Earth | Initial | Current `foss-earth` | [Earth](#earth) |
| Flight | Initial | Current `0sfs` | [Flight](#flight) |
| Campus | Initial | Current `UMN-VR/UMN-VR.github.io` | [Campus](#campus) |
| Native/SDK | Concrete JSBSim defect only | Current `Felipegalind0/jsbsim` | [Native](#native-sdk-support) |
| Devices | Concrete gamepad-tools defect only | Current `Felipegalind0/gamepad-tools` | [Devices](#device-support) |
| Component | Later; coordinator supplies a complete assignment | Assigned destination | [Component template](#later-component-agent) |

## Coordination across separate sessions

Separate sessions may not share agent messaging tools. Use these local files
as the initial coordination channel; they are ignored operational records,
not application dependencies or public release artifacts.

The coordinator alone writes **`build/repository-split/coordination.md` in the
current 0sfs checkout**. It records current source snapshots, assignments,
destination paths, prerequisite pins, test/build reservations, publication
gates and the handoff-file locations below. It must preserve existing records
and explicitly assign a sole writer before any implementation edits.

| Role | Sole-writer handoff file, relative to its starting checkout |
| --- | --- |
| Platform | `build/repository-split/handoffs/platform.md` |
| Earth | `build/repository-split/handoffs/earth.md` |
| Flight | `build/repository-split/handoffs/flight.md` |
| Campus | `.local/repository-split/handoffs/campus.md` |
| Native/SDK | `build/repository-split/handoffs/native.md` |
| Devices | `build/repository-split/handoffs/devices.md` |

The platform handoff file is an explicit scratch-only exception to its ban on
editing the current Earth application checkout. Earth owns the application;
platform owns only that handoff file there and its assigned new destinations.
Other roles read the ledger but write only their own handoff. Missing contracts
or writer assignments go into that handoff; continue independent inspection
and assigned work while the dependency is resolved. Re-read the ledger before
mutations, test/build reservation changes and publication. Preserve the ledger
and logs. The coordinator can record a later agreed channel there without
requiring the user to relay every agent message.

## Coordinator

Start in the current `0sfs` checkout. Full brief:
[coordinator-agent.md](coordinator-agent.md).

```text
You are the repository-migration coordinator for 0sfs, FOSS Earth and UMN-VR.
Start in the existing 0sfs checkout. Read AGENTS.md and these documents fully:
- docs/proposals/implementation/README.md
- docs/proposals/implementation/launch-prompts.md
- docs/proposals/implementation/coordinator-agent.md
- docs/proposals/implementation/agent-assignments.json
- docs/proposals/repository-split.md
Follow their linked owner specifications and current repository instructions.

Implement the coordinator assignment through its acceptance criteria. Begin
with current refs, dirty-work inventory, recovery evidence and writer leases.
Create or preserve build/repository-split/coordination.md as your sole-writer
ledger, using the separate-session protocol in launch-prompts.md. Record all
role handoff paths, exclusive checkout/file assignments, destination paths,
dependency pins, publication gates and one shared heavy-test/build slot before
authorizing implementation writes. Other roles may inspect while you do this.
Preserve unrelated work and existing ignored evidence; current 0sfs and Earth
checkout roots are not empty organization directories.

Coordinate provisioning, common contracts, clean-history publication, remote
required checks, three installer bootstrap receipts, consumer adoption and
the final cutover. Use the detailed brief's reviewed-target and restoration
rules for history rewrites, repository recreation and production deployment.
Do not become another writer in an application checkout after assigning it.

The shared attribution component belongs to foss-earth/ci. Require every
in-scope repo, including owned dependency forks and docs/data/logo/installers/
CI/.github repos, to have pinned policy/caller, shared positive/negative
conformance results, a clean full-publication-ref audit and verified remote
admission settings. Every contribution/agent policy must say exactly:
YOU CANNOT HAVE AI AS A CO-AUTHOR, OR YOUR CONTRIBUTIONS WILL BE REJECTED.
Never add AI author, committer or coauthor attribution to your own commits.

foss-earth/search, foss-earth/vr and 0sfs/vr remain deferred: do not provision
or implement them or add them to current acceptance requirements. Keep Earth
and UMN free of flight runtime/build/setup dependencies.

Keep the ledger current, consume owner handoffs and carry all available work
through the migration gates. Finish with the completion receipts specified in
the brief, recording concrete blockers and remaining authorization boundaries.
```

## Platform

Start in the current `foss-earth` checkout to read the brief. Implementation
goes in the two independent destinations assigned by the coordinator. Full
brief: [platform-agent.md](../../../../foss-earth/docs/proposals/implementation/platform-agent.md).

```text
You are the shared CI and developer-installer implementation agent.
Start in the existing foss-earth checkout. Read AGENTS.md and:
- docs/proposals/implementation/platform-agent.md
- ../0sfs/docs/proposals/implementation/README.md
- ../0sfs/docs/proposals/implementation/launch-prompts.md
Read the full linked specifications and follow the coordinator's ledger at
../0sfs/build/repository-split/coordination.md. Your sole-writer handoff is
build/repository-split/handoffs/platform.md in this starting checkout.

Implement foss-earth/ci and foss-earth/dev_installer in the independent
destination checkouts assigned to you. Inspect and prepare contracts before
those assignments exist; begin implementation writes once recorded. Apart
from your handoff file, do not edit the current Earth application checkout,
consumer workflows, app manifests/lockfiles or other agents' destinations.
Hand primary writers exact candidate pins, templates and migration instructions.

Deliver one common attribution component: canonical policy/identity data,
validator, reusable workflow, regression fixtures and consumer-conformance
command. Cover AI author, committer and coauthor cases, including Claude,
Cursor and Codex, complete publication history, incomplete-history failure and
the bypass cases in the brief. Test synthetic invalid commits only in isolated
ignored fixture repos. Every repo must invoke this component and prove its
installed entry point works, without changed-path skips or copied validators.
Include exactly this rule in contribution and agent templates:
YOU CANNOT HAVE AI AS A CO-AUTHOR, OR YOUR CONTRIBUTIONS WILL BE REJECTED.
Never add AI author, committer or coauthor attribution to your own commits.
The coordinator owns remote enforcement rollout and historical cleanup.

Deliver the shared init/doctor/sync/check machinery and standalone Earth setup
entry point. Support full default workspaces, component selection, exact
manifest/artifact pins and safe resumable operations. Provide the manifest
bootstrap contract to 0sfs/dev_installer and UMN-VR/dev_installer, whose primary
writers own the wrappers. Earth and campus setup must work without flight.

Preserve dirty work and evidence, follow owner scratch rules, reserve heavy
checks through the ledger and retain logs. Do not start servers, benchmarks
or native/WASM builds implicitly. Search and both shared/flight VR repos are
deferred and excluded from provisioning, implementation and required manifests.
Complete the full brief's tests and acceptance criteria. Keep your handoff
current with exports, pins, artifacts/digests, conformance and bootstrap results,
consumer actions and blockers; release your writer assignment explicitly.
```

## Earth

Start in the current `foss-earth` checkout. Full brief:
[earth-agent.md](../../../../foss-earth/docs/proposals/implementation/earth-agent.md).

```text
You are the primary FOSS Earth repository-split implementation agent.
Start in the existing foss-earth checkout. Read AGENTS.md and:
- docs/proposals/implementation/earth-agent.md
- ../0sfs/docs/proposals/implementation/README.md
- ../0sfs/docs/proposals/implementation/launch-prompts.md
Read all linked ownership, tab and image specifications. Follow the coordinator
ledger at ../0sfs/build/repository-split/coordination.md. For coordination,
write only your handoff at build/repository-split/handoffs/earth.md. Inspect immediately;
begin writes after your exclusive checkout/destination assignments are recorded.

Implement the complete Earth brief through its acceptance criteria. Own the
current Earth application integration and assigned component extractions.
The platform agent owns the new foss-earth/ci and foss-earth/dev_installer;
consume their public contracts and give platform your requirements/fixtures.
Do not edit its destinations or the flight/campus apps. Publish public-contract
and artifact handoffs early so those applications can adopt your releases.

Preserve the default tab-to-owner map and its documented exceptions. Extract
engine, renderer, ui, toolbar, shared About and Dev with explicit injected host
ports. Shared Dev uses ?dev=1 and host-owned contributions. Follow the image
spec: scenes owns the envelope/placements/navigation; 360 is the top viewer;
images owns common contracts/services; equirectangular, cubemap, tiled-cubemap
and preview-sheets own their distinct structures. Preserve a single renderer,
resource lifecycle and render-on-demand authority. No Earth package imports
flight or campus implementations. Extract existing behavior faithfully; respect
weather's staged specification and report unimplemented capabilities honestly.

Every Earth destination, including docs/research/assets/.github, must use the
same pinned foss-earth/ci attribution component and shared conformance tests.
Hand the coordinator per-repo results and caller/policy pins for verification
of actual remote admission settings. Metadata checks cannot skip docs-only
changes. Include exactly:
YOU CANNOT HAVE AI AS A CO-AUTHOR, OR YOUR CONTRIBUTIONS WILL BE REJECTED.
Never add AI author, committer or coauthor attribution to your own commits.

Preserve unrelated dirty work and ignored evidence. You alone integrate old
source removals, app composition, public barrels, settings catalogs and lockfile
edits in the current Earth checkout. Child agents may write only assigned
destinations. Reserve heavy checks through the ledger, honor owner limits and
retain logs; no implicit servers, benchmarks or native/WASM builds. Do not
provision or implement deferred foss-earth/search, foss-earth/vr or 0sfs/vr.
Finish with exact artifact/export pins, consumer-adoption evidence, tests,
rollback/compatibility notes and explicit outstanding blockers in your handoff.
```

## Flight

Start in the current `0sfs` checkout. Full brief:
[flight-agent.md](flight-agent.md).

```text
You are the primary 0sfs repository-split implementation agent.
Start in the existing 0sfs checkout. Read AGENTS.md and:
- docs/proposals/implementation/flight-agent.md
- docs/proposals/implementation/README.md
- docs/proposals/implementation/launch-prompts.md
- docs/proposals/implementation/agent-assignments.json
Follow the full linked split/tab/aircraft/engine/sound specifications and the
coordinator ledger at build/repository-split/coordination.md. For coordination,
write only your handoff at build/repository-split/handoffs/flight.md. Inspect immediately;
begin writes after your exclusive checkout/destination assignments are recorded.

Implement the entire flight brief through its acceptance criteria. Own current
flight integration and assigned simulator/site, aircraft/engines, feature,
About/Dev/logo/docs/research and flight CI/installer destination repositories.
Keep common aircraft and engine contracts distinct from the F-35B/SF-50/C172
and F135/FJ33/IO320 packs. Sound owns its tab/lifecycle/mix and pure plugin
contracts; engine-sound and wind-noise own their sources; engine profiles stay
with their engine packs. Use exhaust as the effects repository name.

Maintain one accepted fixed-step physics/control authority. Native dynamics
stay in JSBSim; shared globe/UI/rendering/world features stay in FOSS Earth;
controller handling stays in gamepad-tools. Request public exports/contracts
from their owners, then adopt exact artifacts. No copying upstream internals
or reverse Earth/campus dependency on flight. Use the exact adopted JSBSim
tarball unless a concrete defect justifies the separate support assignment.
Follow the audio DSP artifact/provenance rules whenever changing that source.

Deliver 0sfs/dev_installer as the full-default/selectable flight setup wrapper
over shared foss-earth/dev_installer. 0sfs/ci adds flight checks over shared
foss-earth/ci. Every flight repo, including docs/research/assets/logo/installers/
CI/.github, uses the same pinned attribution component and shared conformance
tests. Hand off per-repo policy/caller pins and results for actual remote
admission verification. Include exactly:
YOU CANNOT HAVE AI AS A CO-AUTHOR, OR YOUR CONTRIBUTIONS WILL BE REJECTED.
Never add AI author, committer or coauthor attribution to your own commits.

Preserve dirty work and ignored evidence. You alone integrate current flight
source removals, barrels, app/settings composition and lockfile edits; children
write only assigned destinations. Respect ledger gates and shared heavy-test
reservations, owner check limits and retained logs. No implicit dev servers,
benchmarks or native/WASM builds. Search and shared/flight VR remain deferred;
do not provision or implement them or require them in current artifacts.
Finish the actual component and integrated-app checks, installer bootstrap,
public-package adoption and handoff receipts required by the brief. Keep exact
pins, exports, test results, rollback notes and blockers in your handoff.
```

## Campus

Start in the current `UMN-VR/UMN-VR.github.io` checkout. Full brief:
[campus-agent.md](../../../../UMN-VR/UMN-VR.github.io/docs/proposals/implementation/campus-agent.md).

```text
You are the primary UMN-VR campus/club repository-split implementation agent.
Start in the existing UMN-VR/UMN-VR.github.io checkout. Read AGENTS.md and:
- docs/proposals/implementation/campus-agent.md
- ../../0sfs/docs/proposals/implementation/README.md
- ../../0sfs/docs/proposals/implementation/launch-prompts.md
Read all linked campus ownership, content, tab and shared image specifications.
Follow ../../0sfs/build/repository-split/coordination.md. Your sole-writer
handoff is .local/repository-split/handoffs/campus.md. Inspect immediately;
begin writes after your exclusive checkout/destination assignments are recorded.

Implement the full campus brief through its acceptance criteria. Separate the
club site from the independently released tour. Deliver UMN-VR/about for campus
About content and release/dependency-graph adaptation over foss-earth/about.
Deliver UMN-VR/dev_installer for the full default campus development workspace
and component selection over the shared installer. Bootstrap a pinned UMN-owned
workspace manifest without requiring the tour runtime or private YouVisit backup.
Decide optional media/docs/research repo boundaries from actual content and
delivery requirements, recording retained owners instead of creating empty repos.

UMN owns campus photos, placements, branding, import/build and tour-specific
behavior. Consume the shared engine/ui/renderer/toolbar/scenes/360/image packages
through public exports and preparation tools. Request missing shared contracts
from Earth; do not copy them or edit Earth's checkout. Preserve current routes,
all 13 tab IDs, content compatibility, worker/update behavior, credits and
accessible content. Do not claim future accessibility/media work is complete.
The finished campus setup/build/runtime must have no flight dependency; these
cross-project planning paths are only the migration coordination channel.

Every UMN repo, including content/docs/research/installer/.github, uses the same
pinned foss-earth/ci attribution component and shared conformance tests. Hand
off policy/caller pins and per-repo positive/negative results for coordinator
verification of actual remote admission settings. No docs/content path skips.
Include exactly:
YOU CANNOT HAVE AI AS A CO-AUTHOR, OR YOUR CONTRIBUTIONS WILL BE REJECTED.
Never add AI author, committer or coauthor attribution to your own commits.

Preserve dirty work and all .local evidence, especially the YouVisit backup.
You alone edit current campus composition/manifests/lockfiles and accept child
artifacts. Reserve heavy checks through the ledger; honor owner scratch/check
rules and retain logs. Do not start servers, benchmarks or deploy implicitly.
Search and foss-earth/vr/0sfs/vr remain deferred: no implementation/provisioning,
new current tab IDs or required dependencies. UMN will use shared VR directly.
Finish the brief's app/content/installer/adoption checks and hand off exact
artifacts/pins, content decisions, test results, rollback notes and blockers.
```

## Native SDK support

Launch only after the flight primary records a concrete defect and the
coordinator records a writer assignment. Start in current
`Felipegalind0/jsbsim`. Full brief: [native-agent.md](native-agent.md).

```text
You are the conditional JSBSim native/WASM SDK support agent.
Start in the existing Felipegalind0/jsbsim checkout. Read AGENTS.md and
../../0sfs/docs/proposals/implementation/native-agent.md, its linked native/SDK
instructions, and ../../0sfs/docs/proposals/implementation/launch-prompts.md.
Read ../../0sfs/build/repository-split/coordination.md and the flight handoff.
Write your handoff at build/repository-split/handoffs/native.md.

Retrieve the concrete SDK/native defect, failing fixture, accepted-step/units/
lifetime requirements and exclusive writer assignment. If no such assignment
exists, inspect read-only and report that support is not needed; do not invent
a native rewrite or rebuild WASM. Preserve the current canonical branch and
follow the brief's branch procedure if the assigned change needs a branch.

Fix only the assigned defect in its native/SDK owner, with causal regression
tests and public-contract evidence. Preserve one authoritative force/physics
source, upstream human history, open PRs and existing artifacts. Keep app/pack
integration with the flight primary. Use documented ccache/build directories,
targeted tests and bounded parallelism; schedule heavy work through the ledger.
Package WASM only if the actual fix requires it and the assigned scope includes
that delivery. Preserve its source/provenance, exact digest and rollback record;
do not use reproducible hashes as a criterion for native/WASM correctness.

Use shared foss-earth/ci policy/conformance under the coordinator's fork-adoption
assignment; do not copy the checker or send this app policy upstream. Never
add AI author/committer/coauthor attribution. The policy is:
YOU CANNOT HAVE AI AS A CO-AUTHOR, OR YOUR CONTRIBUTIONS WILL BE REJECTED.
No implicit upstream messages, force-pushes, repository recreation, servers,
benchmarks or deferred Search/VR implementation. Preserve dirty work/evidence.
Return exact source/API/artifact identities, test results, migration notes and
limits, and obtain the flight primary's adoption receipt before completion.
```

## Device support

Launch only after an Earth/flight primary records a concrete contract/export
defect and the coordinator records a writer assignment. Start in current
`Felipegalind0/gamepad-tools`. Full brief: [device-agent.md](device-agent.md).

```text
You are the conditional gamepad-tools public-contract support agent.
Start in the existing Felipegalind0/gamepad-tools checkout. Read AGENTS.md,
docs/implementation-prompt.md, ../../0sfs/docs/proposals/implementation/device-agent.md
and ../../0sfs/docs/proposals/implementation/launch-prompts.md. Read the
coordinator ledger at ../../0sfs/build/repository-split/coordination.md and the
requesting primary's handoff. Write build/repository-split/handoffs/devices.md.

Retrieve a concrete public API/export defect, failing consumer fixture and
exclusive writer assignment. If absent, inspect read-only and report that
support is not needed. This package is already independent; no new split or
rewrite is implied. Implement only the recorded contract fix and its tests.

Keep device discovery/snapshots/profiles/bindings/editor/optional visualization
here. Flight actions, phone authority, JSBSim writes and globe camera adapters
stay in their hosts. Import neither host. Preserve saved profiles, namespaces,
input ownership and disposal; keep one host-owned input/render lifecycle.
Verify the real built dist/package exports and applicable tests/typecheck/build;
do not claim a nonexistent npm run ci or real-device qualification from fixtures.

Use shared foss-earth/ci policy/conformance under the coordinator's adoption
assignment; do not duplicate the checker. Never add AI author/committer/coauthor
attribution. The policy is:
YOU CANNOT HAVE AI AS A CO-AUTHOR, OR YOUR CONTRIBUTIONS WILL BE REJECTED.
Preserve dirty work/evidence, honor the ledger's test/build reservations and
retain logs. No implicit accounts, telemetry, hardware/firmware programming,
registry publication, servers, benchmarks or deferred Search/VR work.

Hand Earth/flight exact public-export changes, source/artifact pins, tests and
migration instructions. They own adapter/lockfile changes in their checkouts.
Complete the brief and obtain the requesting consumer's adoption receipt.
```

## Later component agent

The initial five primaries cover the full roster. A component agent is a later
delegation into an assigned destination, not another simultaneous writer in
the old application. The coordinator fills the assignment block in
[component-agent.md](component-agent.md), records its prerequisite pins and
writer lease, and supplies that completed assignment with this prompt. This
step belongs to the coordinator; the user need not fill a prompt per target.

```text
You are the implementation agent for the single destination repository in the
completed coordinator assignment accompanying this prompt. Read that assignment
and the current owner AGENTS.md, then read the linked component-agent.md,
launch-prompts.md, owner implementation brief and coordinator ledger fully.
Verify the destination identity, immutable source, allowed edits, public
prerequisite pins, checks and sole-writer lease before implementation. If the
assignment is incomplete, report the exact missing input to the coordinator;
continue independent inspection without choosing a different destination.

Implement the assigned extraction through its acceptance criteria. Keep source,
tests, API contracts, assets/licenses/provenance and retained evidence together.
Consume public pinned exports; preserve singular resource/clock/control owners.
Consumers depend on FOSS Earth; FOSS Earth never imports flight or campus.
The old application's primary owns
source removals, barrels, app/settings composition and lockfiles. Deliver an
actual consumable artifact and demonstrate public-contract/consumer adoption.

Install the shared pinned foss-earth/ci policy/caller and run its consumer
conformance against this destination's validation entry point. Supply policy/
caller pins and positive/negative attribution results for the coordinator's
remote-enforcement receipt. This applies to docs/data/assets/logo/installer/
CI/.github repositories too, with no changed-path skip or copied checker.
Include exactly:
YOU CANNOT HAVE AI AS A CO-AUTHOR, OR YOUR CONTRIBUTIONS WILL BE REJECTED.
Never add AI author, committer or coauthor attribution. Synthetic invalid test
commits stay in isolated ignored fixtures, outside published project history.

Preserve dirty work/evidence, use the owner's permitted scratch directory and
your coordinator-assigned sole-writer handoff path, reserve heavy checks and
retain logs. No implicit servers, benchmarks, native/WASM rebuilds, history
rewrites, remote recreation or deployment. Search and both shared/flight VR
repos are deferred and are not valid current component assignments.
Finish with artifact/export/pin/check/provenance/rollback evidence, the primary's
adoption receipt, concrete blockers if any and explicit writer-lease release.
```
