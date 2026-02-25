---
title: "Phase 2: Verification & Reasoning"
description: "How the system moves from 'LLM says it's correct' to 'we can prove it's correct' — and how it handles uncertainty"
---

## Why Verification Matters

In Phase 1, the kernel relies heavily on Claude Code for both resolution and verification. This means confidence is capped at semantic levels — the LLM's judgment, which is useful but not provable. Phase 2 introduces deterministic verification layers that push sigma toward 1.0 wherever the constraint type allows it.

The goal: **use formal methods (sigma = 1.0) wherever possible, fall back to LLM judgment (sigma &lt; 1.0) only for constraints that can't be formally expressed.**

This matters because confidence is multiplicative along paths. If every node on a 5-level path has sigma = 0.85 (semantic), the path confidence is 0.85^5 = 0.44 — below any reasonable threshold. Replace three of those with deterministic checks (sigma = 1.0) and the path confidence becomes 1.0 * 1.0 * 1.0 * 0.85 * 0.85 = 0.72.

## The Four Verification Layers

Verification in Grafton is organized as four layers, from cheapest and most deterministic to most expensive and most flexible:

### Layer 1: Domain Ontology (Datalog Rules)

**What it is**: A set of domain-specific rules encoded in Datalog — a declarative logic language where you define facts and rules, and the engine derives all consequences.

**What it does**: Given the current design state, the ontology identifies ALL constraints that apply. This is constraint discovery — making implicit requirements explicit.

**Example** (thermal domain):

```
% Facts (from the design)
material(heatsink, aluminum_6061).
thermal_conductivity(aluminum_6061, 167).
power_dissipation(chip, 45).
contact_area(heatsink, chip, 0.0012).

% Rules (from thermal physics)
heat_flux(Part, Source, Q) :-
    power_dissipation(Source, P),
    contact_area(Part, Source, A),
    Q = P / A.

thermal_resistance(Part, R) :-
    material(Part, Mat),
    thermal_conductivity(Mat, K),
    thickness(Part, T),
    contact_area(Part, _, A),
    R = T / (K * A).

margin(Part, M) :-
    max_temperature(Part, Tmax),
    ambient_temperature(Tamb),
    thermal_resistance(Part, R),
    heat_flux(Part, _, Q),
    M = Tmax - Tamb - (Q * R).

violation(Part, "thermal_margin", M) :-
    margin(Part, M), M < 0.
```

The Datalog engine forward-chains all facts and rules to exhaustion. If `violation(heatsink, "thermal_margin", -5.2)` is derived, the system knows the heatsink design violates thermal margin by 5.2 degrees — before any expensive simulation runs.

**Key property**: Datalog is **complete** — it finds ALL derivable facts. If a constraint applies, Datalog will surface it. This is sigma = 1.0 discovery.

### Layer 2: Constraint Checking (Datalog Engine)

Same engine as Layer 1, but focused on checking whether specific constraints are satisfied rather than discovering new ones. Given a set of constraints and the current design state, run the rules and check each constraint's status.

**Sigma**: 1.0 for any constraint expressible as a Datalog query.

### Layer 3: Abductive Repair (Z3 SMT Solver)

**What it is**: When constraints are violated, you don't just want to know WHAT failed — you want to know the MINIMUM CHANGE that would fix it.

**What Z3 does**: Takes the violated constraint set, finds the unsatisfiable core (the specific subset of constraints that conflict), and computes the minimum relaxation that makes the system satisfiable.

**Example**: A stiffening rib has aspect ratio 10:1, but the maximum is 8:1.

Z3 analysis:
```
UNSAT core: {rib_height = 20mm, rib_thickness = 2mm, aspect_ratio <= 8:1}

Minimum repairs (ranked by impact):
  1. Reduce rib_height to 16mm (aspect ratio → 8:1)  [cost: +$0]
  2. Increase rib_thickness to 2.5mm (aspect ratio → 8:1)  [cost: +$0.50 material]
  3. Relax aspect_ratio limit to 10:1  [requires: structural analysis justification]
```

The resolver receives these specific repair options rather than a vague "rib design failed." This dramatically improves retry success rate.

**Sigma**: 1.0 for the repair computation itself (Z3 is a formal solver). The selected repair still needs verification.

### Layer 4: LLM Fallback (Semantic Judgment)

**What it is**: For constraints that can't be expressed in formal logic — "the design should look clean," "the error messages should be helpful," "this approach is reasonable for the team's skill level" — Claude Code evaluates using a structured rubric.

**Sigma**: Calibrated between 0 and 1. The rubric makes the evaluation reproducible and traceable, but it's fundamentally judgment, not proof.

**When used**: Only when Layers 1-3 can't express the constraint. The system actively tries to formalize semantic constraints into logic constraints over time (e.g., "looks clean" might eventually decompose into measurable properties like "symmetry score >= 0.8" + "no overlapping elements" + "consistent spacing").

## How the Layers Work Together

