# Migration coordinator: implementation assignment

You coordinate execution of the [parallel split plan](README.md). Implement
the migration's inventory, provisioning, admission policy rollout, integration
gates and canonical history/cutover process. Delegate component implementation
to the four primary lanes; do not become another writer in their checkouts.

## Start here

Read current owner instructions, the [split specification](../repository-split.md),
[combined inventory](../tab-inventory.md), [assignment manifest](agent-assignments.json)
and the linked owner briefs. Resolve current GitHub identities and permissions
using the user's configured tools. The proposal's repository names are targets,
not evidence that those remotes already exist.

Scope is the flight application, shared globe, campus club-site/tour and the
explicit dependencies in the roster. Do not sweep unrelated UMN or personal
repositories into the migration. Organization policy rollout beyond that set
requires an explicit inventory/scope decision, never guessed repository names.

The roster's `deferred-todo` entries (`foss-earth/search`, `foss-earth/vr` and
`0sfs/vr`) stay as future plans. Do not provision or implement them, add them
to current release manifests, or treat their absence as a migration blocker.

## G0: inventory, recovery and writer assignments

1. Inventory each current checkout: actual root, branch, HEAD, remotes, refs,
   tags, worktrees, dirty tracked/untracked work, dependencies, assets and
   deployment entry points. Record pre-existing changes without publishing
   private working material. The source snapshot must distinguish committed
   inputs from user edits; never silently omit or absorb those edits.
2. Audit all refs selected for publication, using the platform validator or a
   transparent temporary read-only audit until it exists. Store exact affected
   commit IDs privately in migration evidence. Do not rely on the old counts
   in this conversation or the contributor UI.
3. Capture recoverable Git bundles/mirrors, release assets, issue/PR inventory,
   branch/ruleset settings, Pages/domain/DNS configuration, labels, webhooks,
   integrations and access requirements for affected remotes. Never place
   credentials, private metadata or backup refs in public implementation repos.
   Encrypted/secret settings need a restoration procedure; an API export does
   not imply their values can be recovered.
4. Record an immutable source snapshot and old-to-new mapping plan. Verify that
   a sample backup can restore refs and essential assets before relying on it.
   Preserve local `build/` and UMN `.local/` evidence separately; never delete
   or publish those trees as a shortcut.
5. Establish a task ledger with exactly one writer per checkout and target.
   Reserve shared test/build capacity and a release-integration queue. Domain
   writers own their existing app manifests/barrels/lockfiles; the coordinator
   owns remote administration and the ledger.

Keep public migration notes with the coordination proposal while appropriate;
keep operational backups/private inventories in permitted ignored storage.
When the flight writer is modifying this checkout, hand proposed tracked
ledger/index edits to that writer instead of concurrently editing them.

The ledger must name actual source refs, writer, destination remote, local
checkout path, visibility, license/provenance, package/artifact identity,
dependency gate, required checks, history status, candidate revision and
consumer adoption status for every target. Treat `agent-assignments.json` as
the initial roster, not a live status database or deployment manifest.

The [copy-ready launch prompts](launch-prompts.md) establish the initial
separate-session channel: you alone write this checkout's ignored
`build/repository-split/coordination.md`; each role writes its own named handoff
file in its permitted owner scratch directory. Record these locations and
exclusive assignments before implementation writes. Other agents can inspect
while the ledger is being prepared. Preserve existing records and state any
later agreed channel there; do not assume separately launched sessions share
collaboration-tool mailboxes. Public migration receipts remain separate from
this private operational ledger.

## Provisioning and common package contracts

Choose the new workspace root explicitly before cloning or creating modules.
Never assume the current `gh/0sfs` or `gh/foss-earth` directories can be replaced
by organization directories. An ignored, backed-up migration workspace may
hold independent new checkouts during transition; its location must not leak
into published package dependencies or become the only copy of source.

Create destination repos only for the reviewed roster and useful first slices.
Resolve existing remote-name collisions by inspecting ownership/content; never
overwrite or delete an existing remote to make provisioning convenient. Record
visibility explicitly; do not expose private source/metadata during extraction.
Do not create empty research/catalog/media repos solely to satisfy a graph node.
Resolve those nodes to a populated repo or a documented retained owner.

Freeze a package map: GitHub slug, package name, export surface, release method,
exact dependency version/artifact pin, license, assets, supported runtime and
build instructions. GitHub names such as `F-35B` or `360` are not necessarily
the eventual npm names. Use one consistent consumption mechanism; a local
`file:` link is useful for development but cannot be the release's only source.
Publishing to a package registry is a separate concrete release action; Git
and immutable package artifacts are sufficient while that decision is open.

Have platform provide policy/scaffold templates and a pinned validator before
ordinary new public repositories accept code. Bootstrap `foss-earth/ci` itself
by locally auditing the complete initial candidate history and reviewing its
validator/fixtures, publishing only that reviewed ref with writes restricted
to the coordinator, then enabling and testing its own enforcement before
normal contributions. Record that initial audit and immutable validator pin;
the bootstrap is not an ongoing exemption. Each primary agent installs its
own explicit caller/check configuration. Organization defaults alone do not
make CI execute. Configure required checks/rulesets through the coordinator and
verify the permissions/plan actually support the intended enforcement.

