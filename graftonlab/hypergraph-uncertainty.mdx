---
title: "Hypergraph Uncertainty"
description: "Clause-level uncertainty quantification, evidence fusion, and upward propagation in the reasoning hypergraph"
---

## Goal

Make the reasoning hypergraph *compute* credibility/uncertainty end-to-end:

* identify uncertainty per atomic claim (clause)
* fuse multimodal evidence (CAD now, FEA later, others anytime)
* propagate upward: **Evidence → Clause → Contract → Spec/Req/Goal → Intent/Summary**
* output: `P(pass)` + uncertainty width + top drivers, always traceable

The UQ output is treated as a decision interface: **probability + uncertainty + attribution**, used to gate, to choose next actions, and to generate auditable explanations

**Non-goal:** tool plumbing, storage formats, runtime optimizations, UI.

## Core idea

We break each requirement into atomic, testable **clauses** and attach uncertainty to each clause (and to shared numeric **parameters** like material strength or loads). Every tool/modality (CAD now, FEA later, others) produces **evidence** plus a small, explicit adapter that turns that evidence into a probabilistic update for the relevant clause(s), while missing or conflicting evidence is represented explicitly to widen epistemic uncertainty rather than being ignored or causing brittle flips. For any decision (contract/goal), we compile just the relevant subgraph and run inference to get clause posteriors `P(pass)` with uncertainty, then roll those up through **probability-model aggregators** (e.g., noisy-AND/OR with criticality and correlation control) to produce `P(contract/spec/goal pass)` + uncertainty width + the top drivers/unknowns, all traceable back to specific evidence and assumptions and used directly for gating and "what to do next" (VOI).

## Definitions used

### Glossary

* **V&V**: *Verification & Validation*.
* **Verification**: evidence the tool/model ran correctly (numerics, convergence, consistency, geometry validity).
* **Validation**: evidence the tool/model matches reality for this use case (comparison vs test/field/baseline).
* **UQ**: *Uncertainty Quantification* — represent + propagate uncertainty.
* **Clause**: atomic, testable claim; smallest unit we attach belief to.
* **Contract**: grouped clauses (assumptions + guarantees) with an explicit aggregation rule; yields `P(contract_pass)`.
* **Assumption-clause**: clause not directly specified; used to make evaluation possible; typically high epistemic.
* **Guarantee-clause**: clause that must hold to satisfy a requirement/contract.
* **Parameter (Θ)**: uncertain numeric input feeding one or more clauses (material, load, geometry tolerance).
* **QoI**: *Quantity of Interest* produced by analysis (stress, deflection, margin).
* **Evidence (Y)**: observed output from a modality/tool (measurement, check result, solver output, classifier score).
* **Tool invocation**: provenance anchor for evidence (tool/version/config/context).
* **Prior**: belief before new evidence.
* **Posterior**: belief after fusing all evidence.
* **Likelihood**: mapping from evidence to clause support, e.g. `P(Y | X, context)` (or `P(X | Y)` via calibration).
* **Calibration**: map tool scores to probabilities so predicted `P(pass)` matches observed frequency.
* **Credibility**: how strongly a node/evidence should influence decisions (V&V maturity, calibration quality, coverage).
* **Aleatory uncertainty**: inherent variability (material scatter, manufacturing variation).
* **Epistemic uncertainty**: lack of knowledge (missing views, unknown BCs, unvalidated model, spec gaps).
* **Missingness**: absent evidence (tool/view/backend/spec) treated as epistemic inflation, not silent skip.
* **Unknown node**: typed representation of missingness (e.g., `missing_view`, `tool_missing`, `backend_unavailable`).
* **Conflict**: evidence disagreement; should increase uncertainty unless strong credibility resolves it.
* **Robust fusion**: conflict-handling that limits impact of outliers/unreliable evidence (mixtures/reliability weights).
* **Correlation group**: label for shared failure modes/lineage so evidence isn't double-counted as independent.
* **Shared latent**: hidden variable capturing common cause (e.g., material property, solver bias, shared export bug).
* **Numerical error**: discretization/solver tolerance effects (mesh density, convergence, integration error).
* **Factor**: local function encoding uncertainty semantics for an edge (likelihood, coupling, missingness, aggregation).
* **Marginal**: per-node posterior, e.g. `P(X_i=pass | all evidence)`.
* **Noisy-AND / Noisy-OR**: soft logical aggregators for combining clause beliefs without brittleness.
* **Coverage**: how much of the requirement space is supported by evidence (views, regimes, load cases, geometry regions).
* **VOI**: *Value of Information* — heuristic to pick next work (often `impact × epistemic uncertainty`).

