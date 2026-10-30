# Documentation

Catalog of this repository. Pages are sorted the way the shared convention describes: [documentation layout](https://github.com/foss-earth/docs/blob/main/docs/documentation-layout.md). Current pages say how things work today. Proposals are designs that are not shipped, or that stay the design record after something shipped. Archive is session notes and finished procedures. Do not follow the archive as current procedure.

`docs/build/` is a generated publish mirror and is not listed here.

## Current

[Modding](modding.md) explains how to find and install mods, choose discovery
topics, and publish aircraft, engines, components, payloads, plugins and modpacks.

The monolith split is published. [repository-split.md](repository-split.md) is where things stand and what is left. [parity-checklist.md](parity-checklist.md) is the human check against the old app. Open follow-up is in [TODO.md](TODO.md), the working backlog (split leftovers, performance, flight model, map, collision, debug views, aircraft and branding, plus one FOSS Earth tooltip note).

[Choosing the useful peaks of realism and cost](realism-cost.md) explains why we compare multiple simulation implementations and how we retain the choices that justify their resource and maintenance costs.

[software-dependency-graph.md](software-dependency-graph.md) separates distributable software from live services and creative assets, and records the installed JSBSim fork.7 package.

The [resource administration reference](https://github.com/0SFS/fly/blob/main/docs/resources/README.md)
documents the flight app end to end: main-thread execution, frame times, memory,
worker CPU, GPU, network, persistent storage and audio. Each guide links the
owning implementation and explains controls, limits, cleanup, observability and
efficiency tradeoffs. Its source audit is dated 29 October 2026; historical
measurements remain linked in their original records.

[AGENTS.md](AGENTS.md) points at the shared agent rules in `0SFS/.github`.

### Validation

[validation/layout.md](validation/layout.md) is where specifications, retained measurements, runnable tools and scratch belong.

The [local acceptance record](validation/aircraft-health-2026-10-25.md) covers health, synchronized model selection, host permissions and review instructions.

[validation/README.md](validation/README.md) indexes a retained evidence tree. Those measurements moved on 2026-09-19. What remains under [validation/evidence/](validation/evidence/) is the relocation receipt, [2026-09-19-docs-evidence.json](validation/evidence/relocations/2026-09-19-docs-evidence.json). The move is written up in [evidence-relocation-2026-09-19.md](validation/evidence-relocation-2026-09-19.md). [relocate-evidence.md](validation/relocate-evidence.md) is the handoff for that move, kept as written; it was completed the same day. JSBSim adoption, parity and rollback records now cited from research are in [research/validation/evidence/jsbsim](https://github.com/0SFS/research/tree/main/validation/evidence/jsbsim).

### Release

- [release/RELEASE_CHECKLIST.md](release/RELEASE_CHECKLIST.md) — gate list for a tagged release. The 2026-09-09 lint and audit rows are marked done; the rest is still open.
- [release/RELEASE_AUDIT.md](release/RELEASE_AUDIT.md) — read-only audit on 2026-09-09, when this tree was still named `flight-sim`. It is not legal advice and it is not the post-split state.
- [release-notifications.md](release-notifications.md) — announcement drafts. Bracketed repository, tag and playable URLs are still placeholders.

## Proposals

Open designs in [proposals/](proposals/). The [implementation/](proposals/implementation/) briefs are historical and listed under Archive.

- [multiplayer.md](proposals/multiplayer.md) — draft, 2026-10-22, for two or more pilots in the same sky. The first mode is a serverless WebRTC flight. The flight app already has a two-pilot Host/Join milestone; a four-pilot mesh, shared weather, voice and a hosted server are later.
- [vr.md](proposals/vr.md) — 0SFS flies in a headset from its VR tab through FOSS Earth's immersive package. Tested on an emulated headset, not yet on a real one. The `0sfs/vr` repository is still to come.
- [github-modding.md](proposals/github-modding.md) — specification and implementation plan, 27 October 2026, for publishing aircraft, engines and stores as GitHub content.
- [tab-inventory.md](proposals/tab-inventory.md) — 2026-10-08 audit of every tab in 0SFS, FOSS Earth and the UMN tour, with the repository each tab was proposed to live in. Written before the split was published; the page still calls those destinations proposals.
- [repository-split.md](proposals/repository-split.md) — the proposal the split carried out on 2026-10-12. The file stays as written on 2026-10-08 and still speaks as if the move had not happened. The published record is [repository-split.md](repository-split.md).
- [aircraft-damage.md](proposals/aircraft-damage.md) — records the implemented whole-aircraft health model and the candidate cost/realism tradeoffs. Off and whole-aircraft health are in the local checkout. Component damage and physical damage below are still proposals.
- [aircraft-damage-physical.md](proposals/aircraft-damage-physical.md) — proposed design that does not exist yet. It covers impacts, systems, structural failure and JSBSim coupling. No F-35B damage characteristics, material thresholds or CPU budgets in it are claimed measured.
- [downloadable-app.md](proposals/downloadable-app.md) — collects what only a downloadable app could do, such as turning and zooming at once on a Mac trackpad. No app is planned.
- [spatial-upscaling-and-dynamic-resolution.md](proposals/spatial-upscaling-and-dynamic-resolution.md) — records why neural frame generation (`RIFE 4.25.lite`) and neural super-resolution (`RT4KSR x2`) stopped at Stage 1/2, why they do not fit `0sfs`'s main-thread CPU profile and flight-control latency constraints, and specifies the `@foss-earth/renderer` implementation of analytical spatial upscaling (`P9`, EASU + RCAS) and dynamic resolution scaling (`P8`).
- [SPEC-360-DRONE.md](proposals/SPEC-360-DRONE.md) — implementation specification for a removable 360° drone mod. A fresh flight does not download it until the pilot adds the mod.
- [360-drone-flight-model-roadmap-2026-10-28.md](proposals/360-drone-flight-model-roadmap-2026-10-28.md) — the drones' flight-model plan and its status. The rotors are simulated down to the spinning propeller by [`multirotor/`](../multirotor/README.md), without motors, ESCs or battery. Each stage says what was built, deferred or dropped. The [realism audit](proposals/360-drone-realism-audit-2026-10-28.md) it answers sits beside it.
- [jit-code-download.md](proposals/jit-code-download.md) — pointer kept so existing links still open. The specification is [in the FOSS Earth docs](https://github.com/foss-earth/docs/blob/main/docs/proposals/jit-code-download.md), not in this file.

## Archive

Not current procedure.

### Split implementation briefs

[proposals/implementation/](proposals/implementation/) is the 2026-10-08 launch kit. The split is published; these briefs are historical launch notes. The [index](proposals/implementation/README.md) says so at the top and still links the roles from that date, including platform, Earth and campus briefs that live in the other organizations.

- [launch-prompts.md](proposals/implementation/launch-prompts.md) — copy-and-paste prompts from 2026-10-08.
- [coordinator-agent.md](proposals/implementation/coordinator-agent.md) — coordinator assignment.
- [flight-agent.md](proposals/implementation/flight-agent.md) — flight extraction assignment for the then-current checkout.
- [native-agent.md](proposals/implementation/native-agent.md) — conditional JSBSim SDK support, only for a concrete contract defect.
- [device-agent.md](proposals/implementation/device-agent.md) — conditional gamepad-tools support, only for a concrete export defect.
- [component-agent.md](proposals/implementation/component-agent.md) — later per-repository assignment template.
- [agent-assignments.json](proposals/implementation/agent-assignments.json) — starting work roster from that launch. Not proof a remote exists.

### Brainstorm and session prompts

- [brainstorm/next-session-prompt.md](brainstorm/next-session-prompt.md) — continuation prompt for iPhone phone-controller defects. The session notes it cites are in the [rc repository](https://github.com/0SFS/rc).
- [proposals/360-drone-session-2026-10-28.md](proposals/360-drone-session-2026-10-28.md) — 28 October 2026 handoff for the 360° drone mods. The specification stays [SPEC-360-DRONE.md](proposals/SPEC-360-DRONE.md). This note is what that session learned and what is still open, including the resume prompt.
- [theme-support-prompt.md](theme-support-prompt.md) — one-shot prompt for light and dark theme support.
- [agent-documentation-optimization-report.md](agent-documentation-optimization-report.md) — 22 October 2026 audit of agent documentation across 0SFS, FOSS Earth and UMN-VR.

### Historical notes

[old/](old/) keeps dated records that used to sit with this repository. [old/README.md](old/README.md) names where each one actually is now. In this folder:

- [project-origin-spark-2026-09-19.md](old/project-origin-spark-2026-09-19.md) — origin spark: JSBSim discussion #984 and the WASM-meets-FOSS-Earth idea. Not procedure.
- [docs-refresh-after-controller-work-prompt.md](old/docs-refresh-after-controller-work-prompt.md) — one-shot prompt from a completed session. Its checkout paths predate the split.

### Scratch

- [RENAME_PLAN.md](RENAME_PLAN.md) — record of the rename from `flight-sim` to 0SFS. The rename is done.
- [build-scratch.md](build-scratch.md) — inventory, as of 2026-09-25, of large gitignored `build/` folders, plus a recipe for committing that page while other sessions edited it.
- [song.md](song.md) — cue sheet for a promo soundtrack.

## Community

- [Code of conduct](https://github.com/0SFS/.github/blob/main/CODE_OF_CONDUCT.md)
- [Contributing](https://github.com/0SFS/.github/blob/main/CONTRIBUTING.md)
- [License](LICENSE)
- [Security](https://github.com/0SFS/.github/blob/main/SECURITY.md)
