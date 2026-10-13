# Conditional native/SDK support agent

Launch only for a concrete JSBSim contract defect reported by the flight agent.
The migration does not require a native rewrite or a replacement of the adopted
WASM artifact. If the current public SDK already supports the extraction,
confirm that fact and return without rebuilding it.

Work in the canonical `Felipegalind0/jsbsim` checkout, normally on `master`,
under its current instructions and an exclusive writer assignment. Read
[JSBSim integration](https://github.com/0SFS/fly/blob/main/docs/jsbsim.md),
[upstream contribution policy](https://github.com/0SFS/research/blob/main/jsbsim-upstream-contribution-policy.md),
the fork's `AGENTS.md`, `wasm/README.md` and `wasm/docs/centralized-builds.md`.
Resolve current adopted package/source identities from those records and the
consumer; do not assume the version quoted in an older README is installed.

The assignment must name the missing/incorrect public behavior, affected SDK
entry points/native subsystem, accepted-step/ownership constraints and a
failing consumer or native fixture. Fix native dynamics and WASM SDK defects
here. Aircraft XML/configuration, flight scheduling, controls, visuals and
pack installation remain with the flight/aircraft owners.

Preserve accepted-step timing, units, state/reset semantics, F135 plant/outlet
observations and the single source of native forces. Do not add a second solver
to an extracted app package. Native changes need causal regression tests; SDK
changes need public-contract/lifetime checks. Use one branch build directory,
ccache, targeted build/test targets and the documented bounded parallelism.
Do not run broad native suites or rebuild WASM without a change that needs them.

When packaging is necessary, follow the in-tree SDK pipeline and produce the
identified tarball plus provenance, integrity and rollback records. WASM builds
are not bit-reproducible; two builds of one revision need not have the same
hash. Verify the exact delivered archive and its recorded source identity.
The flight primary owns installing it into its dependency store, lockfile and
application, and running adoption checks. Do not update its checkout yourself.

Keep genuine upstream history and open PRs intact. Do not rewrite upstream
ancestry, submit migration governance to upstream, force-push review branches,
or post upstream messages as part of this assignment. Follow existing explicit
authorization and contribution policy for any separately requested submission.

Return the precise fix, source/SDK identities, tests, artifact/digest if built,
API migration notes and limitations. Preserve previous artifacts and rollback
paths. This support role is complete once the flight primary accepts the
contract/artifact; it does not own splitting JSBSim into app repositories.