Maintain an attribution-enforcement adoption matrix for every real in-scope
repository, including existing owned dependency forks, code/data/docs/research,
logo, installer, CI and `.github` repos. Record the exact policy and caller pin,
shared consumer-conformance result, actual required-check identity, configured
merge/push/tag admission paths and bypass settings, full published-ref audit
and evidence location. Verify the remote settings for each repo; a green test
in the shared validator repo does not establish consumer enforcement. Checks
must run on documentation/content-only changes without path-filter skips.

Assign a sole writer for policy adoption in existing JSBSim/gamepad-tools forks
even when no API change needs a support agent. Respect upstream governance and
preserve human ancestry; do not send this app policy upstream or waive dirty
history. An adopted dependency still awaiting cleanup/enforcement stays an
explicit incomplete gate. Deferred repos acquire these requirements when
created and remain outside today's provisioning scope.

## Parallel extraction and handoffs

Use G1–G4 from the [index](README.md). Record public contract revisions before
consumers adopt them. Never resolve a dependency deadlock by copying upstream
code downstream or adding a reverse import. Forward concrete defects to the
native/device support agent only when needed.

For each child-repo assignment, fill the [component brief](component-agent.md)
with immutable input snapshots, exact allowed destinations, forbidden shared
files, prerequisite pins and acceptance tests. The old monolith's primary
writer removes/re-exports source and updates its consumer once the child
release is accepted. A package can have multiple entry points without forming
a package dependency cycle through its consumers.

Track candidates and adoption separately. Require archive contents/asset
verification and a clean install with local source links disabled. Have each
primary test the exact dependency artifacts it will release, not an unrecorded
neighboring worktree. Maintain public compatibility exports until every known
consumer has migrated or the coordinator records an explicit breaking release.

Keep an exception ledger for source assets, upstream history, incomplete
features, baseline test failures and optional repos. A waiver cannot silently
turn a failed attribution audit into a successful required check. A known
baseline failure remains visible and does not qualify the affected behavior.

## Historical attribution cleanup

New work follows the policy immediately. Legacy histories with prohibited
metadata will fail a strict full-history audit until their clean publication
is prepared. Label that state honestly; do not add a hidden skip range or an
allowlist of AI identities to get a green check.

Prepare filtered/sanitized histories in isolated migration copies from recorded
snapshots. Preserve legitimate human authorship, committer provenance as far as
the rewrite permits, license notices and asset rights. Remove prohibited
identity-bearing trailers; do not substitute invented humans for AI-primary
author/committer identities. Investigate provenance and resolve or exclude those
commits before publishing. A mention of an AI tool in ordinary prose is not an
attribution violation.

New component repos may receive audited extracted histories before final
application cutover. Re-audit every selected branch/tag, and publish only the
reviewed ref list. Never push backup, old-source or quarantined refs. Keep a
mapping of original revisions to cleaned/extracted revisions for evidence,
links and package-source metadata. Existing upstream JSBSim ancestry and open
upstream PR branches follow their own policy; do not rewrite unrelated upstream
history or send app-governance changes upstream.

At final application cutover, freeze affected writers and record their final
heads. Incorporate any newer work explicitly into the clean histories, rerun
the full ref audit and reconcile the mapping. Do not rewrite an active branch
under another agent or merge an old clone back into a cleaned repository.

## Rehearsal and optional repository recreation

Before any deletion/recreation, prepare an exact repository list with current
and intended names, URLs, refs, preserved assets/metadata, restoration steps,
unrestorable state and a rollback path. Dry-run recreation/restoration against
an isolated rehearsal target when possible. The contributor UI may be cached;
do not assert that a deletion/recreation or clean push guarantees immediate UI
removal. Use audited refs as the source of truth and verify the actual UI
separately.

Repository recreation is a destructive, separate operation. Use existing
explicit authorization for its concrete targets if present. If the exact
targets and restoration consequences have not been authorized, finish the
reviewable plan first and ask once before that operation. This execution-spec
writing request itself does not authorize deleting every organization repo.
Normal preparation, reversible provisioning and component work can continue.

After a selected recreation, restore access, rules, CI, Pages/domain settings,
release artifacts and supported metadata before declaring it canonical.
Reconfigure integrations needing new repository identity/keys through approved
channels without logging secrets. Old URLs and PR/issue identities may not be
preservable by simple re-creation; report the actual restoration result.

Production deployment is distinct from source migration. Stage and validate
the assembled release first; publish only the intended app/content/site scope
under the session's deployment authorization.

## Completion receipt

Finish with one receipt per organization/application, listing canonical repos,
accepted revisions/artifacts, retained/optional nodes, clean-ref audit results,
required-check/ruleset verification, tests and consumer adoption, restored
service configuration, new-clone/onboarding results and outstanding limitations.
Every in-scope target must be implemented, explicitly retained in an owner, or
clearly blocked with evidence. Deferred Search/VR targets remain recorded TODOs
and are excluded from completion gates. A blocked required target still means
the migration is incomplete. Preserve recoverable backups and native artifacts.

Verify that a fresh clone of each app builds with exact dependencies, About
reports those installed versions, no Earth/UMN setup needs a flight checkout,
no stale push route can reintroduce dirty history through a normal merge, and
the three primary agents have released their writer assignments. The process
is not complete merely because many new remotes exist.

Require a new-developer bootstrap receipt for each installer repository:
`foss-earth/dev_installer`, `0sfs/dev_installer` and `UMN-VR/dev_installer`.
Each must reach its full default development workspace and support explicit
component selection without requiring a pre-existing application checkout.
Flight and campus wrappers must use the accepted shared installer release.