```
Design output arrives for verification
         |
         v
Layer 1: Ontology discovers ALL applicable constraints
         |
         v
Layer 2: Datalog checks each constraint
         |
   +-----+-----+
   |             |
   v             v
 All SAT?      Some UNSAT
   |             |
   v             v
 Done          Layer 3: Z3 finds minimum repair
 (sigma=1.0)     |
                 v
              Repair suggestions → resolver retries
                 |
                 v
              Any constraints not formalizable?
                 |
                 v
              Layer 4: LLM judges (sigma < 1.0)
```

## Confidence Quantification

Phase 2 also introduces rigorous uncertainty tracking, going beyond simple sigma floats.

### The Unit of Uncertainty: Atomic Clauses

Every verifiable statement in the system is an **atomic clause** — e.g., "hole_diameter is within 6.6mm +/- 0.1mm." Clauses are the finest-grained unit. Everything else aggregates from clauses:

- **Evidence nodes** feed clauses (a test result, a simulation output, a measurement)
- **Contracts** aggregate clauses (a set of clauses that must all hold)
- **Requirements** aggregate contracts
- **Goals** aggregate requirements

### Evidence Types

Three kinds of evidence feed into clause confidence:

| Type | What it answers | Example |
|------|----------------|---------|
| Verification | "Does the tool compute correctly?" | Unit test passes (sigma = 1.0) |
| Validation | "Does the tool model reality?" | Simulation matches physical measurement |
| Calibration | "Does the tool's score map to real probability?" | LLM says sigma = 0.8; historically, 80% of such claims are correct |

### Missing Evidence

A critical design choice: missing evidence does NOT mean failure. It means **epistemic inflation** — uncertainty increases, but the system doesn't auto-reject. This is represented explicitly as an "unknown" node in the evidence graph.

This matters because early in a design, most evidence is missing. Auto-failing on missing evidence would block all progress. Instead, the system tracks what's known, what's unknown, and what the highest-value next measurement would be.

### Decision Gating

A node passes verification when ALL of these hold:

1. **P(pass) >= tau_p**: Probability of satisfying all constraints exceeds threshold
2. **Uncertainty &lt;= tau_u**: We're confident enough in our estimate (not just that the estimate is high)
3. **Evidence coverage**: Sufficient evidence exists for the claim

### Value of Information (VOI)

When a node fails gating, the system asks: "What is the highest-value next action?" This is computed as:

```
VOI(action) = impact_on_goal * epistemic_uncertainty_of_target
```

The action that most reduces uncertainty on the most impactful unknown gets priority. This prevents the system from wasting effort on well-understood areas while critical unknowns remain.

## Autonomous Capability Building for Verification

When the system encounters a domain for the first time (e.g., thermal analysis), it has no Datalog rules for that domain. Phase 2's growth mechanism:

1. **Detect gap**: Verification needed for thermal constraints, no thermal ontology exists
2. **Build ontology as subgoal**: "Construct Datalog rules for thermal analysis of heat sinks" — uses the same kernel loop (decompose, resolve, verify)
3. **Claude Code drafts rules**: Using physics knowledge, generates initial Datalog ruleset
4. **Verify the verifier**: Test the rules against known cases (analytically solved problems)
5. **Register**: Persist the ontology in the registry for all future thermal goals

This is the system bootstrapping its own verification capabilities. The first thermal goal is expensive (rules must be built). Every subsequent thermal goal is cheaper (rules exist and sigma = 1.0).

## Background You Need

**Datalog**: A declarative logic language. You state facts (`parent(tom, bob).`) and rules (`ancestor(X, Y) :- parent(X, Y).` and `ancestor(X, Y) :- parent(X, Z), ancestor(Z, Y).`). The engine derives all consequences. It always terminates and finds everything derivable. If you know SQL, think of Datalog as recursive SQL with cleaner syntax.

**SMT Solvers (Z3)**: Satisfiability Modulo Theories. Given a set of logical constraints over variables, Z3 determines whether any assignment satisfies all constraints. If unsatisfiable, it identifies the minimal conflicting subset (unsat core). Used here for abductive repair — working backward from a violation to the minimum fix.

**Bayesian Inference**: Updating beliefs (probabilities) as new evidence arrives. Prior belief + new evidence = posterior belief. Used here for fusing evidence from multiple sources (tests, simulations, measurements) into a single confidence estimate.

## Deep Dive

- [Symbolic Reasoning for Verification](/graftonlab/symbolic-reasoning-verification) — the four-layer verification architecture with domain examples
- [Hypergraph Uncertainty](/graftonlab/hypergraph-uncertainty) — clause-level uncertainty, evidence fusion, VOI-driven regeneration
- [Domain-Agnostic Goal Decomposition](/graftonlab/domain-agnostic-goal-decomposition) — how the system builds ontologies and resolvers autonomously
- [Constraint-Driven Problem Solving](/graftonlab/constraint-driven-problem-solving) — philosophical foundation for constraint handling
