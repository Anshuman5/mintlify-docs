---
title: "Symbolic Reasoning for First-Principles Verification"
description: "Using Datalog, Z3, and symbolic reasoning for deterministic constraint verification"
---

## 1. Introduction

### 1.1 The Problem

In [Hierarchical System Decomposition](https://graftonsciences.getoutline.com/doc/hierarchical-decomposition-design-document-wZWekif5tJ) we describe how to recursively decompose a complex chip fabrication engineering systems into verifiable subproblems. [Hierarchical Constraint-Driven Planning Formalization](https://graftonsciences.getoutline.com/doc/hierarchical-constraint-driven-planning-formalization-8c6MpeON72)  further formalizes the mathematical framework for this process: a decomposition tree $\mathcal{T}$ where each node $v$ carries a subgoal $G_v$, a constraint subset $C_v^* \subseteq \mathcal{C}$, a resolver function $f_v$, and a confidence $\sigma_v$.

At the leaves of this tree, individual components must be checked against physical constraints (thermal limits, structural loads, DFM rules, etc.) and the results propagated upward through the confidence path $\sigma_{\text{path}}(v) = \prod \sigma_u$ to determine whether system-level specifications are satisfied.

Three problems arise at this boundary between decomposition and physical reality:


1. **Constraint surfacing**: The formalization defines a relevance function $\rho(c, G_v, d)$ that filters which constraints from the universal set $\mathcal{C}$ apply at node $v$. But $\rho$ is left abstract. For physics constraints, we need a concrete, deterministic, complete implementation of $\rho$ -- one that finds *all* relevant physical checks, not a stochastic sample.
2. **Evidence composition**: Each leaf produces domain-specific evidence (thermal measurements, structural analysis, manufacturing feasibility). Parent nodes need a uniform view to check aggregate constraints. A thermal budget at the system level needs to find *all* sources of heat at the leaf level -- this is the global consistency check ($\text{SAT}(\bigcup C_v^{sat})$, formalization Section 11) applied to physics.
3. **Constraint repair**: When VERIFY returns UNSAT at node $v$, the formalization's feedback loop (Section 7) provides "which constraint failed, how." For physics constraints, we can go further: compute the *minimum-cost set of parameter changes* that would make the constraint satisfiable. This is enhanced feedback via abduction.

### 1.2 Why LLMs Alone Cannot Solve This

The obvious question: why not use an LLM as the resolver $f_v$ for implementing $\rho$ -- let it decide which physics checks apply? LLMs are powerful and general-purpose, but they have fundamental properties that make them unsuitable as the *reasoning engine* for constraint surfacing:

**Non-determinism.** Ask an LLM the same question twice with the same context and you may get different answers. The checks it lists on Monday may differ from Tuesday. For verification -- where the entire point is reliable, auditable assessment of whether a constraint is met -- this is disqualifying. We need the same inputs to produce the same $C_v^{eligible}$ every time.

**No completeness guarantee.** When an LLM lists thermal sources affecting a component, there is no mechanism ensuring it found *all* of them. It may list 5 of 7 sources one time and 6 of 7 the next, but there is no way to know whether the list is exhaustive. For the global consistency check (formalization Section 11), where missing a single cross-branch constraint coupling invalidates the result, this is structural, not a quality problem.

**No provenance or audit trail.** When the LLM says "check thermal drift," why did it say that? What rule or physical principle led to that conclusion? The answer is opaque. For an engineering verification system that needs to justify its conclusions to regulators, customers, or internal review, this lack of traceability is a serious liability. The formalization distinguishes logic constraints ($\sigma = 1.0$) from semantic constraints ($\sigma \in [0,1]$) -- an LLM-derived check assignment is inherently semantic, never logic.

**Sensitivity to prompt framing.** The checks an LLM surfaces depend heavily on how you ask. "What DFM checks apply?" yields different results than "What could go wrong with this part?" which yields different results than "List all physical phenomena affecting this component." The relevance function $\rho$ should not depend on the phrasing of the question.

**Cost at scale.** At 10,000+ components, querying an LLM per-component for applicable checks is expensive in both latency and token cost, and the non-determinism compounds -- each query is an independent roll of the dice. A deductive system evaluates $\rho$ for all 10,000 components in a single batch evaluation in seconds.

**Hallucination risk in critical paths.** An LLM may confidently assert that a check passes when it should fail, or invent a physical phenomenon that does not apply. In a verification system, a false positive (claiming a constraint is satisfied when it is not) propagates upward through $\sigma_{\text{path}}$ with $\sigma = 1.0$ when it should be 0 -- creating false confidence at the system level.

**Core claim**: LLMs are valuable as resolvers $f_v$ for *populating facts* (extracting properties from datasheets), *generating hypotheses* (suggesting what might matter for novel situations), and *explaining results* (summarizing symbolic derivations in natural language). But the implementation of $\rho$ -- from component properties to applicable constraints -- must be deterministic, complete, and auditable. This is what symbolic reasoning provides.

### 1.3 Relationship to the Constraint-Driven Planning Formalization

This document solves a **subproblem** within the formalization's framework. The formalization defines the outer algorithmic skeleton (decompose, select constraints, resolve, verify, feedback, gate, recurse). This document provides concrete mechanisms for several of its abstract operations, specifically for the *physics domain*:

| Formalization (abstract) | This Document (concrete) |
|--------------------------|--------------------------|
| Universal constraint set $\mathcal{C}$ -- "all constraints that could govern the problem" | Physics ontology (Layer 1) + Datalog rules (Layer 2). Our ontology is what makes $\mathcal{C}$ exhaustive for physics constraints. Gap detection (Section 6) audits this completeness. |
| Relevance function $\rho(c, G_v, d)$ -- filters which constraints apply at node $v$ | Datalog forward chaining: `applicable_check(Component, Check)`. The Datalog layer is the implementation of $\rho$ for physics constraints -- deterministic, complete, auditable. |
| Constraint typing `type(c_i)` as logic vs semantic | Our Layers 1-2 (deductive rules) produce logic-typed constraints ($\sigma = 1.0$). Layer 4 (neural bridge) produces semantic-typed constraints ($\sigma \in [0,1]$). The layer boundary IS the type boundary. |
| VERIFY via CSP (Section 5) -- Z3 + LLM judge per type | Our Layer 2 (deductive evaluation) implements VERIFY for physics-domain logic constraints. Existing Z3 V4 pipeline handles the CSP formulation. |
| Feedback loop (Section 7) -- "which constraint failed, how" | Our Layer 3 (abductive repair) extends this: not just identification of the failed constraint, but computation of minimum-cost repairs via Z3 Optimize and valid alternative configurations via ASP. |
| Global consistency (Section 11) -- $\text{SAT}(\bigcup C_v^{sat})$ | Our transitive closure over physics coupling paths (Datalog) finds cross-branch dependencies that the global check must verify. Without this, Section 11 can only check constraints it knows about. |
| Confidence $\sigma_{\text{path}}(v) = \prod \sigma_u$ | Our unknown-unknowns margin (Section 8.3) is a correction term: $\sigma_v' = \sigma_v \cdot (1 - \epsilon_{\text{novelty}})$ accounting for incompleteness of $\mathcal{C}$ at that node. |
| Irreducible core $C_v^{core}$ -- constraints humans must seed | Our N4 novelty level (Section 8.1) -- genuinely unknown physics requiring expert input. At the physics layer, $C_v^{core}$ includes constraints that the ontology cannot derive. |

**What the formalization provides that we don't**: the recursive algorithm control flow (Section 8 of formalization), parsimonious constraint selection (Section 3), resolver selection criteria (Section 4), confidence gating and backtracking (Section 6), termination conditions (Section 9), artifact assembly (Section 12).

**What we provide that the formalization needs**: physics-domain $\rho$, completeness mechanism for $\mathcal{C}$, enhanced feedback via abduction, cross-branch coupling detection for global consistency.


---

## 2. Background: Logic Programming and Deductive Databases

*This section provides context for team members unfamiliar with these techniques. Those with a logic programming background can skip to* [*Section 3*](#3-the-three-verification-challenges)*.*

### 2.1 What Problem Do They Solve?

Consider this question: "Does the collector optics assembly satisfy its thermal constraint of &lt;= 50 degrees C?"

In the formalization's terms, this is a VERIFY call at node $v$ = collector optics, checking constraint $c_i$ = "temperature &lt;= 50C" against the output $\mathcal{O}_v$. To evaluate this, you need to:

* Find every component that produces heat (actuators, electronics, friction sources)
* Trace every thermal path from those sources to the collector optics
* Aggregate the total thermal load
* Compare against the constraint

In Python, you'd write nested loops, graph traversals, and aggregation code. If you later add a new type of heat source (say, eddy current heating from a nearby motor), you'd need to modify the traversal code. If you want to ask a different question ("what is the vibration budget usage?"), you'd write a largely parallel set of loops.

**Deductive databases** solve this differently. You state *facts* (component X dissipates 5W, component X is thermally coupled to component Y) and *rules* (if A is a thermal source and A is thermally coupled to B, then A contributes to B's thermal load). The system then *automatically derives all conclusions* from those facts and rules. You don't write traversal code -- you write the rules once, and the engine handles finding all paths, all sources, all aggregations.

The key properties:

* **Declarative**: You state *what* is true, not *how* to compute it. The engine figures out the computation.
* **Complete**: The engine finds *all* derivable conclusions, not just the first one. If there are 47 thermal contributors, it finds all 47.
* **Deterministic**: Same facts + same rules = same conclusions, every time. This is what makes deductive evaluation suitable for logic-typed constraints ($\sigma = 1.0$).
* **Composable**: Rules from different domains (thermal, structural, electrical) coexist and interact without special integration code.

### 2.2 Datalog: The Core Language

Datalog is a query language for deductive databases. It is a restricted subset of Prolog (no function symbols, guaranteed termination) that is powerful enough for the reasoning we need.

A Datalog program consists of **facts** and **rules**.

**Facts** are ground truths about the world:

```
% Components and their properties
component(mirror_mount).
component(actuator).
component(collector_frame).

power_dissipation(actuator, 5.0).        % actuator dissipates 5W
domain(mirror_mount, mechanical).
material(mirror_mount, aluminum_7075).

% Physical connections
thermally_coupled(actuator, collector_frame).
thermally_coupled(collector_frame, mirror_mount).
```

**Rules** express relationships that can be derived from facts:

```
% Direct rule: anything dissipating power is a thermal source
thermal_source(X) :- power_dissipation(X, P), P > 0.

% Recursive rule: thermal paths are transitive
%   "X has a thermal path to Z if X has a thermal path to Y,
%    and Y is thermally coupled to Z"
thermal_path(X, Y) :- thermally_coupled(X, Y).
thermal_path(X, Z) :- thermal_path(X, Y), thermally_coupled(Y, Z).

% Aggregation: total thermal load on a component
thermal_load(Target, sum(P)) :-
    thermal_source(Src),
    thermal_path(Src, Target),
    power_dissipation(Src, P).
```

**Queries** ask what is true:

```
?- thermal_load(mirror_mount, Load).
% Answer: Load = 5.0
%   (actuator's 5W reaches mirror_mount through collector_frame)
```

The critical point: *we never wrote code to traverse the thermal graph*. The Datalog engine does this automatically via its evaluation algorithm. If we add a new component with power dissipation and a thermal path to the mirror mount, it is automatically included in the next query -- no code changes needed.

**Why not just use SQL?** SQL can express simple joins and aggregations, but it cannot express *recursive* queries naturally (thermal paths of arbitrary length through a graph). Datalog's recursion is first-class and guaranteed to terminate. Modern SQL does support recursive CTEs, but Datalog is purpose-built for this kind of reasoning and has formal completeness guarantees that SQL lacks.

**Why not just use Python?** You could implement all of this in Python with graph traversal code. The problem is maintainability and completeness. Every new type of physical coupling requires new traversal code. Every new query requires a new function. With Datalog, you add a rule and the engine handles all affected queries. More importantly, Datalog's bottom-up evaluation *guarantees* it finds all derivable facts -- a Python implementation has to be manually verified for completeness.

### 2.3 Souffle: Production Datalog Engine

[Souffle](https://souffle-lang.github.io/) is a high-performance Datalog engine developed at Oracle Labs and the University of Sydney. It compiles Datalog programs to parallel C++, achieving performance suitable for industrial-scale analysis.

**Key properties**:

* Handles billions of facts (our system will have millions at most)
* Compiles to native code -- evaluation is extremely fast
* Supports aggregation (sum, min, max, count)
* Supports stratified negation ("X is true if Y is NOT true")
* Mature, well-tested, open-source
* Used in production for program analysis at Oracle, Amazon, and others

**Limitation**: Souffle is purely symbolic. Every fact is either true or false ($\sigma = 1.0$ only). It does not handle probabilities or confidence scores natively. For our system, this means confidence propagation for semantic-typed constraints would need to be handled separately.

### 2.4 Scallop: Neurosymbolic Datalog

[Scallop](https://www.scallop-lang.org/) (I co-authored this) extends Datalog with two capabilities critical to our problem:

**Probabilistic facts**: Every fact can carry a confidence score -- directly modeling the formalization's $\sigma_i \in [0,1]$:

```
thermal_conductivity(mount_frame, 237.0) :: 0.95.  % 95% confident
thermal_conductivity(mount_frame, 200.0) :: 0.05.  % 5% chance it's lower
```

When the engine derives conclusions from probabilistic facts, it automatically computes the confidence of the conclusion using provenance semirings (a mathematical framework for tracking how conclusions depend on their inputs). This maps directly to the formalization's confidence accumulation: the provenance semiring computes $\sigma_v$ from the $\sigma_i$ of contributing constraints.

**Neural integration**: Scallop can call neural networks (including LLMs) to populate facts, while keeping the reasoning itself symbolic and deterministic.

```
@gpt("Given this component description, what is its power dissipation in watts?")
type power_dissipation(component: String, watts: f64)
```

The LLM extracts the fact (semantic-typed, $\sigma < 1.0$); Scallop reasons over it deductively. The reasoning chain is auditable even though the inputs came from a neural model. Facts derived purely from deductive rules over logic-typed inputs retain $\sigma = 1.0$.

**Trade-offs vs Souffle**:

|     | Souffle | Scallop |
|-----|---------|---------|
| Performance | Compiled C++, billions of facts | Rust backend, millions of facts |
| Probabilistic reasoning | No (encode manually) | Native (provenance semirings) |
| Neural integration | No      | Native (@gpt, @clip, etc.) |
| Maturity | Production-grade | Research project |
| Incremental evaluation | Limited | Limited |

**Our recommendation**: Start with Scallop for prototyping (probabilistic reasoning is too valuable to skip -- it natively implements the formalization's $\sigma_v$ computation). If performance becomes a bottleneck at scale, port the core rules to Souffle and handle confidence in a separate layer.

### 2.5 Abductive Reasoning

Standard deductive reasoning goes **forward**: given facts and rules, derive conclusions.

```
Facts: power_dissipation(actuator, 5.0), thermal_path(actuator, mirror_mount)
Rule: thermal_load(X, P) :- thermal_source(S), thermal_path(S, X), power_dissipation(S, P)
Conclusion: thermal_load(mirror_mount, 5.0)   [derived automatically]
```

**Abductive reasoning** goes **backward**: given an unwanted conclusion (UNSAT from VERIFY), find what facts would need to change to eliminate it.

```
Unwanted conclusion: thermal_violation(collector_optics)   [VERIFY returned UNSAT]
Question: What is the minimal set of fact changes that makes this
          conclusion no longer derivable?
Answers:
  (a) Remove: power_dissipation(actuator, 5.0), Add: power_dissipation(actuator, 2.0)
  (b) Remove: thermally_coupled(actuator, collector_frame)  [add insulation]
  (c) Change: thermal_constraint(collector_optics, 50) to thermal_constraint(collector_optics, 60)
```

Each answer is a *repair strategy*. The system can rank them by cost, feasibility, or locality (prefer leaf-level changes over system-level constraint relaxation).

This extends the formalization's feedback loop (Section 7). The formalization says: on UNSAT, provide feedback about *which constraint failed and how*. Abductive reasoning goes further: it computes *what to change to fix it*, ranked by minimum cost. In the formalization's terms, abduction produces structured feedback that accelerates convergence of the multi-turn loop:

$$
\text\{feedback\}(\mathcal\{O\}_v^\{(t-1)\}, C_v^*) = \\{(c_i, \text\{violation details\}, \text\{min-cost repair\})\\}
$$

**Implementation options**:

* **Z3 / SMT solvers**: We already use Z3 for V4 SAT checks. Z3's `Optimize` class can solve "find minimum-cost parameter changes such that all constraints are satisfied." This handles continuous constraints (thermal &lt;= X) naturally.
* **Answer Set Programming (ASP / Clingo)**: Good for combinatorial repair -- "find all valid \{material, geometry, process\} combinations satisfying these constraints." Handles preferences and defaults natively.
* **Abductive Logic Programming (ALP)**: Direct implementation of backward reasoning. Finds minimal explanations for observations. Less mature tooling than Z3 or ASP.

### 2.6 Summary: When to Use What

| Tool | Use For | Analogy |
|------|---------|---------|
| **Datalog** (language) | Expressing physics rules and queries; implementing $\rho$ | The "language" you write rules in |
| **Souffle** (engine) | Fast, large-scale rule evaluation | A fast compiler for that language |
| **Scallop** (engine) | Rule evaluation with $\sigma_v$ tracking and neural integration | A smarter compiler that also handles uncertainty |
| **Z3** (solver) | VERIFY_CSP for logic constraints; minimum-cost repair (abduction) | A calculator that can work backwards from answers to inputs |
| **ASP/Clingo** (solver) | Combinatorial search over valid configurations | A search engine for valid designs |

These are complementary, not competing. A typical query flow:


1. **Datalog rules** (in Scallop or Souffle) implement $\rho$ to derive $C_v^{eligible}$, then evaluate constraints and check for violations
2. If VERIFY returns UNSAT, **Z3 Optimize** computes minimum-cost parameter changes (enhanced feedback for Section 7 loop)
3. If the repair requires choosing among discrete alternatives (different materials, different geometries), **ASP** enumerates valid combinations ranked by preference


---

## 3. The Three Verification Challenges

With the background in place, we can state the three challenges in terms of the formalization:

### Challenge 1: Implementing $\rho$ -- Complete Constraint Surfacing

**Problem**: The formalization defines $C_v^{eligible} = \{c \in \mathcal{C} \mid \rho(c, G_v, d) = 1\} \setminus C^{resolved}_{<v}$ but leaves $\rho$ abstract. For physics constraints, what is the concrete implementation of $\rho$ that is deterministic, complete, and auditable?

**Why LLMs fail here**: An LLM implementing $\rho$ produces a reasonable but non-deterministic and incomplete $C_v^{eligible}$. Ask again, get a different set. No guarantee of exhaustiveness. No auditability of why a constraint was or wasn't included. (See Section 1.2.)

**Symbolic solution**: Encode physics knowledge as Datalog rules. Forward-chain from the component's properties and context to derive all applicable checks. Completeness follows from the rule base: if the rules cover all known physical phenomena, and the engine evaluates all rules, then $C_v^{eligible}$ is complete. Deterministically, every time. All derived constraints are logic-typed ($\sigma = 1.0$).

```
% This IS the implementation of rho for physics constraints
applicable_check(Component, Check) :-
    component_property(Component, Prop),
    check_requires(Check, Prop),
    context_applies(Component, Check).
```

### Challenge 2: Deterministic Evidence Composition for Global Consistency

**Problem**: The formalization's global consistency check (Section 11) requires $\text{SAT}(\bigcup C_v^{sat})$ -- the union of all leaf-resolved constraints must remain jointly satisfiable. For physics, this means aggregating contributions from potentially thousands of leaf components. A system-level thermal constraint ("total heat dissipation &lt;= X watts") needs to find *all* sources -- missing a single thermal source makes the global SAT check invalid.

**Symbolic solution**: Datalog's transitive closure automatically finds all coupling paths across branches. Aggregation operators (sum, max) compose the evidence. Adding a new component with a thermal path automatically includes it in the next global consistency evaluation. This provides the cross-branch coupling detection that Section 11 requires.

### Challenge 3: Enhanced Feedback for Constraint Repair

**Problem**: When VERIFY returns UNSAT at node $v$ (formalization Section 5), the feedback loop (Section 7) needs structured feedback. For physics constraints, we can provide more than "constraint $c_i$ failed" -- we can compute the minimum-cost repair.

**Symbolic solution**: Abductive reasoning (via Z3 Optimize or ASP) systematically enumerates repair options. Z3 finds minimum-cost continuous parameter changes. ASP enumerates valid discrete configurations. Both are deterministic and exhaustive (within their search space). This accelerates convergence of the multi-turn loop by giving the resolver $f_v$ precise repair targets rather than just failure descriptions.


---

## 4. Proposed Architecture: Four Layers

The system is organized into four layers, each implementing specific operations from the formalization:

```
Layer 4: Neural Bridge          [LLMs]
         Resolvers for semantic-typed constraints (sigma < 1.0)
         Populates facts, generates hypotheses for novel situations
         ---------------------------------------------------
Layer 3: Abductive Repair       [Z3 Optimize, ASP/Clingo]
         Enhanced feedback for formalization Section 7 loop
         Computes minimum-cost repairs on UNSAT
         ---------------------------------------------------
Layer 2: Deductive Rules        [Datalog via Scallop/Souffle]
         Implements rho: forward-chains to derive C_v^eligible
         Implements VERIFY for physics logic constraints (sigma = 1.0)
         ---------------------------------------------------
Layer 1: Physics Ontology       [Structured knowledge base]
         Populates C (universal constraint set) for physics domain
         Encodes phenomena, coupling mechanisms, property relevance
```

Information flows upward (facts from L1 feed rules in L2, UNSAT triggers L3, gaps invoke L4) and downward (L4 populates facts in L1, L3 proposes changes that L2 re-evaluates).

The layer boundary corresponds to the formalization's constraint typing: Layers 1-2 produce logic-typed constraints ($\sigma = 1.0$, symbolically verified). Layer 4 produces semantic-typed constraints ($\sigma \in [0,1]$, LLM-judged).

### 4.1 Layer 1: Physics Coupling Ontology

A formal ontology encoding how the physical world works at the level our system needs. This layer populates $\mathcal{C}$ for the physics domain -- it is the source of truth for what physical constraints exist.

**What it encodes**:

| Category | Examples |
|----------|----------|
| Physical phenomena | thermal conduction, thermal radiation, convection, mechanical vibration, fatigue, electromagnetic coupling, corrosion, outgassing |
| Propagation mechanisms | "thermal conduction propagates through solid contact," "vibration propagates through rigid connections" |
| Property relevance | "thermal conduction depends on thermal conductivity, contact area, temperature gradient" |
| Phenomenon interactions | "thermal expansion causes mechanical stress," "vibration causes fatigue" |
| Validity regimes | "Fourier's law valid for T &lt; 3000K in metals," "Hooke's law valid below yield strength" |

**What it does NOT encode**: specific component properties, specific check results, or design decisions. Those are facts that live in the component database and hypergraph. The ontology encodes the *invariant structure of physics* -- the rules that apply regardless of what specific system is being designed.

### 4.2 Layer 2: Deductive Rules -- Implementing $\rho$ and VERIFY

Datalog rules that operate over the ontology (Layer 1) plus component-specific facts from the hypergraph. This layer has two roles in the formalization:


1. **Implements** $\rho$: "What constraints from $\mathcal{C}$ apply to component $v$?" -- forward chaining from properties + context to derive $C_v^{eligible}$
2. **Implements VERIFY for physics constraints**: "Is constraint $c_i$ satisfied by output $\mathcal{O}_v$?" -- constraint evaluation with $\sigma = 1.0$ (logic-typed)

Example rules for thermal analysis:

```
% Any component dissipating power is a thermal source
thermal_source(X) :- power_dissipation(X, P), P > 0.0.
thermal_source(X) :- friction_source(X, F), F > 0.0.
thermal_source(X) :- resistive_heating(X, R), R > 0.0.

% Transitive thermal coupling -- finds cross-branch paths for Section 11
thermal_path(X, Y) :- thermally_coupled(X, Y).
thermal_path(X, Z) :- thermal_path(X, Y), thermally_coupled(Y, Z).

% All thermal contributors to a target -- COMPLETE by construction
thermal_contributor(Target, Src, P) :-
    thermal_source(Src),
    thermal_path(Src, Target),
    power_dissipation(Src, P).

% Aggregate and check -- this is VERIFY for the thermal constraint
thermal_budget_usage(Target, sum(P)) :- thermal_contributor(Target, _, P).
thermal_violation(Node) :-
    thermal_constraint(Node, MaxTemp),
    thermal_budget_usage(Node, Actual),
    Actual > MaxTemp.
```

The completeness guarantee: Datalog's bottom-up evaluation derives *all* facts entailed by the rules. If there are 47 thermal contributors to the collector optics, the engine finds all 47. This means $C_v^{eligible}$ is complete (up to the ontology's coverage), and the global consistency check (Section 11) can verify cross-branch coupling exhaustively.

### 4.3 Layer 3: Abductive Repair -- Enhanced Feedback

When Layer 2's VERIFY returns UNSAT (e.g., `thermal_violation(collector_optics)`), Layer 3 activates. This extends the formalization's feedback loop (Section 7) from diagnostic feedback to prescriptive repair:

**Formalization Section 7 feedback**: "Constraint $c_i$ (thermal &lt;= 50C) failed. Actual value: 65C. Contributing facts: actuator (5W), pump motor (3W)."

**Our enhanced feedback**: "Constraint $c_i$ failed. Minimum-cost repairs: (a) reduce actuator power from 5W to 2W \[cost: $500\], (b) add thermal insulation between actuator and frame \[cost: $200\], (c) relax constraint to 70C \[requires parent contract renegotiation\]."

**Using Z3 (continuous parameter optimization)**:

```python
opt = z3.Optimize()

# Design parameters as variables with current values as soft constraints
for param, current_val in design_params.items():
    delta = z3.Real(f"delta_{param}")
    opt.add_soft(delta == 0, weight=change_cost[param])

# Hard constraint: thermal budget must hold
opt.add(sum(thermal_contributions) <= thermal_limit)

# Minimize total cost of changes
opt.minimize(sum(abs(deltas) * costs))
```

**Using ASP/Clingo (discrete configuration search)**:

```
% Available materials for mount
material_option(mount, aluminum_7075).
material_option(mount, invar_36).
material_option(mount, titanium_6al4v).

% Constraint: CTE must be below threshold
:- selected_material(mount, M), cte(M, C), C > cte_limit.

% Preference: minimize cost
#minimize { cost(M) : selected_material(mount, M) }.
```

### 4.4 Layer 4: Neural Bridge -- Semantic-Typed Constraints

LLMs serve as resolvers $f_v$ for specific, bounded roles. Their outputs are semantic-typed ($\sigma < 1.0$):

| Role | What the LLM Does | What It Does NOT Do |
|------|-------------------|---------------------|
| **Fact extraction** | Extract component properties from datasheets, CAD metadata, manufacturer specs | Implement $\rho$ (decide which checks to run) |
| **Hypothesis generation** | "What failure modes might exist for this novel component?" | Determine whether a failure mode actually applies (that's Layer 2's job) |
| **Rule drafting** | Draft candidate Datalog rules from textbook descriptions | Validate those rules (human/test review required before entering Layer 1) |
| **Explanation** | Generate natural-language summaries of symbolic derivations | Replace the symbolic derivation |
| **Gap filling** | Suggest what physics might matter for truly novel situations | Serve as the source of truth for those suggestions |

**The critical boundary**: LLM outputs are semantic-typed ($\sigma < 1.0$). Symbolic derivations are logic-typed ($\sigma = 1.0$). The system labels which conclusions come from which layer so that $\sigma_v$ is computed correctly via the formalization's type-dependent confidence (Section 5).


---

## 5. Ontology Bootstrapping

### 5.1 Physics Knowledge is Hierarchical

Building a physics ontology sounds unbounded, but physics knowledge has natural layers that map to encoding effort:

| Layer | Content | Estimated Rules | Stability |
|-------|---------|-----------------|-----------|
| **L0: Conservation laws** | Energy, momentum, mass, charge conservation | \~10 rules      | Universal, essentially permanent |
| **L1: Constitutive relations** | Fourier's law, Hooke's law, Ohm's law, Stefan-Boltzmann | \~50-100 per domain | Well-established, change rarely |
| **L2: Engineering correlations** | Thermal resistance networks, beam deflection formulas, stress concentration factors (Peterson's), Nusselt correlations | \~500-1000 per domain | Textbook-derived, stable |
| **L3: Design rules** | DFM rules, PCB trace width, fillet radii constraints | \~100-500 per domain | What we already have in `verifier_core/rulepack/` |

We don't need all of physics. We need enough to make $\mathcal{C}$ (the universal constraint set) complete for the domains the decomposition tree encounters. We build top-down: conservation laws first (cheap, universal), then constitutive relations for encountered domains, then engineering correlations as needed.

### 5.2 Three Bootstrapping Sources

**Source A: Formalize existing rules.** The DFM rules in `verifier_core/rulepack/` are already Layer 3 rules expressed in Python. Translating them to Datalog is mechanical work -- no new physics knowledge needed, just a representation change that immediately enables queryability, composability, and gap detection against $\mathcal{C}$.

**Source B: LLM-assisted extraction from handbooks.** The real bottleneck is Layer 2 -- engineering correlations from textbooks (Shigley's Mechanical Engineering Design, Incropera's Heat Transfer, Sedra/Smith for electronics). The process:


1. Feed LLM a handbook chapter + target Datalog schema
2. LLM extracts candidate rules with validity regimes and source citations
3. Domain expert reviews (or test cases validate expected derivations)
4. Approved rules enter the ontology with provenance metadata

LLMs excel at structured extraction from text. This accelerates knowledge engineering from months to days, though validation remains human. Note: the extracted rules enter as logic-typed ($\sigma = 1.0$) once validated, not semantic-typed.

**Source C: Inductive Logic Programming (ILP) from existing examples.** Given our existing hand-crafted check assignments ("for a mirror mount, run checks X, Y, Z"), ILP systems (Popper, ILASP) can learn the *general rules* that produce those assignments:

```
% From examples like:
%   + applicable_check(mirror_mount, dfm_wall_thickness)
%   - applicable_check(mirror_mount, pcb_trace_width)
% ILP learns:
applicable_check(C, dfm_wall_thickness) :-
    domain(C, mechanical), has_thin_wall(C).
```

This bootstraps general rules from specific cases. The learned rules are inspectable and correctable, unlike a neural model.

### 5.3 Incremental Growth Model

The ontology grows with the system. A viable timeline:

| Phase | Scope | Rules Added | Outcome |
|-------|-------|-------------|---------|
| Week 1-2 | Formalize existing DFM rules as Datalog | \~100 rules (Layer 3) | $\rho$ implemented for DFM constraints |
| Week 3-4 | Add thermal coupling rules for mechanical domain | \~50 rules (Layers 1-2) | "Find all thermal contributors" works; $\mathcal{C}$ covers thermal |
| Week 5-6 | Add structural / vibration rules | \~50 rules (Layers 1-2) | Resonance and fatigue surfacing |
| Week 7-8 | Add electrical domain basics | \~50 rules (Layers 1-2) | Cross-domain checks (EMI, power integrity) |
| Ongoing | Each new decomposition run identifies gaps; rules added to close them | Incremental | $\mathcal{C}$ coverage grows with usage |

The closed-world assumption is a feature: if the ontology can't derive a check, it reports "no check found" rather than hallucinating one. That silence is the gap signal, triggering the gap detection mechanisms described next.


---

## 6. Rule Coverage Auditing and Gap Detection

The meta-problem: the formalization assumes $\mathcal{C}$ is given. But for physics, who ensures $\mathcal{C}$ is complete? This section describes how we audit the completeness of $\mathcal{C}$ for the physics domain.

### 6.1 Structural Completeness Checking

For each physical phenomenon in the ontology, verify that four types of rules exist:

| Required | Question | Gap If Missing |
|----------|----------|----------------|
| **Source rules** | "What generates this phenomenon?" | "We recognize thermal radiation but can't identify what emits it" |
| **Propagation rules** | "How does it travel through the system?" | "We know the actuator emits heat but can't trace where it goes" |
| **Effect rules** | "What does it damage or influence?" | "We trace heat to the mirror but don't know what it does to surface figure" |
| **Check rules** | "What verification detects the problem?" | "We know thermal distortion is possible but have no quantitative check" |

This audit is itself a Datalog query -- $\rho$ applied reflexively to the ontology itself:

```
gap(Phenomenon, no_source) :-
    phenomenon(Phenomenon), not source_rule(_, Phenomenon).
gap(Phenomenon, no_propagation) :-
    phenomenon(Phenomenon), source_rule(_, Phenomenon),
    not propagation_rule(Phenomenon, _).
gap(Phenomenon, no_check) :-
    phenomenon(Phenomenon), effect_rule(Phenomenon, _),
    not check_rule(_, Phenomenon).
```

Run after every ontology update. Output: a concrete list of gaps in $\mathcal{C}$ -- "thermal radiation has source + propagation rules but no check rules for ceramic materials."

### 6.2 Cross-Referencing Against Standards

Engineering standards (ASME Y14.5 for GD&T, IPC-2221 for PCB, MIL-STD-810 for environmental) enumerate required checks per component type. These are "gold standard" $C_v^{eligible}$ sets for specific component classes.

The audit:


1. Encode standard-mandated checks as Datalog facts
2. Run $\rho$ (the deductive system) on the same component type
3. Diff: checks in the standard but not derived by $\rho$ = gaps in $\mathcal{C}$
4. Checks derived but not in the standard = potentially novel coverage

### 6.3 Failure Mode Coverage

FMEA databases enumerate known failure modes per component class. For each:

```
failure_mode(mirror_mount, thermal_creep,
    "Material creep under sustained thermal load causes drift").
```

Audit query:

```
uncovered_failure(Component, Mode) :-
    failure_mode(Component, Mode, _),
    not detects_failure(_, Component, Mode).
```

FMEA databases represent decades of engineering experience with what actually goes wrong. Cross-referencing against them is something a symbolic system can do systematically that manual review cannot. Every uncovered failure mode is a gap in $\mathcal{C}$.

### 6.4 Adversarial Probing with LLMs

Use LLMs not as reasoning engines but as adversaries for auditing $\mathcal{C}$:


1. Give the LLM: component description + operating conditions + the $C_v^{eligible}$ set that $\rho$ derived
2. Ask: "What failure modes or physical effects could occur that are NOT covered by these checks?"
3. For each LLM suggestion, check if the ontology has rules that would catch it
4. If not: is this a real gap in $\mathcal{C}$ or an LLM hallucination? Domain expert decides
5. Confirmed gaps become new rules in the ontology

This is red-teaming $\mathcal{C}$. The LLM's value is associative breadth (it's been trained on failure reports, papers, etc.). The symbolic system's value is rigorous coverage verification. Together: stronger than either alone.


---

## 7. Scalability

### 7.1 Expected Scale

| Component | Estimated Size | Notes |
|-----------|----------------|-------|
| Base facts (component properties) | \~200K (10K components x 20 properties) | Moderate |
| Coupling edges | \~50K (sparse: most components don't couple) | Key driver of transitive closure |
| Transitive closure of coupling | \~500K (depends on graph structure) | Largest derived relation; critical for Section 11 global check |
| $C_v^{eligible}$ derivations | \~100K-1M (10-100 checks per component) | Output of $\rho$ |
| Total derived facts | \~2M           | Well within engine capabilities |

Souffle handles billions of facts. 2M is trivial. Scallop handles millions; for 10K components it's fine (seconds). Scalability is not a near-term concern.

Physical coupling is inherently sparse (thermal conduction requires contact, vibration requires rigid paths, EM requires proximity). Computing transitive closure per domain keeps the graph manageable:

```
% Separate closure per domain - much sparser than a single general closure
thermal_path(X, Z) :- thermal_path(X, Y), thermally_coupled(Y, Z).
vibration_path(X, Z) :- vibration_path(X, Y), mechanically_coupled(Y, Z).
```

### 7.2 Incremental Maintenance

Batch re-evaluation on every change is wasteful. When the formalization's feedback loop (Section 7) triggers a parameter change at node $v$, or when confidence gating (Section 6) triggers backtracking to re-resolve an ancestor, only the affected portion of the derivation should recompute.

**Differential Dataflow** (by Frank McSherry, used in the Materialize database) represents the Datalog computation as a dataflow graph and propagates changes incrementally. The cost of an update is proportional to the *size of the change*, not the *size of the database*.

This maps directly to the formalization's backtracking: when gating detects $\sigma_{\text{path}}(v) < \tau_{\text{global}}$ and the weakest ancestor is re-resolved, incremental evaluation recomputes only the affected subtree's $C_v^{eligible}$ and VERIFY results -- not the entire tree.

**Practical path**: Start with batch Souffle/Scallop (fast enough at our scale for seconds-latency re-evaluation). Migrate to Differential Dataflow when real-time responsiveness or 100K+ component scale requires it.


---

## 8. Handling Novel Physics

### 8.1 Taxonomy of Novelty

The formalization defines an **irreducible core** $C_v^{core}$ -- constraints that humans must seed because the system cannot derive them. For the physics domain, the irreducible core corresponds to novelty: how much of the physics at node $v$ is outside the ontology's coverage?

| Level | Description | Relation to $C_v^{core}$ | Example |
|-------|-------------|------------------------|---------|
| **N0** | Known phenomenon, known regime. Rules exist and are valid. | $c \in C_v^{derived}$ -- system handles it | Thermal conduction in aluminum at room temperature |
| **N1** | Known phenomenon, extrapolated regime. Rules exist but conditions exceed validated range. | $c \in C_v^{derived}$ but $\sigma < 1.0$ due to regime extrapolation | Thermal conduction in aluminum at 800 C (near melting) |
| **N2** | Known phenomenon, missing rules. Ontology recognizes the phenomenon but lacks check rules for this component type. | $c$ should be in $C_v^{eligible}$ but $\rho$ can't derive it -- detectable gap in $\mathcal{C}$ | Thermal radiation in EUV vacuum (radiation rules exist but not for this geometry) |
| **N3** | Analogous phenomenon. Something structurally similar exists. | $c \notin \mathcal{C}$ but analogs suggest it should be -- hypothesis, not verified | Tin debris contamination -- ontology has particle contamination but not metal vapor deposition |
| **N4** | Genuinely unknown. Nothing in the ontology covers this. | $c \notin \mathcal{C}$ and system cannot detect the gap -- part of the irreducible core $C_v^{core}$ | Quantum tunneling leakage at 2nm gate oxide |

### 8.2 Responses by Novelty Level

**N1 -- Regime extrapolation**: Rules carry validity regimes. When applied outside that regime, the constraint transitions from logic-typed to effectively semantic-typed -- the rule still runs but with degraded confidence:

```
check_result(mirror_mount, fourier_thermal,
    result: pass,
    confidence: 0.60,            % degraded from 1.0 (logic) to 0.60 (regime uncertainty)
    warning: "operating at 650C, rule validated only to 500C",
    suggested_action: "validate thermal model in 500-700C range").
```

In formalization terms: $\sigma_i$ for this constraint drops from 1.0 to 0.60, which feeds into $\sigma_v$ and propagates through $\sigma_{\text{path}}$. If the path product drops below $\tau_{\text{global}}$, gating (formalization Section 6) triggers backtracking.

**N2 -- Known gap**: Structural completeness checking (Section 6.1) catches this. At runtime, the system generates an Unknown node:

```
Unknown:
    phenomenon: thermal_radiation
    affected_contracts: [collector_optics_drift]
    description: "Thermal radiation effects recognized but
                  no quantitative check exists for this geometry"
    suggested_resolution: "Add thermal radiation check rule to ontology"
```

The Unknown blocks the parent contract from reaching SATISFIED -- the correct behavior. In formalization terms: the constraint exists in $\mathcal{C}$ but $\rho$ cannot evaluate it, so VERIFY returns UNKNOWN rather than SAT, contributing $\sigma_i = 0$ until the gap is filled.

**N3 -- Analogous phenomenon**: The system finds structurally similar phenomena in the ontology and suggests analogous checks, explicitly labeled as hypotheses (semantic-typed, $\sigma < 1.0$). Case-based reasoning within the Datalog framework.

**N4 -- Genuinely unknown**: The symbolic system cannot help directly. This is the irreducible core $C_v^{core}$ for the physics domain. Three fallback mechanisms:


1. **Simulation as discovery**: High-fidelity simulation (FEA, CFD) may reveal behaviors the ontology doesn't predict. Unexpected simulation results trigger "phenomenon not predicted by ontology -- manual analysis required."
2. **Literature-backed hypothesis generation**: LLMs + retrieval over scientific literature surface potentially relevant phenomena. These enter the system as hypotheses (semantic-typed) requiring expert validation before becoming logic-typed rules.
3. **Expert-in-the-loop**: For truly novel situations, the system flags clearly and routes to a human. The expert's response becomes a new rule in $\mathcal{C}$.

### 8.3 The Unknown Unknowns Budget

Accept that N4 gaps exist and account for them structurally. The formalization's confidence $\sigma_{\text{path}}(v) = \prod \sigma_u$ only accounts for *known* constraints. It doesn't account for constraints that should exist but don't (gaps in $\mathcal{C}$). We introduce a correction:

$$
\sigma_v' = \sigma_v \cdot (1 - \epsilon_\{\text\{novelty\}\}(v))
$$

where $\epsilon_{\text{novelty}}$ is the unknown-unknowns margin, proportional to the novelty level at node $v$:

| Novelty Category | $\epsilon_{\text{novelty}}$ | Rationale |
|------------------|---------------------------|-----------|
| Off-the-shelf, validated | 0.02                      | Decades of field experience; $\mathcal{C}$ is essentially complete |
| Adapted existing design | 0.10                      | Known unknowns in adaptation |
| Fully novel design | 0.25                      | Significant unknowns; $\mathcal{C}$ may be missing constraints |
| Novel physics regime | 0.40+                     | Operating beyond established knowledge |

The corrected path confidence becomes:

$$
\sigma_\{\text\{path\}\}'(v) = \prod_\{u \in \text\{root\} \to v\} \sigma_u \cdot (1 - \epsilon_\{\text\{novelty\}\}(u))
$$

This ensures novel subsystems propagate high uncertainty upward even when all *known* constraints pass. The margin decreases as experiments are run, the ontology grows, and $\mathcal{C}$ becomes more complete at that node.

**Three epistemic states the system must distinguish**:


1. **Known known**: "We checked thermal drift and it passes." --> $\sigma_i = 1.0$ (logic-typed), evidence node.
2. **Known unknown**: "We know thermal radiation matters but haven't checked it." --> Unknown node, $\sigma_i = 0$, explicit gap in $\mathcal{C}$.
3. **Unknown unknown**: "We don't know what we don't know." --> $\epsilon_{\text{novelty}} > 0$, novelty-proportional margin on $\sigma_v$.

Most engineering systems operate only in state 1 and fail silently in states 2 and 3. The symbolic layer's closed-world assumption enables honest treatment of all three.


---

## 9. Integration with Grafton Platform

Given the current codebase (Z3 for SAT, SHACL pipeline, hand-crafted DFM rules, confidence propagation), a phased integration mapped to the formalization's operations:

| Phase | Work | Formalization Operation Implemented | Outcome |
|-------|------|-------------------------------------|---------|
| **1. Encode existing rules** | Translate `verifier_core/rulepack/mech_tier0` rules to Datalog | $\rho$ for DFM constraints          | Existing checks become queryable, composable, auditable. Gap audit on $\mathcal{C}$ possible immediately. |
| **2. Physics coupling ontology** | Add \~100 thermal + structural rules from engineering handbooks | Populates $\mathcal{C}$ for thermal/structural domain | System auto-identifies all thermal/structural contributors. Demonstrates exhaustive $\rho$ without hand-crafting queries. |
| **3. Wire into verification pipeline** | Add Datalog evaluation as enhancement to V0/V1 checks. Scallop $\sigma_v$ feeds confidence propagation. | VERIFY_CSP for physics logic constraints (Section 5) | Check surfacing is automatic. Evidence carries provenance. $\sigma = 1.0$ for deductively verified constraints. |
| **4. Abductive repair via Z3** | Extend existing Z3 SAT checks to use Z3 Optimize for repair suggestions | Enhanced feedback for Section 7 loop | On UNSAT: "Change parameter Y by delta Z" becomes automatic. Accelerates convergence. |
| **5. Incremental evaluation** | Integrate incremental Datalog with bidirectional propagation | Efficient re-evaluation on backtrack (Section 6 gating) | Leaf change triggers targeted re-evaluation of $\sigma_{\text{path}}$, not full recomputation. |
| **6. Gap detection** | FMEA cross-referencing, standards cross-referencing, adversarial LLM probing | $\mathcal{C}$ completeness auditing | Systematic identification of missing physics constraints. |
| **Research horizon** | Scallop neural integration for fact extraction, ILP for rule learning, analog reasoning for N3 | Shrinking $C_v^{core}$ over time    | Self-improving ontology; irreducible core decreases with each decomposition run. |


---

## 10. Open Questions

**Ontology governance**: Who owns the rules in $\mathcal{C}$? What is the review process for adding, modifying, or deprecating a rule? How do we version the ontology alongside the codebase? A bad rule (e.g., a missing validity regime) silently produces wrong conclusions with $\sigma = 1.0$.

**Rule interaction testing**: Individual rules can be correct but compose incorrectly. How do we test that rule *combinations* produce expected $C_v^{eligible}$ sets? This is analogous to integration testing but for a rule base.

**Confidence calibration across levels**: The formalization uses $\sigma_{\text{path}}(v) = \prod \sigma_u$. A path confidence of 0.8 composed from hundreds of children means something very different from 0.8 on a single leaf. How do we ensure scores are interpretable and comparable? Does the product model even hold, or do we need a more sophisticated composition (e.g., Bayesian update, copulas for correlated constraints)?

**Abduction search space**: For complex UNSAT results, the space of possible repairs can be very large. Z3 Optimize handles continuous parameters well, but combinatorial repairs (change material AND geometry AND process) may be computationally expensive. What heuristics or decompositions keep abduction tractable?

**Neural-symbolic boundary management**: When the LLM populates a fact that turns out to be wrong, how does the error propagate through $\sigma_{\text{path}}$? Can we bound the "blast radius" of a bad neural-derived fact? Provenance tracking helps (we know which conclusions depend on semantic-typed inputs), but systematic error handling needs design.

**Existing ontology reuse**: Efforts like the Basic Formal Ontology (BFO), the Engineering Ontology (ENGONTO), and the Physics-based Simulation Ontology (PhysSim) exist in academic contexts. How much can we reuse vs needing to build from scratch for our specific $\rho$ requirements?

**Parsimony interaction**: The formalization selects $C_v^* = \arg\min |S|$ s.t. SAT decidable. Our Datalog layer derives the full $C_v^{eligible}$. How does parsimony apply on top -- do we evaluate all eligible constraints (cheap, since evaluation is batch) and only use parsimony for deciding which constraints to *propagate to children*? Or does parsimony apply to the Datalog evaluation itself?