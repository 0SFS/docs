# Choosing the useful peaks of realism and cost

Design guidance, 2026-10-21. Applies to simulation models and their runtime implementations. The [G-response proposal](../fly/docs/proposals/g-vision-response-curves.md) is a worked application; its budgets and model rankings remain provisional.

Build a few meaningfully different implementations, compare their behavior and resource use, and retain the points where additional work buys a useful increase in realism. Prefer the most credible behavior that fits the available budget. Keep a more expensive implementation when its added capability justifies its cost, and a cheaper one when it serves a real workload or device constraint.

We call these points the **useful peaks of realism/cost**. More precisely, they are useful choices on a Pareto frontier, often near its knees: places where the next increase in cost yields much less improvement. There may be several such choices, or just one.

## Why create multiple implementations?

A simulator has to choose which parts of reality to represent. One implementation might reproduce a steady value accurately but miss a transient. Another might preserve history and explain the transient with only a little more state. A third might reproduce a reference model closely while adding little visible or measurable benefit in ordinary flight.

Implementing a small set of alternatives lets us test those possibilities. It gives us:

- **A measured reason to spend resources.** We can identify which added mechanism changes an outcome, and whether the improvement survives validation.
- **Useful choices for different workloads.** A method suitable for one pilot may be unsuitable for thousands of contacts or sound sources. CPU, memory and latency limits can favor different implementations.
- **A reference for cheaper reductions.** A detailed implementation can generate comparison traces or reveal a failure in an approximation, even if it never ships in the runtime.
- **A way to challenge assumptions.** A simpler model can outperform a more elaborate one on independent evidence. Comparing both helps expose wrong equations, weak parameters and unnecessary numerical work.

The purpose is to learn which implementations earn a place in the system. Creating alternatives does not commit us to maintaining or exposing all of them forever. Prototype the computational core first; build full product controls for the retained choices.

## What counts as realism?

Define realism through the behavior the feature is meant to reproduce. Examples include contact timing and location, a response to a changing control input, engine transients, or the onset and recovery of pilot impairment. Measure those outcomes under specified conditions.

Keep several dimensions visible:

| Dimension | What to establish |
| --- | --- |
| Physical fidelity | Agreement with suitable measurements, including transient and history-dependent behavior |
| Coverage | Which conditions, configurations and phenomena the model supports |
| Numerical fidelity | Integration/interpolation error and stability for the chosen discretization |
| Perceptual usefulness | Whether an improvement affects something the person can see, hear or control |
| Confidence | Quality of the evidence, calibration and independent validation |

More equations, points, triangles, samples or substeps describe work and representation. They do not establish these outcomes by themselves. Likewise, reproducing a complex reference establishes agreement with that reference; agreement with reality needs separate evidence.

Set mandatory correctness requirements before trading accuracy for cost. For example, finite state, correct units, deterministic time accounting and required collision coverage may be invariants at every budget. When a budget cannot resolve an answer, report the permitted approximation or unresolved result explicitly. A lower budget must not silently change the meaning of a successful result.

## What counts as cost?

Treat cost as a set of quantities rather than one universal score:

- CPU work and elapsed time per simulation step/query, its update rate and instance count; include tail latency and catch-up bursts.
- GPU work, synchronization and transfer cost where relevant. Measure these separately from CPU submission time.
- Resident and peak memory, shared data versus per-instance state, allocations and garbage collection.
- Startup, asset preparation, compilation, calibration and download/storage costs.
- Engineering effort: implementation, validation, maintenance and the risk of inconsistent behavior between variants.

Keep offline and runtime budgets separate. More preparation may produce a better cheap runtime model. A lookup table can save computation while spending memory; it needs an error bound and a measured benefit. An asynchronous backend can improve throughput while worsening response latency. Name the constraint that matters.

Allocate a feature's budget from the whole application's budget. Express it in real units and at a declared workload: for example, microseconds per accepted physics step for one active pilot at the selected update rate, plus retained bytes. Include adapters, native reads, transfers and scheduling. An isolated solver benchmark does not establish the integration cost.

## Choose physics frequency separately from render FPS