### Uncertainty vs credibility

* **Uncertainty**: "how wide could truth be?" (variance/interval/ignorance mass)
* **Credibility**: "how much should this node influence decisions?" (V&V maturity, calibration quality, coverage)

Keep separate: high credibility can still be uncertain (good model, limited data), low credibility can be overconfident (bad calibrator).

### Aleatory vs epistemic (must be explicit)

* **Aleatory**: inherent variability (material scatter, manufacturing tolerances)
* **Epistemic**: lack of knowledge (missing views, unknown BCs, unvalidated solver, assumption not grounded)

Propagation should *not* collapse these into one scalar; store both when possible.

### Verification vs validation (how evidence should behave)

* **Verification evidence**: "tool executed correctly" (numerics, convergence, geometric validity)
* **Validation evidence**: "model corresponds to reality" (comparison vs test/field/accepted baseline)

Verification tends to shrink "tool error"; validation tends to shrink "model-form error".

## Data model (conceptual)

Uncertainty lives on Clause + Parameter; Evidence updates Clause; Contract aggregates Clauses; higher nodes roll up posteriors.

### Node types

* **Clause**: atomic, testable claim. Examples:
  * `hole_diameter within 6.6mm ±0.1mm`
  * `von_mises_stress ≤ allowable_stress`
  * `deflection ≤ 1.0mm`
  * `moment_arm within 0.10m ±5%` (assumption-clause)
* **Contract**: container of clauses (guarantees + assumptions), with aggregation semantics.
* **Parameter**: numeric uncertain input (material, load location, torque, tolerance).
* **Evidence**: observation produced by a modality/tool (measurement, CAD check, solver output, classifier score).
* **Tool Invocation**: provenance anchor (tool id, version, config, runtime context).
* **Unknown**: typed missingness node (tool missing, view missing, backend unavailable, spec gap).

### Edge types (semantics-bearing)

* `EVIDENCES` / `VALIDATES`: Evidence → Clause (never only Contract; Contract fusion happens via clauses)
* `DEPENDS_ON`: Clause ↔ Parameter, or ToolInvocation ↔ environment assumptions
* `AGGREGATES`: Clause(s) → Contract
* `DERIVES_FROM`: Spec/Req/Goal ancestry (used for upward propagation summaries)

## Uncertainty representation (minimal, composable)

Represent each Clause with one of these, plus provenance:

### 1) Boolean clause belief

`X_i ∈ {pass, fail}` with:

* posterior probability `p_i = P(X_i=pass | evidence)`
* uncertainty measure: credible interval on `p_i` or entropy `H(p_i)`

Recommended priors:

* **Spec-derived**: informative (tight-ish)
* **Assumption-derived**: weak (wide, high epistemic)

Practical prior family:

* **Beta** prior on `p_i`: `p_i ~ Beta(α, β)` (encodes confidence + base rate)

### 2) Numeric quantity belief (parameter or QoI)

`Θ_j` as:

* parametric distribution (Normal/LogNormal/Triangular/etc), or
* interval with confidence level, or
* sample set (particles)

### 3) Ignorance-aware belief

Mass on `{pass, fail, unknown}` or "opinion-like" tuple:

* belief(pass), belief(fail), uncertainty(unknown)

Use when tools are uncalibrated or missingness/conflict dominates.

