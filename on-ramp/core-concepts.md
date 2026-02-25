---
title: "Core Concepts & Glossary"
description: "Every key term and concept you'll encounter across the Grafton design documents, explained with examples"
---

## The Building Blocks

These are the fundamental abstractions that everything in Grafton is built from. Understanding these lets you read any design document in the Grafton Lab section.

### Goal

What the system is trying to achieve. A goal can be expressed in any modality — natural language text, a sketch, a reference to an existing artifact, a partial specification. The system's job is to take a goal and produce a verified output that satisfies all constraints.

**Example**: "Design a mounting bracket for a 5kg load, CNC machined from aluminum 6061."

Goals form a tree. A root goal decomposes into child goals, which may decompose further, until you reach leaf goals that can be directly resolved.

### Constraint

A condition that must hold for a goal's output to be acceptable. Every constraint has a **type** that determines how it gets verified:

- **Logic constraints**: Formally verifiable. The system can check these with certainty (sigma = 1.0). Examples: "wall thickness >= 1mm", "all tests pass", "hole diameter / depth ratio &lt;= 10."
- **Semantic constraints**: Require judgment. Checked by an LLM or human, yielding calibrated confidence (sigma between 0 and 1). Examples: "the design should be aesthetically clean", "the code should be readable."

Every constraint also tracks **provenance** — where it came from (human-seeded, resolver-derived, or inherited from a parent goal) and which domain it belongs to (thermal, structural, cost, or universal).

### Irreducible Core

The minimum set of constraints a human must provide for a goal to be decidable. Without these, the goal is underdetermined — the system can't distinguish good solutions from bad ones.

**Example**: For "build me a website," the irreducible core might be: target audience, primary action the user should take, and technology stack. Without these, any website is equally valid.

The irreducible core is largest at the root goal (the system knows nothing yet) and shrinks as you go deeper (the system derives constraints from context and domain knowledge).

### Resolver

Any process that takes constraints + input and produces output. This is the system's broadest abstraction — a resolver can be:

- An LLM (Claude Code generating code or designs)
- A code executor (running tests, scripts)
- A SAT/SMT solver (Z3, formal verification)
- A simulation (FEA, thermal analysis)
- A human (domain expert providing judgment or building something)
- A knowledge lookup (querying a database or ontology)

Each resolver has a **capability profile** (what it can do), a **cost profile** (per-invocation or per-token costs, latency), and a **quality vector** scoring it on six criteria: completeness, executability, optimality, representation quality, generalization, and efficiency.

**Key insight**: Decomposition is itself a resolver call. The process of breaking a goal into subgoals has the same signature: takes constraints + goal, produces subgoals.

### Verifier

A specialized resolver that checks whether output satisfies constraints. Returns:

- **Verdict**: SAT (satisfies all constraints) or UNSAT (at least one violation)
- **Confidence (sigma)**: How certain the verdict is
- **Trace**: Per-constraint results with violation details and repair suggestions

At boot, Claude Code is the only verifier (semantic, sigma &lt; 1.0). As the system grows, it builds deterministic verifiers — Datalog rule sets (sigma = 1.0), Z3 checkers, test suites — that replace LLM judgment where possible.

### Artifact

Everything the system produces wraps in an artifact envelope. An artifact has a type (code, design, ontology rule, resolver, data), format (Python, JSON, STEP, Datalog), content, and **lineage** — which resolver produced it, under which constraints, from which inputs. Lineage enables traceability: you can always trace an output back to the goal, constraints, and resolver that created it.

### Confidence (sigma)

A float between 0 and 1 quantifying certainty. There are three levels:

- **Constraint-level sigma**: How confident is the system that a specific constraint is satisfied? Logic checks yield sigma = 1.0. Semantic checks yield calibrated sigma.
- **Node-level sigma**: The minimum sigma across all constraints at a goal node. The weakest constraint sets the ceiling.
- **Path-level sigma**: The product of sigma values along the path from root to a node. Represents cumulative certainty. If any ancestor is uncertain, that uncertainty propagates to all descendants.

**Gating**: If path sigma drops below a global threshold (tau_global), the system halts decomposition at that branch and backtracks to the weakest ancestor.

### Registry