The requested physics-rate policy is **60 Hz by default, 30 Hz as a lower-cost option to validate, and 120 Hz only by explicit opt-in**. fly implements it as the `Physics [30 / 60 / 120 Hz]` setting ([physicsRate.ts](../fly/src/flight/physics/physicsRate.ts)), applied on the next flight. 30 Hz has not passed: a C172 at rest keeps bouncing on its gear, and a quick pull peaks about 40 ms late ([physicsRate.integration.test.ts](../fly/src/flight/physics/physicsRate.integration.test.ts)). Consider 30 Hz as a future default only if the relevant aircraft, controls and ground-contact tests demonstrate that its tradeoff is acceptable. Do not silently select 120 Hz through a quality preset, device heuristic or fallback.

Physics Hz counts integration steps per simulated second. Render FPS counts displayed frames per wall-clock second. Keep the physics timestep fixed for a selected rate and interpolate the display independently. A 60 Hz physics setting must not cap rendering at 60 FPS or make simulation speed depend on display refresh.

Rate is another candidate dimension in the realism/cost comparison. Evaluate 30, 60 and 120 Hz using matched physical scenarios and real elapsed simulated time. Report trajectory/control error, input latency, ground-contact stability, event timing, missed short loads and CPU per simulated second. Keep render-rate independence tests separate from physics-rate convergence tests; changing integration frequency need not produce bitwise-identical trajectories.

Use one timing configuration for native integration, the accumulator and every consumer of accepted-step duration. Update controllers, wheel/contact feedback, pilot physiology and time accounting consistently. Apply a rate change on initialization/reset, and keep backlog limits in simulated-time units rather than accidentally multiplying catch-up time by retaining a fixed step count. Never advance a 120 Hz native integrator only 60 times/s and label the result 60 Hz real-time physics.

Allow a subsystem to use finer internal steps when evidence shows it needs them, and record that work explicitly. A solver with a fixed 1 ms internal step can retain roughly the same work per simulated second even when the outer physics rate falls. Repeating the entire native aircraft simulation at 120 Hz behind a default 60 Hz label would defeat the requested choice. If lower rates fail validation, document the failing cases and the targeted remedy instead of concealing the cost.

The per-step cost multiplied by the number of steps gives a first budget estimate. It does not promise a proportional FPS improvement: measure the integrated application and its actual bottleneck. Default to the lowest validated rate that meets the required behavior within the requested 30/60 Hz range, with 60 Hz as the initial implementation target.

## Find the frontier, then choose the useful knees

Suppose two candidates support the same required conditions. One dominates the other when it is no worse in the relevant accuracy/capability dimensions and costs no more in the constrained resources, with at least one meaningful improvement. Account for measurement noise and uncertainty before declaring dominance.

The remaining candidates form a frontier among the implementations evaluated; this does not prove that no better method exists. A knee is a region where further spending starts to have sharply diminishing returns. We are interested in those regions because they make useful product choices: a cheap credible model, perhaps a model that adds an important mechanism, and occasionally a detailed model whose last improvement matters to a particular workload.

Avoid reducing the decision to a raw `realism / cost` ratio. Realism has no universal numeric unit, and an extremely cheap model that misses a required phenomenon is not a good choice. Use explicit budgets, acceptance limits and incremental gains instead. A visually prominent bend in a chart is also insufficient without useful supported behavior.

Use this workflow:

1. **Define outcomes and invariants.** Identify important steady, transient, boundary and recovery cases. State the supported configurations and what evidence can validate each one.
2. **Propose a few distinct candidates.** Start with the current baseline, a small credible alternative, and a reference or more explanatory model. Add an intermediate candidate when there is a concrete mechanism or cost-saving hypothesis to test.
3. **Set provisional envelopes.** State CPU, memory and latency ceilings for the actual workload. Mark estimates and targets explicitly. Do not label proposed budgets as measured peaks.
4. **Compare fairly.** Use the same inputs, initial conditions, accepted simulation time, event definitions and output adapter. Separate input-sampling improvements from equation changes. Fit parameters on one set of protocols and validate on independent ones where possible.
5. **Measure the implementation.** Use paired/interleaved runs, warmup and consumed results. Record the machine, runtime, workload, variation, state/data bytes and adapter cost. Begin with a short local run; move expensive sweeps to an explicit longer-run handoff.
6. **Inspect each behavior family.** Report units, uncertainty, missed events and false events. Averages must not hide a serious regression. Missing validation coverage is a gap, not a passing score.
7. **Select and simplify.** Remove dominated candidates. Retain a higher-cost choice when it adds a meaningful validated capability or improvement that fits a real budget. Collapse tiers when the better model fits the cheaper envelope.

