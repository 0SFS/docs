# Single extracted repository: agent assignment template

The coordinator must fill this block before launching an agent. This is the
later per-repository fan-out task, not permission for multiple agents to edit
one current application checkout.

| Required assignment field | Value to supply |
| --- | --- |
| Destination GitHub owner/repository | Exact provisioned remote and visibility |
| Local destination / branch / writer | Exact checkout, branch and exclusive agent |
| Primary integration owner | Earth, flight or campus primary and its contact/handoff path |
| Governing scope | Link to the component row and owner implementation brief |
| Immutable source | Source commit(s), paths and approved dirty-work disposition |
| Prerequisite contracts | Exact package versions/artifacts, exports, schemas and digests |
| Allowed source edits | Usually none; removal/barrel/lockfile edits go to the primary |
| Required behavior/assets | Existing behavior, data files, licenses, provenance and fixtures |
| Validation/adoption | Owner checks, consumer contract tests and publication gate |

Implement the assigned component in its owning destination repo. Read that
repo's instructions and the [execution rules](README.md). Verify repository
identity and clean staging before edits. Do not clone or overwrite an unrelated
remote merely because it has the desired name. The coordinator owns provisioning
and history publication; keep its source snapshot and ref mapping intact.

Extract behavior, its tests, parameters, API contracts and retained public
evidence together. Split mixed files by responsibility. Build public core,
panel, GPU/host adapter, preparation and assets entry points where applicable;
do not expose the former application's singleton or internal file paths as the
new API. A generic implementation written solely for a flight feature remains
flight-owned. New features, deferred Search and shared/flight VR are outside
this extraction. Do not provision those TODO repos or introduce their dependencies.

Consume prerequisite public exports at pinned revisions. Report missing
interfaces to their owner instead of copying implementation or introducing
reverse imports. Pure contracts/core must import without constructing a DOM,
renderer or native engine. Keep resource ownership, clocks, loops, caches and
setting homes singular across package boundaries. Preserve lifecycle disposal,
readiness/generation checks and error behavior.

Prepare a real consumable artifact with exact version/source identity, license,
required assets and compatibility metadata. Verify the archive, not just an
in-place source import. Install it in an isolated consumer fixture without the
original checkout's source links; test the actual exposed API and assets.
Do not use placeholder constants or wrapper packages around code still owned
by the monolith as proof of extraction.

Install the pinned shared authorship/CI caller and local checks. Commit only
your assigned work, with permitted attribution and no AI coauthor trailer.
Run the shared consumer-conformance command against this repo's installed
validation entry point, proving allowed attribution passes and prohibited
author/committer/coauthor fixtures fail. Include policy/caller pins and that
result in the handoff; the coordinator verifies this remote's required checks
and admission settings. This applies to docs/data/tooling repositories too,
without path-filter skips or invalid fixture commits in published history.
Run the owner's relevant checks and final suite once when the change is ready;
coordinate shared compute, retain logs and isolate failures. Do not invent
lint/test commands in a package that has none; establish appropriate checks
as part of the real component rather than reporting a nonexistent suite green.

Return a handoff containing commit/artifact identity, changed paths, exported
APIs and migration example, old-to-new file ownership, dependencies, assets,
checks, compatibility limits and the patch the primary must apply to its
consumer. The primary accepts the candidate, adopts it, removes old source and
updates app manifests/lockfiles. You do not make those edits concurrently.

Completion requires the actual consumer's acceptance receipt, the owning
repo's checks and clean publication audit, no duplicate implementation, and
documented recovery/pins. Release the writer assignment explicitly. Do not
delete old source histories, evidence or ignored build folders.
