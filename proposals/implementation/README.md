# Parallel implementation of the repository split

The repository split is published. These briefs are historical launch notes
from 2026-10-08, not a current procedure. Where the split stands now is
[repository-split.md](../../repository-split.md). The paragraphs below are the
launch kit as written, so their links still name the roles and gates from that
date.

Status: launch specifications, 2026-10-08. These briefs describe implementation;
writing them has not launched the migration or created the target repositories.
The [split specification](../repository-split.md), [combined tab inventory](../tab-inventory.md)
and the three proposed graphs define the scope.

Start **five agents**, with one primary writer for each existing application
checkout. One agent per destination repository becomes useful after common
contracts and clean source snapshots exist. Starting all destination agents
against the current monoliths would make them compete for the same source,
barrels, settings catalogs, manifests and lockfiles.

## Starting agents

Open [ready-to-copy launch prompts](launch-prompts.md) and copy one complete
block into each session, starting in its named checkout. There are five initial
prompts, two conditional support prompts and a later component-agent template.
The initial prompts require no placeholder edits. The page also defines the
shared local ledger and sole-writer handoff files for separate sessions.

| Role | Detailed implementation brief | Initial write scope |
| --- | --- | --- |
| Coordinator | [coordinator-agent.md](coordinator-agent.md) | Migration ledger, repository provisioning, GitHub settings, clean-history publication and final cutover; no competing application edits |
| Platform | [platform-agent.md](../../../../foss-earth/docs/proposals/implementation/platform-agent.md) | New `foss-earth/ci`, `foss-earth/dev_installer` and their generic templates; consumer callers stay with their primary writer |
| FOSS Earth | [earth-agent.md](../../../../foss-earth/docs/proposals/implementation/earth-agent.md) | Current FOSS Earth checkout and assigned Earth component repos, excluding platform-owned repos |
| Flight | [flight-agent.md](flight-agent.md) | Current 0sfs checkout and assigned flight/aircraft/engine repos, including flight-specific CI/setup wrappers |
| Campus | [campus-agent.md](../../../../UMN-VR/UMN-VR.github.io/docs/proposals/implementation/campus-agent.md) | Current UMN site checkout and assigned campus repos |

All three projects have a developer setup repository: platform owns
`foss-earth/dev_installer` (shared machinery and the globe setup entry point),
flight owns `0sfs/dev_installer`, and campus owns `UMN-VR/dev_installer`.
Each offers a full default workspace and optional component selection. The
flight/campus wrappers consume the shared CLI with their application's pinned
manifest; neither duplicates its checkout, diagnostics or update machinery.

Start the coordinator first. The other four can immediately inspect their
source and prepare inventories; writes begin after the coordinator records
their exclusive checkout assignments. The primary writers remain responsible
for integration when work is delegated to destination agents.

Two existing dependency repositories need **conditional support agents**, not
automatic rewrites or new splits:

| Role | Brief | Launch condition |
| --- | --- | --- |
| Native/SDK | [native-agent.md](native-agent.md) | A concrete JSBSim SDK/native contract defect blocks extraction; otherwise keep the exact adopted tarball |
| Devices | [device-agent.md](device-agent.md) | A concrete gamepad-tools public-export/contract change is needed; otherwise consume its existing public package |

The [assignment manifest](agent-assignments.json) accounts for every repository
node in the three graphs. It is a starting work roster, not proof that a remote
exists or permission to create unrelated organization repositories. Contract
nodes describing release/workspace data are not repositories.

## Deferred features: keep as TODOs

Three proposed repositories are recorded for later work, **excluded from this
migration's implementation, provisioning and acceptance criteria**:

| Future repository | TODO and ownership |
| --- | --- |
| `foss-earth/search` | [Shared Search tab](../../../../foss-earth/docs/proposals/search.md) for finding registered tabs, settings and content across all hosts |
| `foss-earth/vr` | [Shared VR](../../../../foss-earth/docs/proposals/vr.md) for Earth, UMN and flight |
| `0sfs/vr` | [Flight VR adapter](../vr.md), consuming shared VR for aircraft/cockpit behavior |

Retain these nodes with `launch: deferred-todo` in the roster and a separate
future group in the graphs. Do not create stubs, dependencies or new current
tab IDs for them. Finishing the split does not depend on implementing them.

## What runs in parallel