Choose an improvement threshold appropriate to the evidence and the feature. There is no universal percentage that establishes realism. When data cannot distinguish candidates, say so and prefer the simpler implementation unless another demonstrated requirement favors the more elaborate one.

Plot measured error or an explicitly defined quality measure against CPU, with memory and coverage labels. Keep separate charts or tables when a single score would conceal important tradeoffs. Revisit the choice when workloads, hardware, models or evidence change.

## Multiple implementations of what?

Separate these axes so their effects can be evaluated independently:

| Axis | Example difference | Question |
| --- | --- | --- |
| Physical model | Generic lag versus supply/reserve dynamics | Does representing another mechanism improve predictions? |
| Numerical method/detail | Step size, solver or spatial refinement | Does resolving the same model more accurately matter? |
| Representation/backend | Analytic expression versus compiled table; TS versus WASM | Can the same behavior be delivered more cheaply? |

A faster backend is a performance comparison until behavior differs. A finer solver can accurately solve a poor physical model. Mixing these axes in one change makes the reason for an improvement hard to establish.

Use one shared input/output contract, declared units, reset/time semantics and diagnostic conventions. Keep each model's internal state private and its assumptions explicit. Share settings infrastructure, fixtures and presentation instead of duplicating the whole feature. Do not force every physical model to interpret the same parameter as the same thing when its meaning differs.

Preserve meaningful differences in the tests: equivalence tests suit alternate backends for the same model; distinct physical models need shared invariants plus evidence-based outcome tests. They should not be forced to reproduce each other's known approximations.

Keep production selection stable for stateful models. An automatic downgrade is appropriate only when there is a defined state transfer or error-controlled refinement policy. Otherwise choose the model at initialization or an explicit reset. Changing model identity in response to one slow frame can create a discontinuity unrelated to the simulated event.

## Worked applications

The [G-response alternatives](../fly/docs/proposals/g-vision-response-curves.md) propose a Curve control, an Exposure model with retained debt/recovery, a Perfusion model with pressure/flow and reserves, and a Reference implementation for comparison. The proposed 10 / 50 / 250 µs envelopes are experiments. They do not establish that each model consumes that budget or deserves a separate runtime tier. Their value must come from temporal outcomes and evidence.

The [collision geometry comparison](https://github.com/0SFS/ground-contact/blob/main/docs/validation/collision-geometry-cost-quality.md#compute-budget-ladder) applies the same idea to competing bounds, fitted parts and source-triangle refinement. An exact cheap path for a supported case can belong at every budget. A higher primitive count is not a universal measure of cost or complete coverage. Representation error, temporal error, resident memory and offline fitting effort remain separate.

These examples illustrate why the architecture supports several candidates while the product exposes only useful retained choices. A reference can stay in tooling; a special-case fast path can be shared by every implementation.

## Record the decision and its limits

Each feature's comparison should leave a short decision record with:

- The question, required behaviors and why alternatives were worth implementing.
- A candidate table: assumptions, calibration/validation coverage, error by behavior family, measured CPU/latency/memory, and preparation cost.
- Reproduction commands, versions, workload, evidence links and measurement uncertainty.
- The retained choices and why each adds value; rejected candidates and the reason for rejection.
- The default, selection/reset policy, supported parameter ranges, known gaps and what would justify reopening the decision.

Distinguish **proposed**, **implemented**, **measured** and **validated**. Source fidelity, numerical convergence and human/physical validation are separate claims. Keep runnable tools and selected evidence with the owning feature, following the [validation layout](validation/layout.md); scratch runs belong in ignored `build/` directories.

Stop adding variants when the next candidate has no concrete improvement hypothesis, when gains fall below useful evidence/measurement resolution, or when maintenance costs outweigh the capability it adds. The desired result is a small, justified set of implementations and a clear explanation of the resources each one earns.
