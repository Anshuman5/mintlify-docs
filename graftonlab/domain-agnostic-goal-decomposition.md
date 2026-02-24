---
title: "Domain-Agnostic Goal Decomposition"
description: "From meta-goals to emergent capability through hierarchical constraint-driven planning"
---

## 1. Introduction

The [Hierarchical Constraint-Driven Planning Formalization](/graftonlab/hierarchical-constraint-planning) is already domain-agnostic. Its decomposition tree $\mathcal{T}$, constraint set $\mathcal{C}$, resolver functions $\mathcal{F}$, and VERIFY oracle make no assumptions about whether the goal is physical, digital, or meta. This document explores the implications of that generality: specifically, what happens when the system encounters a subproblem it doesn't yet have the tools to solve, and how it can construct those tools autonomously.

The structural observations hold regardless of domain:

* Any goal decomposes hierarchically (the formalization's tree $\mathcal{T}$)
* Each task carries constraints (cost, time, capability requirements) alongside domain-specific ones
* Completing early tasks builds capability for later ones, so the system's effective resolver set $\mathcal{F}$ grows during execution
* The system may discover mid-cascade that it needs to acquire knowledge from an unexpected domain

Three principles guide how the system operates within this structure:

**Irreducible core.** The formalization's $C_v^{core}$ (Section 3) defines what humans MUST provide: constraints the system cannot derive from context. Beyond this minimum, the system should search autonomously. The design goal is to shrink $C_v^{core}$ as quickly as possible by building capability that lets the system self-derive.

**Autonomous search with improvable heuristics.** The system's search quality may be poor initially, but as long as the heuristic improves (through accumulated capabilities, better cost estimates, richer ontologies), it converges. Each completed task refines the heuristic for the next.

**Capability bootstrapping.** The cascade A, B, ..., Z is not just a task list but a capability accumulation trajectory. Task A's output enables Task B. The system that finishes Task D is strictly more capable than the system that started it, because $\mathcal{F}$ has grown.

## 2. Background: Connecting Constraint Verification to Capability Discovery

### What Datalog Provides

Datalog is a declarative logic programming language where you define facts and rules, and the engine computes all derivable conclusions via forward chaining. Its key property: given a set of rules and facts, Datalog **guarantees completeness**, meaning it will find every conclusion that logically follows. This makes it suitable for tasks where missing a relevant check is unacceptable. See the [Symbolic Reasoning for Verification](/graftonlab/symbolic-reasoning-verification) document (Section 2) for a full primer on Datalog, Souffle, Scallop, and abductive reasoning.

### What the Formalization Already Covers

The [formalization](/graftonlab/hierarchical-constraint-planning) handles:

* **Constraint selection**: $C_v^* = \arg\min |S|$ subject to SAT decidability (Section 3)
* **Constraint optimization**: Z3/NSVIF verifies whether outputs satisfy constraints, with logic ($\sigma = 1.0$) and semantic ($\sigma \in [0,1]$) typing (Section 5)
* **Feedback on failure**: NSVIF-style traces identifying which constraint failed and how (Section 7)
* **Resolver selection**: multi-criteria optimization over completeness, executability, optimality, representation, generalization, efficiency (Section 4)

### What This Document Adds

The formalization assumes the system KNOWS its constraints and HAS resolvers available. This document addresses two gaps:

1. **Constraint discovery**: How does the system determine what the constraints even ARE for a novel subproblem? When the system encounters "build a fault localization tool," what are the time and cost constraints of that subtask? The system must estimate these, and that estimation itself may require domain knowledge it doesn't yet have.
2. **Resolver construction**: What happens when $\mathcal{F}$ contains no resolver capable of handling a subproblem within its constraints? The formalization's escalation path (Section 7) treats this as infeasibility. We propose an alternative: decompose "build a resolver" as a new subgoal, using the same framework recursively.

Datalog provides the machinery for both. It tracks what the system knows, what capabilities it has, and what's blocked. Crucially, it also tracks what's UNKNOWN: tasks whose constraints haven't been estimated yet. This isn't optimization (Z3 handles that); it's **knowledge state management**.

## 3. Layered Ontology

The framework operates at three layers. Only Layer 0 needs to exist at startup. Layers 1 and 2 are constructed on demand.

### Layer 0: Capability and Knowledge State Tracking

This layer doesn't optimize; the formalization's Z3/NSVIF pipeline handles constraint satisfaction. Layer 0 instead maintains a model of **what the system knows and can do**, surfacing gaps that must be filled before execution can proceed.

```
% What tasks exist and what do they need?
task(T) :- decomposes_from(T, SomeGoal).
depends_on(TaskB, TaskA).
requires(Task, Capability).

% What does the system currently have?
has(System, Capability).
blocked(T) :- requires(T, C), not has(system, C).
ready(T) :- not blocked(T), all_deps_complete(T).

% Capability accumulation: completing tasks grants new capabilities
enables(Task, Capability) :- task_output(Task, Capability).
has(system, C) :- completed(T), enables(T, C).

% Constraint discovery: flag tasks with unknown constraints
unknown_cost(T) :- task(T), not cost_estimated(T).
unknown_time(T) :- task(T), not time_estimated(T).
needs_estimation(T) :- unknown_cost(T).
needs_estimation(T) :- unknown_time(T).
```

The last four rules are the key addition. Before the system can even select constraints (formalization Section 3), it must know what constraints apply. A task with `unknown_cost(T)` can't participate meaningfully in Z3 optimization. Layer 0 flags this, and the system must estimate it first, which may require domain knowledge (Layer 2) that doesn't exist yet.

This creates a natural priority ordering:

1. Estimate constraints for unknown tasks (may require domain knowledge acquisition)
2. Pass estimated constraints to the formalization's Z3 pipeline for optimization
3. Execute with the selected resolver
4. Update capabilities on completion

**How does this layer get populated?** Datalog is the bookkeeping layer, not the creative one. The decomposition step itself (the formalization's DECOMPOSE in Section 8) is LLM-driven: an LLM breaks a goal into tasks and labels each with required capabilities. These labels are initially semantic estimates ($\sigma < 1.0$), since the LLM is guessing what a task needs. As the system executes tasks and observes what actually blocked or unblocked, it refines these labels with ground-truth evidence. The LLM proposes structure; Datalog maintains and queries it; Z3 verifies and optimizes over it. Over time, the proportion of LLM-estimated labels shrinks as the system replaces them with empirically confirmed facts.

### Layer 1: Domain Discovery

When Layer 0 detects a blocked task (requires a capability the system doesn't have), Layer 1 determines which domain the capability lives in and how to acquire it.

```
% A blocked task needs a capability
needs_acquisition(T, Cap) :- blocked(T), requires(T, Cap), not has(system, Cap).

% What domain does this capability live in?
domain_of(thermal_analysis, physics).
domain_of(fault_localization, software_engineering).
domain_of(timing_verification, electrical_engineering).

% How can we acquire it?
acquisition_path(Cap, use_tool, Cost, Time) :- external_tool_exists(Cap, Tool, Cost, Time).
acquisition_path(Cap, build_ontology, Cost, Time) :- ontology_buildable(Cap, Cost, Time).
acquisition_path(Cap, llm_approximate, Cost, Time) :- llm_can_approximate(Cap, Cost, Time).
acquisition_path(Cap, human_consult, Cost, Time) :- human_available(Cost, Time).
```

Z3 (via the formalization's resolver selection) picks the acquisition path satisfying cost and time constraints. If no path satisfies, the system decomposes the acquisition itself: "how do I build a cheaper version of this capability?" becomes a new subgoal.

### Layer 2: Domain-Specific Ontology

This is what the [Symbolic Reasoning for Verification](/graftonlab/symbolic-reasoning-verification) document describes for physics. But Layer 2 ontologies are **artifacts the system produces**, not things we hard-code. When the system discovers it needs thermal verification, it builds (or retrieves) a thermal Datalog ontology:

```
% Example: thermal rules (built on demand, not pre-loaded)
thermal_source(X) :- power_dissipation(X, P), P > 0.0.
thermal_path(X, Y) :- thermally_coupled(X, Y).
thermal_path(X, Z) :- thermal_path(X, Y), thermally_coupled(Y, Z).
thermal_violation(Node) :-
    thermal_constraint(Node, MaxTemp),
    thermal_budget_usage(Node, Actual),
    Actual > MaxTemp.
```

When it discovers it needs software fault localization, it builds a different ontology:

```
% Example: SWE rules (built on demand)
imports(File, Module) :- import_statement(File, Module).
call_graph(Caller, Callee) :- function_call(Caller, Callee).
test_touches(TestFile, SrcFile) :- call_graph(TestFile, SrcFile).
likely_fault(SrcFile) :-
    failing_test(T), test_touches(T, SrcFile),
    recently_modified(SrcFile).
```

The same Datalog machinery. Different facts and rules. Constructed when needed.

## 4. Running Example: Autonomous SWE-Bench Solver

### 4.1 The Goal

> "Build an autonomous SWE agent that achieves 80% on SWE-bench Lite, within 24 hours of development, for less than $500 in total compute."

Constraints:

* $c_1$: success rate >= 80% on SWE-bench Lite (300 tasks)
* $c_2$: development time &lt;= 24 hours
* $c_3$: total compute cost (development + evaluation) &lt;= $500
* $c_4$: autonomous, with no human-in-the-loop during evaluation runs

### 4.2 Initial Decomposition

Layer 0 decomposes the goal:

```
Task A: Analyze SWE-bench task structure
         What does "solving a SWE-bench task" actually require?
Task B: Survey solution approaches
         What methods exist? What are their cost profiles?
Task C: Select approach (must satisfy c2, c3)
Task D: Build fault localization capability
Task E: Build patch generation capability
Task F: Build verification loop (apply patch, run tests, diagnose failure)
Task G: Integration + end-to-end testing
Task H: Evaluation on SWE-bench Lite (must satisfy c1)
```

Tasks A and B are research: the system (or a human in the first hours) gathers information that becomes facts in the Datalog knowledge base.

### 4.3 The Cost Wall

Task B produces a cost analysis of known approaches:

| Approach | Per-Task Cost | 300 Tasks | Dev Time | Satisfies c3? |
|----------|---------------|-----------|----------|---------------|
| Pure GPT-4 (SWE-Agent style) | ~$2.00 | $600 | ~8h | NO |
| Pure Claude (Devin style) | ~$1.50 | $450 | ~8h | Marginal |
| Fine-tuned small model | ~$0.10 | $30 | ~40h | NO (c2) |
| Retrieval-augmented | ~$0.80 | $240 | ~16h | YES |
| Hybrid: ontology + light LLM | ~$0.50 | $150 | ~20h | YES |

The pure LLM approach is the obvious first choice. But VERIFY at Layer 0:

```
budget_violation(swe_agent, pure_gpt4) :-
    cost(swe_agent, pure_gpt4, 600), allocated_budget(swe_agent, 500), 600 > 500.
```

**UNSAT.** The cost constraint kills the obvious approach.

### 4.4 Abductive Repair at the Meta Level

The system doesn't just report "budget exceeded." It asks: **what's the minimum-cost change to make a solution feasible?**

Z3 unsat core analysis identifies the bottleneck as per-query LLM cost. The LLM is doing two things:

1. **Understanding the codebase** (reading files, tracing imports, finding relevant code), consuming ~60% of tokens
2. **Generating the patch** (the actual creative/reasoning step), consuming ~40% of tokens

Insight: step 1 is deterministic. You don't need an LLM to trace imports or build a call graph. That's what Datalog does. If the system replaces LLM-based code understanding with Datalog-based code understanding, per-query cost drops by ~60%.

New cost estimate: ~$0.50/task x 300 = $150. **SAT.**

But this requires a capability the system doesn't have: `code_understanding_ontology`.

### 4.5 Dynamic Capability Acquisition

Layer 0 detects:

```
blocked(build_fault_localization) :-
    requires(build_fault_localization, code_understanding_ontology),
    not has(system, code_understanding_ontology).
```

Layer 1 activates. How to acquire `code_understanding_ontology`?

```
acquisition_path(code_understanding_ontology, build_ontology, 80, 6h).
    % Build Datalog rules for imports, call graph, type info
    % Cost: ~$80 in LLM-assisted rule generation + testing
    % Time: ~6 hours

acquisition_path(code_understanding_ontology, use_existing_tool, 0, 1h).
    % Use tree-sitter + LSP for parsing
    % Cost: free (open source)
    % Time: ~1 hour to wire up
```

Z3 selects: use tree-sitter/LSP for the parsing layer (free, fast), then build Datalog rules on top for the reasoning layer ($80, 6h). Total: $80, 7h. Satisfies constraints.

This spawns a sub-decomposition:

```
Task D.1: Set up tree-sitter parsing for Python repos
Task D.2: Build Datalog rules: imports, call graph, type info
Task D.3: Build Datalog rules: test-to-source mapping
Task D.4: Build Datalog rules: fault localization (failing test -> likely files)
Task D.5: Verify: run fault localization on 10 known SWE-bench tasks, check accuracy
```

Each is a concrete, verifiable subtask. The system builds a Layer 2 ontology for software engineering, not because we told it to, but because the cost constraint forced it to discover that pure LLM is wasteful for deterministic reasoning.

### 4.6 The Hybrid Architecture Emerges

After completing D.1-D.5, the system has:

```
has(system, code_understanding_ontology).
has(system, fault_localization).
```

Task E (patch generation) is now unblocked. The system uses a light LLM ONLY for the irreducible creative step: given localized context (from Datalog), generate a patch. Per-query cost: ~$0.50.

Task F (verification loop) uses deterministic checks:

* Apply patch to repo (deterministic)
* Run test suite (deterministic, $\sigma = 1.0$)
* If tests fail, use Datalog fault localization to diagnose (deterministic)
* Feed diagnosis to LLM for repair (one more LLM call)

The system has organically arrived at a hybrid architecture: Datalog for everything deterministic, LLM only for the creative residual. This wasn't designed top-down; it emerged from constraint satisfaction.

### 4.7 Verification at Each Step

| Task | Verification | Cost | $\sigma$ |
|------|--------------|------|--------|
| D.1: tree-sitter parsing | Parse 100 Python files, check AST correctness | ~seconds | 1.0 |
| D.2-D.4: Datalog rules | Run rules on known repos, check against ground truth | ~seconds | 1.0 |
| D.5: fault localization | 10 SWE-bench tasks, compare to known fault locations | ~minutes | 0.9 |
| E: patch generation | Apply patch, run tests | ~minutes | varies |
| G: integration test | End-to-end on 20 tasks | ~30 min | 0.85 |
| H: full evaluation | 300 SWE-bench Lite tasks | ~4 hours | measured |

Every verification step is cheap. The most expensive (full evaluation) is ~4 hours and ~$150. Total budget used: ~$80 (ontology) + $150 (evaluation) = $230, well under the $500 cap.

### 4.8 Cost Accounting

| Category | Cost | Time |
|----------|------|------|
| Research (Tasks A-B) | ~$20 (LLM queries) | ~3h |
| Approach selection (Task C) | ~$5 | ~1h |
| Ontology construction (Tasks D.1-D.5) | ~$80 | ~7h |
| Patch generator wiring (Task E) | ~$30 | ~4h |
| Verification loop (Task F) | ~$15 | ~3h |
| Integration (Task G) | ~$30 | ~3h |
| Evaluation (Task H) | ~$150 | ~4h |
| **Total** | **~$330** | **~25h** |

Time is slightly over the 24h constraint. The system would detect this during planning (before execution) and either: (a) parallelize tasks D and E (they're partially independent), (b) reduce evaluation set size, or (c) flag to the user that 24h is tight and request a relaxation.

## 5. Dynamic Resolver Construction

This section connects the running example to the formal notation from the [Hierarchical Constraint-Driven Planning Formalization](/graftonlab/hierarchical-constraint-planning).

### The Gap in the Formalization

The formalization's resolver selection (Section 4) assumes a pre-existing set $\mathcal{F}$ of resolver functions:

$$
f_v^* = \arg\max_\{f \in \mathcal\{F\}\} \ \mathbf\{w\}_v^\top \mathbf\{q\}(f, C_v^*, G_v)
$$

This works when $\mathcal{F}$ contains at least one resolver capable of handling subgoal $G_v$. But what happens when it doesn't?

In the SWE-bench example, when the system needs `fault_localization` and no resolver in $\mathcal{F}$ provides it, the formalization's feedback loop (Section 7) would cycle through available resolvers, fail on all of them, and eventually escalate as infeasible.

### The Extension: Resolver Construction as Subgoal

When $\mathcal{F}_v^{viable} = \emptyset$ (no resolver satisfies the node's constraints), instead of declaring infeasibility, the system creates a new subtree:

$$
G_\{\text\{build\}\} = \text\{"construct a resolver \} f_\{\text\{new\}\} \text\{ such that \} \mathbf\{q\}(f_\{\text\{new\}\}, C_v^*, G_v) \text\{ satisfies \} \mathbf\{w\}_v
$$

This subgoal decomposes using the same framework ($\mathcal{T}$, $\rho$, VERIFY, feedback) applied to the meta-problem of building a tool. The SWE-bench example's Tasks D.1-D.5 are exactly this: decomposing "build a fault localization resolver" into atomic subtasks.

On completion:

$$
\mathcal\{F\} \leftarrow \mathcal\{F\} \cup \\{f_\{\text\{new\}\}\\}
$$

The resolver set grows during execution. The original node $v$ can now proceed with $f_{\text{new}}$ as its resolver.

### Termination

The recursion bottoms out at **primitive capabilities**, things the system can already do without building new tools:

* Execute code (has a runtime)
* Query an LLM (has API access)
* Read/write files (has filesystem access)
* Search the web (has search API)
* Install packages (has package manager)

If the system cannot construct a resolver using only primitive capabilities within the cost/time budget, THAT is a legitimate infeasibility result. The distinction matters: it's not "I don't have a tool for this" but "no tool can be built within constraints."

### Interaction with Parsimony

The formalization's parsimonious constraint selection (Section 3) still applies. When constructing a new resolver, the system doesn't build the most comprehensive tool possible; it builds the minimum viable tool that satisfies $C_v^*$. The fault localization ontology doesn't need to handle every programming language, just Python (because SWE-bench is Python). Parsimony keeps resolver construction tractable.

## 6. The Self-Bootstrapping Loop

The cascade structure creates a feedback loop where the system's capability grows with each completed task:

```
Iteration 0: System has primitive capabilities only
  {execute_code, query_llm, read_write_files, web_search, install_packages}

  Completes Task A (analyze SWE-bench structure)
  Gains: {swe_bench_task_model}  (understands what a task looks like)

Iteration 1:
  Completes Task B (survey approaches)
  Gains: {approach_cost_models}  (knows cost/accuracy tradeoffs)

Iteration 2:
  Completes Task C (select approach)
  Gains: {selected_architecture}  (committed design decision)

Iteration 3:
  Completes Tasks D.1-D.5 (build ontology)
  Gains: {code_understanding_ontology, fault_localization}
  These are NEW RESOLVERS that didn't exist at iteration 0

Iteration 4:
  Task E is now unblocked (fault localization exists)
  Builds patch generator using the new resolvers
  Gains: {patch_generation}

...and so on
```

Each iteration, the Datalog fact `has(system, Capability)` grows. Tasks that were `blocked` become `ready`. The system's effective $\mathcal{F}$ expands.

This is the evolution analogy made concrete. Evolution's fitness function ("persist") drove the emergence of increasingly complex organisms. Our fitness function ("achieve the goal within constraints") drives the emergence of increasingly capable resolvers. The difference: we can set economically valuable goals and the timescale is hours, not eons.

### The Critical Property

For the loop to converge, each iteration must satisfy: **the capability gained from completing task** $T_i$ **must be sufficient to unblock at least one subsequent task** $T_j$. If a task gains nothing (pure overhead), the system stalls. The decomposer must produce cascades where outputs feed forward.

This is verifiable in Datalog:

```
productive(T) :- enables(T, Cap), requires(T2, Cap), T2 != T, not completed(T2).
unproductive(T) :- task(T), not productive(T).
```

If `unproductive(T)` fires, the system should re-examine why T is in the cascade and whether it can be eliminated or merged.

## 7. PoC Domain Comparison

### 7.1 Robotics Simulation as a Physical Benchmark

To ground the comparison, consider a representative physical goal: "Design a 6-DOF articulated robotic arm: 2kg payload, 0.8m reach, +-1mm repeatability, &lt;$3000 BOM."

**Search space (~10^15 configurations):** The system must choose kinematic architecture (articulated/SCARA/delta), DOF count (4-7), and for each joint: motor type (stepper/BLDC/servo), gear reduction (harmonic/cycloidal/planetary, ratios 30:1 to 160:1), encoder resolution. For each link: material (aluminum/carbon fiber/steel), cross-section geometry, wall thickness (continuous). Then control architecture (PID/computed torque/impedance).

**Why it's hard (the torque-inertia recursion):** Joint N must support the weight of all links and actuators from N+1 to the end-effector, plus payload. A heavier wrist motor means a bigger elbow motor means a bigger shoulder motor. This creates a circular dependency across decomposition levels: decisions at the tip propagate backward to the base. Unlike software tasks where components often decouple cleanly, this coupling is inherent to the physics and cannot be decomposed away.

Additionally, the parameter space is mixed discrete-continuous (motor catalog is discrete, link dimensions are continuous), which defeats both pure gradient methods and pure combinatorial search.

**Verification tools (all seconds-or-less):**

| Check | Tool | Cost |
|-------|------|------|
| Workspace reachability | Forward/inverse kinematics (IKFast) | ~ms |
| Torque requirements | Recursive Newton-Euler dynamics | ~ms |
| Link deflection | Euler-Bernoulli beam theory or FEA | ~ms to ~seconds |
| Full trajectory simulation | MuJoCo (runs >1000x realtime) | ~seconds |
| Collision / self-collision | FCL / MuJoCo built-in | ~ms |
| BOM cost | Component database lookup | ~ms |

Verification is cheap and mostly deterministic. MuJoCo can simulate a full pick-and-place trajectory in 1-5 seconds. The "hard search, cheap verification" asymmetry holds.

### 7.2 Comparison: Software vs Robotics Simulation

| Criterion | Robotics Simulation | Software/Meta Domain |
|-----------|---------------------|----------------------|
| Verification cost | Seconds (MuJoCo, Newton-Euler) | Seconds (run tests, static analysis) |
| Tooling required | MuJoCo, PyBullet, IKFast | Python runtime, test frameworks |
| Domain ontology needed | Physics (torque, deflection, thermal) | Task structure, code understanding |
| Exercises Datalog | Yes: torque-inertia recursion, thermal path propagation | Yes: dependency tracking, fault localization, call graph |
| Exercises Z3 abduction | Yes: motor sizing, link optimization, gear ratio selection | Yes: cost optimization, approach selection |
| Exercises confidence propagation | Yes: $\sigma$ varies (Newton-Euler = 1.0, MuJoCo = 0.85) | Yes: $\sigma$ varies (tests = 1.0, LLM judge = 0.7) |
| Constraint coupling | Very high (torque-inertia recursion, cross-level physics) | Moderate (mostly at interface boundaries) |
| Exercises resolver construction | Partially (needs MuJoCo integration) | Fully (builds its own analysis tools) |
| Immediately useful | Mid-term (manufacturing validation target) | Yes (we use the output directly) |
| Setup overhead | MuJoCo env, URDF models, component database | Minimal (Python + test runners) |

Software goals exercise every mechanism with minimal setup and produce artifacts we directly use. Robotics exercises stronger constraint coupling but requires domain-specific tooling. Both validate the framework; software is a faster first iteration.

### 7.3 The Physical Domain Isn't Gone

Physics-domain verification (the [Symbolic Reasoning for Verification](/graftonlab/symbolic-reasoning-verification) document's Datalog rules for thermal, structural, etc.) becomes a Layer 2 ontology that the system constructs WHEN a goal requires it. If the goal is "build an autonomous manufacturing system," the cascade will eventually hit a task that requires thermal verification, and the system will build (or retrieve) the thermal ontology at that point.

The ontology work we've done isn't wasted. It's a blueprint for what a Layer 2 physics ontology looks like. The system could use it as a template or starting point when it discovers it needs physics reasoning.

## 8. Unresolved Questions

* **Constraint discovery depth**: When the system encounters `unknown_cost(T)`, how does it estimate? LLM heuristic? Analogical reasoning from similar past tasks? And how confident is that estimate ($\sigma$ of the estimate itself)?
* **Capability representation**: How do we formally represent "capabilities" so Datalog can reason about them? Is it just string tags, or do capabilities have structure (input types, output types, accuracy guarantees)?
* **Infinite regress**: Building a resolver requires capabilities. If those capabilities also need resolvers, we recurse. What's the practical depth before we hit primitives? Is it usually 2-3 levels, or can it explode?
* **Human handoff boundary**: What exactly constitutes the "irreducible core" that humans must provide? Just the goal + top-level constraints, or also the first few decomposition levels?
* **Evaluation without evaluation**: Can the system learn to predict its own success rate without running full evaluations? Self-assessment as a built capability?
* **Heuristic improvement**: What is the heuristic, concretely? Is it the Datalog-surfaced priority ordering of tasks? The Z3-computed cost estimates? How does it improve across iterations?

---

## Appendix A: Discussion Analysis

*This appendix captures the team discussion that led to the framing in this document.*

### The Catalyst

Team feedback challenged an early assumption: that the first test of the hierarchical constraint-driven planning framework should be a physical engineering domain (robotics, IC design, etc.). The core argument:

> The system needs to handle any type of goal. A physical goal should not be any different than a digital goal, especially as solving either may require execution through the other. If the goal is "build a general first principles reasoning system within 24 hours, starting now, and do so autonomously, leveraging humans only for the first 6 hours" this is a valid goal that has a solution. One can pencil out the solution by listing all the reasons it wouldn't be achievable (constraints) and then finding a way around each, wherein the path may run through any level of abstraction.

The key observations:

1. **Physical vs digital is a false dichotomy.** A physical goal ("design a bracket") may require software verification. A digital goal ("build an autonomous SWE agent") may require understanding of compute hardware constraints. The framework should not assume the domain up front.
2. **One goal can force everything else to emerge.** Evolution had one goal: persistence of life over time. Everything around us emerged from that. Unlike evolution, we have control over the goal and can set goals that are economically valuable.
3. **The cascade bootstraps capability.** Any goal decomposes into tasks A, B, C, ..., Z. Completing task A may grant capabilities needed for task B. The system gets more capable as it executes.
4. **Limit human input to the irreducible core.** Beyond what only a human can provide, the system should search autonomously. Even poor-quality searches converge if the heuristic can improve.

### Example Goals Discussed

| Goal | Why It's Hard | Cascade Shape |
|------|---------------|---------------|
| Build an autonomous SWE that is fully lights-out, within 24h, for $X cost | Requires robust error recovery, context management, tool use, planning | ~20 tasks, many with sub-decompositions |
| Build an autonomous agent achieving 10x the current METR benchmark without 10h evaluation per run | Current state-of-art: 80% at 55 min. Needs self-recovery, efficient evaluation | ~15 tasks, evaluation efficiency is itself a sub-problem |
| Solve the Riemann Hypothesis for &lt;$X compute | Mathematical, but decomposes into sub-conjectures, proof strategies, verification | ~unknown depth, proof search is the hard part |
| Build autonomous code generation equal to LLMs but 10x faster | Requires either specialized hardware, better algorithms, or compilation tricks | ~25 tasks, may discover it needs hardware co-design |
| Build an autonomous quantitative trading firm posting impossible returns for &lt;$X initial compute | Strategy discovery, risk management, execution infrastructure, regulatory compliance | ~30 tasks, crosses financial + software + legal domains |

The ability to produce this breakdown could itself be a goal we assign. Practically, we can hand-author the first few levels (tasks A-D) and let the system handle E onward, gradually reducing human involvement as capability accumulates.