| Gate | Work that can run concurrently | Exit evidence |
| --- | --- | --- |
| G0 — inventory and ownership | All agents inspect current code; coordinator records dirty work, refs, scope, source snapshots and destination paths | One writer per checkout, recoverable source snapshot, complete target roster, current-repo CI baseline and blockers |
| G1 — policy and contracts | Platform builds enforcement/setup contracts; Earth defines UI/renderer/images/scene ports; flight defines aircraft/engines/sound/control ports; campus captures routes/content/build fixtures | Versioned interfaces, fixtures and package identities; clean new-repo publication procedure; consumers acknowledge the contracts they need |
| G2 — independent components | Earth foundations/representations, flight common libraries/engine packs/features, campus About/content/club separation, docs and tooling | Each component has an audited candidate commit/artifact, public exports, checks and exact dependency pins |
| G3 — consumer adoption | Three primary writers update their own application, callers and lockfiles against accepted dependencies | No deep imports or copied implementations; current behavior/settings/assets retained; fresh install from exact package artifacts works |
| G4 — integrated release rehearsal | Owners run their final checks in scheduled compute slots; platform verifies onboarding; coordinator verifies release metadata and cutover restoration | Three buildable pinned releases, accurate About graphs, clean published-ref audits, required-check enforcement demonstrated |
| G5 — canonical cutover | Coordinator serializes ref/remote changes while writers pause only the affected publication paths | Recreated repositories only if explicitly selected; restored metadata/settings, fresh-clone validation, old push paths retired and rollback receipt |

These are dependency gates, not global barriers to useful work. For example,
flight can separate its native-free autopilot contracts before the shared UI
package is released; it cannot claim the extracted panel is integrated until
the UI contract and consumer checks exist. Earth does not need to finish every
image package before releasing its UI foundation.

The detailed briefs have local phase names. A local gate cannot override the
coordinator's publication, writer or destructive-cutover rules.

## Later fan-out: one agent per extracted repository

Use the [component assignment template](component-agent.md) once the destination
has a recorded path/remote, an immutable source snapshot, public prerequisite
contracts and a sole writer. Fill every field before launch.

- After `images` and renderer contracts: `equirectangular` and `cubemap` can run
  concurrently. `tiled-cubemap` and `preview-sheets` need the accepted cubemap
  core. `360` integrates the representation releases; `scenes` integrates `360`.
- After common engines contracts: F135, FJ33 and IO320 packages can run
  independently. Aircraft packages follow their common aircraft and engine
  dependencies; preserve cross-package data references.
- After sound's plugin contracts: `engine-sound` and `wind-noise` can run
  independently. Their implementation does not move into the Sound tab repo.
- Autopilot, logging, g-forces, controls and other flight features may run in
  separate destination checkouts when their host contracts are pinned. The
  flight primary alone edits current simulator composition and its lockfile.
- About adapters, logo generation and populated documentation/research catalogs
  can proceed independently of physics or image implementation.

The owner-specific briefs and assignment roster define each component's scope;
the template supplies the common extraction procedure. Do not give every child
permission to remove its files from the old checkout. The primary writer makes
source removals and compatibility changes after accepting the child artifact.

## Shared execution rules

- Read the current `AGENTS.md` in every assigned owner. Record the owning
  repository before writing a module. Shared globe code belongs to FOSS Earth;
  campus and flight remain consumers, with no Earth/campus dependency on 0sfs.
- Preserve existing dirty work, assets, upstream attribution, retained evidence
  and ignored working trees. Never clean, reset, stash, delete or publish them
  wholesale to make a migration command succeed.
- Use each owner's permitted scratch directory. The coordinator chooses new
  checkout paths explicitly: today's `0sfs` and `foss-earth` directories are
  existing repository roots, not empty organization folders.
- One writer owns each checkout and integration file. Cross-owner changes are
  handed to that writer. Independent worktrees are for actual concurrency and
  have their own caches; they are not a substitute for ownership.
- Reserve one heavy-test/build slot across agents on this Mac. Honor the
  half-core caps, retain logs, run changed/related checks during work and each
  affected final suite once when ready. Do not start servers or benchmarks
  implicitly. Do not retry a whole suite merely to hide a failed test.
- Every new commit must have permitted human attribution. Include the exact
  rule: **YOU CANNOT HAVE AI AS A CO-AUTHOR, OR YOUR CONTRIBUTIONS WILL BE REJECTED.**
  Historical cleanup is the coordinator's separate, frozen publication step.
- Every repository uses the common `foss-earth/ci` attribution component and
  runs its shared consumer-conformance tests through the installed validation
  entry point. Code, docs, research, assets, logos, installers, CI and `.github`
  repos all require policy/caller pins, positive/negative test results and a
  coordinator-verified remote enforcement receipt. No changed-path filter may
  skip metadata checks. Keep synthetic invalid commits out of published history.
- Normal source commits and accepted component publication follow the recorded
  workflow. Rewriting refs, deleting/recreating remotes and production deployment
  are distinct operations with concrete targets and restoration evidence.

## Handoff record

Each agent reports: owner/repository, task, source snapshot, changed paths,
candidate commit, package/version and archive digest, public exports and
dependency pins, checks/log locations, behavior/provenance limits, remaining
blockers, consumer action required and writer assignment being released.
The coordinator records durable status in its ledger; an idle agent does not
imply its destination is available to another writer.

The migration is complete only when the acceptance criteria in the coordinator
and each owner brief pass. A directory move, an empty repo, a renamed import or
a graph containing the desired names is not completion.