The persistent store of all resolvers, constraints, artifacts, and their relationships. The registry is a hierarchical hypergraph — nodes are entities (resolvers, constraints, artifacts, goals), edges represent relationships (provides, requires, composed_of, built_by, verified_by).

**Critical property**: The registry only grows. Capabilities accumulate with each solved goal. The system never asks for the same capability twice — once a resolver is built (by the system or a human), it persists for all future goals.

### Parsimony

The principle of selecting the minimum viable constraint set at each goal node. Formally: find the smallest set S of constraints such that SAT(G, S) is decidable. Adding more constraints than necessary risks over-constraining (making the problem infeasible). Leaving out essential constraints risks under-constraining (accepting bad solutions).

### Contract

At each level of the hierarchy, a subsystem makes guarantees given certain assumptions. The parent's guarantee becomes the child's assumption. Contracts cascade downward:

| Level | Guarantee |
|-------|-----------|
| System | "Lithography resolution &lt;= 2nm" |
| Subsystem | "EUV source power >= 250W at 13.5nm" |
| Component | "Collector reflectivity >= 65% across aperture" |
| Part | "Mirror surface roughness &lt;= 0.1nm RMS" |
| Atomic part | "Mount thermal drift &lt;= 0.5nm/K over operating range" |

### Ontology

Domain-specific rules encoded for deterministic reasoning. Grafton uses a three-layer ontology system:

- **Layer 0**: Capability and knowledge tracking (what the system can do, what's blocked)
- **Layer 1**: Domain discovery (identifying which domains are relevant to a goal)
- **Layer 2**: Domain-specific rules (physics equations, DFM rules, manufacturing constraints)

Ontologies are encoded in Datalog and built on demand — the first time the system encounters a domain, it constructs the relevant rules as a subgoal.

## The Growth Mechanism

### TRY, BUILD, HIRE

The three-level escalation protocol that drives system growth:

1. **TRY**: Use existing resolvers. On failure, the feedback loop provides specific violation traces so the resolver can correct its output. Retry up to T_max times.

2. **BUILD**: When retries are exhausted, construct a new resolver as a subgoal. This is recursive — the same kernel that solves user goals also solves "build me a resolver for X." The new resolver is persisted in the registry.

3. **HIRE**: When building a resolver exceeds the compute budget, route to a human. The human-built resolver is registered and persists. The system never asks a human for the same capability twice.

**Example**: The system needs to verify thermal constraints. No thermal verifier exists (TRY fails). It builds a Datalog rule set for thermal checking as a subgoal (BUILD). If thermal physics is too complex for automated rule generation, it asks a domain expert to write the rules (HIRE). Either way, the thermal verifier persists for all future thermal goals.

### Why This Works

Each goal solved expands the resolver set. Fewer goals hit BUILD. Fewer hit HIRE. The system's effective quality is monotonically non-decreasing — it can only get better with use.

## The Verification Stack

### Four Layers (from cheapest to most expensive)

| Layer | Tool | Sigma | When Used |
|-------|------|-------|-----------|
| 1. Domain Ontology | Datalog rules | 1.0 | Always — finds ALL applicable constraints |
| 2. Constraint Checking | Datalog engine | 1.0 | Always — forward-chains to check constraints exhaustively |
| 3. Abductive Repair | Z3 SMT solver | 1.0 | When constraints violated — finds minimum fix |
| 4. Semantic Fallback | LLM | 0.0-1.0 | When formal methods can't express the constraint |

**Principle**: Use deterministic verification (sigma = 1.0) wherever possible. Fall back to probabilistic (LLM) only for constraints that can't be formally expressed.

## Quick Reference Table

| Term | One-line definition |
|------|-------------------|
| Goal | What to achieve (any modality) |
| Constraint | Condition that must hold (logic or semantic) |
| Irreducible Core | Minimum human-provided constraints |
| Resolver | Anything that produces output from constraints + input |
| Verifier | Resolver that checks output against constraints |
| Artifact | Output envelope with lineage |
| Sigma (confidence) | Certainty score, 0-1 |
| Registry | Persistent, growing store of all capabilities |
| Parsimony | Minimum viable constraint set |
| Contract | Guarantee/assumption pair at a hierarchy level |
| Ontology | Domain rules in Datalog |
| TRY/BUILD/HIRE | Escalation protocol for growing capabilities |
| Gating | Halt if path confidence drops below threshold |
