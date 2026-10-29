# Historical documentation

Dated records, completed agent prompts and superseded specifications. They are
kept so a clone can reconstruct what was true at the time. **Do not follow them
as current procedure.**

This folder only contains the two notes listed at the bottom. The JSBSim
centralization records and the SF50 handoffs that an older index placed here
now live in the research and SF-50 repositories.

## Current sources

- Agent rules for the flight app: [AGENTS.md](https://github.com/0SFS/fly/blob/main/AGENTS.md)
- JSBSim in the browser: [docs/jsbsim.md](https://github.com/0SFS/fly/blob/main/docs/jsbsim.md)
- Upstream policy: [jsbsim-upstream-contribution-policy.md](https://github.com/0SFS/research/blob/main/jsbsim-upstream-contribution-policy.md)
- Open PR review (2026-09-14): [validation/jsbsim-open-pr-review-2026-09-14.md](https://github.com/0SFS/research/blob/main/validation/jsbsim-open-pr-review-2026-09-14.md)
- Adoption, parity and rollback evidence: [validation/evidence/jsbsim/](https://github.com/0SFS/research/tree/main/validation/evidence/jsbsim)
- SF50 calibration notes: [SF-50/docs/validation/](https://github.com/0SFS/SF-50/tree/main/docs/validation). That directory holds the qualification record, the cruise fuel-flow diagnostic, the AFM audit and the pilot test card, among the other SF50 validation notes. It is not `docs/validation/` in this repository. This repository's [validation/](../validation/) is the org layout and the aircraft-health acceptance record.

| File | Where it is |
| --- | --- |
| [jsbsim-dependency-centralization.md](https://github.com/0SFS/research/blob/main/old/jsbsim-dependency-centralization.md) | [research/old/](https://github.com/0SFS/research/tree/main/old). Implemented two-repo centralization spec. |
| [jsbsim-centralization-2026-09-13.md](https://github.com/0SFS/research/blob/main/old/jsbsim-centralization-2026-09-13.md) | [research/old/](https://github.com/0SFS/research/tree/main/old). fork.1 acceptance / two-repo layout. |
| [jsbsim-in-tree-integration-2026-09-13.md](https://github.com/0SFS/research/blob/main/old/jsbsim-in-tree-integration-2026-09-13.md) | [research/old/](https://github.com/0SFS/research/tree/main/old). SDK-into-native import; fork.3–.4 identities. |
| [jsbsim-upstream-submissions-2026-09-13.md](https://github.com/0SFS/research/blob/main/old/jsbsim-upstream-submissions-2026-09-13.md) | [research/old/](https://github.com/0SFS/research/tree/main/old). Exact PR heads and checks on that date. |
| [jsbsim-pr1504-review-2026-09-13.md](https://github.com/0SFS/research/blob/main/old/jsbsim-pr1504-review-2026-09-13.md) | [research/old/](https://github.com/0SFS/research/tree/main/old). #1504 clarification and roadmap note. |
| [jsbsim-open-prs-brainstorm-prompt.md](https://github.com/0SFS/research/blob/main/old/jsbsim-open-prs-brainstorm-prompt.md) | [research/old/](https://github.com/0SFS/research/tree/main/old). One-shot prompt from that review. |
| [sf50-development-handoff.md](https://github.com/0SFS/SF-50/blob/main/docs/old/sf50-development-handoff.md) | [SF-50/docs/old/](https://github.com/0SFS/SF-50/tree/main/docs/old). Snapshot handoff (wrong starting checkout if used today). |
| [sf50-flight-model.md](https://github.com/0SFS/SF-50/blob/main/docs/old/sf50-flight-model.md) | [SF-50/docs/old/](https://github.com/0SFS/SF-50/tree/main/docs/old). Superseded by [sf50-flight-model-v2.md](https://github.com/0SFS/SF-50/blob/main/docs/proposals/sf50-flight-model-v2.md). |
| [sf50-resume-work-app-prompt.md](https://github.com/0SFS/SF-50/blob/main/docs/old/sf50-resume-work-app-prompt.md) | [SF-50/docs/old/](https://github.com/0SFS/SF-50/tree/main/docs/old). One-shot resume prompt. |

Named SF50 validation files:

| Topic | File |
| --- | --- |
| Qualification | [sf50-calibration-qualification-2026-09-12.md](https://github.com/0SFS/SF-50/blob/main/docs/validation/sf50-calibration-qualification-2026-09-12.md) |
| Cruise fuel flow | [sf50-g1-cruise-fuel-flow-2026-09-13.md](https://github.com/0SFS/SF-50/blob/main/docs/validation/sf50-g1-cruise-fuel-flow-2026-09-13.md) |
| AFM audit | [sf50-afm-audit.md](https://github.com/0SFS/SF-50/blob/main/docs/validation/sf50-afm-audit.md) |
| Pilot test card | [sf50-pilot-test-card.md](https://github.com/0SFS/SF-50/blob/main/docs/validation/sf50-pilot-test-card.md) |

## Still in this folder

| File | Why it is here |
| --- | --- |
| [project-origin-spark-2026-09-19.md](project-origin-spark-2026-09-19.md) | Origin spark: JSBSim discussion #984 plus the WASM-meets-FOSS-Earth idea |
| [docs-refresh-after-controller-work-prompt.md](docs-refresh-after-controller-work-prompt.md) | One-shot work-app prompt from a completed session |