### Always stored alongside belief

* provenance: source span, tool/version/config, artifact hash
* *credibility tags*: calibration status, verification status, validation coverage class
* correlation group id: shared failure modes / shared assumptions

## Clause-first decomposition (why it matters)

Contract-level evidence attachment loses structure:

* cannot explain which guarantee is supported
* cannot combine CAD dimensional evidence with stress/deflection evidence
* cannot propagate correctly to Specs/Goals
* encourages "first contract only" verification behavior

Clause-level fixes this by making every evidence statement target a clause (or a parameter feeding multiple clauses).

## Modality adapters: convert outputs → likelihood over clauses

Every modality produces an Evidence node; the adapter defines a likelihood term: `L_k(X_i) = P(Y_k | X_i, context)` or equivalent soft score.

### CAD / geometry

Typical clause targets:

* dimensions (diameter, spacing, thickness)
* minimum edge distance
* topology validity (watertight, non-self-intersecting)
* manufacturability proxies (thin features, min radii)

Uncertainty sources:

* PMI/tolerance completeness (epistemic)
* STEP translation fidelity (epistemic)
* discretization/measurement resolution from geometry queries (aleatory-ish if quantized)

Adapter pattern:

* numeric measurement `m` with measurement noise `σ` → likelihood around tolerance band
* missing views/backend → "unknown evidence" that increases epistemic uncertainty, not auto-fail

### FEA (in the future)

Clause targets:

* stress margin: `σ_max ≤ σ_allow`
* deflection limit: `δ ≤ δ_max`
* fatigue, buckling, contact separation, etc.

Uncertainty sources:

* input distributions (tolerances, material, load placement)
* numerical error (mesh, solver tolerances)
* model discrepancy (BC idealization, linearization, contact simplifications)

Adapter outputs should be probability of satisfaction:

* `P(margin > 0)` plus uncertainty on that probability (sampling error, surrogate error)

### Other modalities (anytime)

* vision-based geometry check: classifier score → calibrated probability
* rule/SMT checks: satisfiable/unsat with caveats → contributes to structural consistency clauses
* LLM-extracted claims: low-credibility evidence unless grounded

## Fusion + propagation: factor-graph view compiled from hypergraph slice

Pick a target (Contract or Goal) and compile the reachable subgraph of: Clauses, Parameters, Evidence, ToolInvocations, Unknowns, dependencies.

### Random variables

* Clause truth: `X_i ∈ {0,1}`
* Parameters: `Θ_j` continuous
* Tool outputs: `Y_k` observed (score/value)
* Optional: reliability/outlier flags: `R_k`, `O_k`

### Factor types (local semantics)

#### A) Tool likelihood (Evidence → Clause)

Connects a clause to a tool observation:

* `φ_k(X_i, Y_k; calib_k, σ_k, bias_k)`

If tool emits scalar score `s`:

* map `s → P(X_i=1|s)` via calibration model
* uncertainty in calibration becomes epistemic mass

#### B) Physics/consistency (Parameters → Clause)

Clause depends on parameters through a (possibly implicit) margin:

* `margin_i = g_i(Θ, context)`
* soft truth: `P(X_i=1 | Θ) = sigmoid(margin_i / τ)` (τ encodes model-form/numerical softness)

This is where shared parameters create *coupling* across many clauses.

#### C) Correlation control (shared latent causes)

If multiple evidence items share a failure mode:

* connect them via shared latent `R_group` or shared parameter `Θ_shared`
* prevents double counting independent support when it's not independent

#### D) Robust conflict handling

Conflicting evidence should increase uncertainty, not flip-flop:

* mixture likelihood with outlier flag `O_k`
* or reliability-weighted likelihood with `R_k`

#### E) Contract aggregation (Clauses → Contract)

Contract pass variable `C` from guarantee clauses `{X_g}` and assumption clauses `{X_a}`.

Typical aggregator:

* **Noisy-AND** for required guarantees `P(C=1 | X_g) = leak * Π_i P_i` with per-clause weights/criticality (softens brittleness, supports partial coverage)
* Assumptions can gate validity (optional): if assumption low, shrink credibility or widen epistemic uncertainty rather than hard-fail.

### Inference output

Compute marginals:

* per clause: `P(X_i=1 | all evidence)`
* per contract: `P(C=1 | all evidence)`
* per goal/spec: upward aggregate (see next section)

Also compute:

* uncertainty: entropy / credible interval width
* attribution: top contributing factors (largest log-likelihood / sensitivity)

## Upward propagation beyond contracts

Once each clause has a posterior, propagate to:

* **Spec**: aggregate relevant clauses (often "all must hold" for a test spec)
* **Requirement**: aggregate child specs/clauses
* **Goal**: aggregate child requirements
* **Intent summary**: aggregate required goals, with risk tags

Propagation should preserve:

* weakest-link drivers (high impact × high fail probability)
* epistemic hotspots (high uncertainty, high decision impact)

Simple, explicit aggregation policies:

* AND-like: `P(pass)=Π_i p_i` (only if independence acceptable; usually not)
* weighted noisy-AND with correlation groups
* OR-like when any one clause can satisfy intent (rare for safety)

## Decision gates as posterior thresholds (not warning counts)

Gate conditions should be risk-aware:

* **Pass** if `P(pass) ≥ τ_p` *and* `uncertainty ≤ τ_u` *and* coverage OK
* **Hold/regen** if:
  * `P(pass)` borderline and epistemic high (missing evidence)
  * high-impact clause has weak support
* **Fail** if high-confidence fail on critical clause

Targeted regen policy: pick clauses with max `expected_impact × epistemic_uncertainty` ("get the most decision-relevant information next").

## Explainability (must be native)

For any Contract/Goal, report:

* posterior `P(pass)`
* uncertainty (entropy or CI)
* top blockers: clauses with high `P(fail) × impact`
* top unknowns: clauses with highest epistemic mass
* evidence trace: which Evidence/Tools moved posteriors most

This is just "messages" / factor contributions surfaced as a ranked list.

## Generalization framework (add modalities)

### 1) Modality contract: "Evidence → likelihood"

Any new modality must provide:

* observed output `Y`
* target clause(s) or parameter(s)
* likelihood model `φ(Y | X, Θ, context)`
* credibility metadata (calibration status, known failure modes, domain of validity)

### 2) Factor library (edge semantics catalog)

Maintain a small set of reusable factor templates:

* measurement-with-tolerance
* classifier-score-with-calibration
* physics-margin
* robust-mixture (outlier)
* noisy-AND/noisy-OR aggregation
* missingness/unknown inflation

### 3) Separation of concerns

* Hypergraph: provenance + traceability + typed relationships
* Inference view: compiled slice + factors + posteriors
* Policy layer: gates/thresholds/criticality/risk

### 4) Learning loop (optional but inevitable)

Over time:

* learn tool calibrations
* learn correlation structures
* learn priors by spec category + historical outcomes

But inference semantics stay the same.

## Summary

* move from Contract-level evidence to **Clause-level** attribution
* represent uncertainty explicitly (prob + interval/entropy + epistemic vs aleatory tags)
* compile hypergraph neighborhood into **local-factor inference**
* propagate posteriors upward with explicit aggregation + correlation control
* gates operate on **posterior + uncertainty**, driving targeted evidence acquisition

## Open questions

1. **Assumption handling**: do assumptions gate validity, or only widen uncertainty/credibility?
2. **Correlation discovery**: how to auto-assign correlation groups (shared CAD export, shared solver, shared prompt lineage)?
3. **Calibration truth**: what is "ground truth" for tool correctness (manufactured inspection, human review, FEA cross-check, historical outcomes)?
4. **Aggregation policy**: noisy-AND parameters, criticality weights, and how they vary by decision risk.
5. **Temporal drift**: tool versions change; how does credibility decay over time without fresh calibration?
