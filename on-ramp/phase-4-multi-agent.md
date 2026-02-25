---
title: "Phase 4: Multi-Agent Architecture & Scaling"
description: "How the single-kernel system scales to handle multi-domain, multi-project, parallel execution through a DAG-based agent architecture"
---

## Why Single-Thread Doesn't Scale

Phases 1-3 describe a system that processes one goal tree sequentially. For simple goals this works. For complex systems — a semiconductor foundry with 10,000+ subsystems, or concurrent projects sharing manufacturing resources — sequential execution is too slow and a single resolver context is too narrow.

Phase 4 introduces multi-agent orchestration: multiple agents working in parallel on independent branches of the goal tree, coordinated by a thin orchestrator that handles scheduling, routing, and failure recovery.

## The Problem with the Linear Pipeline

Grafton's precursor was a linear pipeline: design generation (Steps 1-3) ran to completion before verification (Steps 4-6) began. Problems with this approach:

- **Adding a new domain** (thermal, electrical, cost) required re-plumbing the entire chain
- **Verification happened too late** — full design committed before any checks
- **No parallelism** — independent branches waited for each other
- **No comparative evaluation** — couldn't test different resolver strategies on the same goal

The multi-agent architecture solves all four by putting agents behind **stable interfaces**.

## The DAG Architecture

### Core Principle: Stable Interfaces, Pluggable Agents

Every agent in the system communicates through the same interface:

- **Agent context**: The goal node, constraints, budget, parent context, and accumulated evidence
- **Typed messages**: Structured envelopes (the same canonical format from Phase 1)
- **File workspaces**: Isolated directories for artifacts

Behind this interface, agents can be anything: an LLM, a SAT solver, a simulation tool, a human. The orchestrator doesn't care what's inside an agent — only that it accepts context and produces artifacts with confidence scores.

### The Thin Orchestrator

The orchestrator is deliberately thin — mostly programmatic logic, not LLM reasoning:

| Orchestrator responsibility | Implementation |
|---------------------------|---------------|
| DAG scheduling | Topological sort of goal tree |
| Agent routing | Capability matching from registry |
| Budget tracking | Ledger operations (deterministic) |
| Failure handling | Route to parent for re-decomposition |
| Confidence aggregation | Path product computation (deterministic) |

LLM reasoning is used only for:
- **Skill selection**: When multiple agents could handle a task, the orchestrator uses Claude Code to select
- **Error triage**: When an agent fails in an unexpected way, Claude Code diagnoses and recommends recovery

This split keeps the orchestrator fast and predictable. Expensive LLM calls happen inside agents, not in scheduling.

## Agent Types

### Decomposition Agent (recursive)

One agent type handles ALL hierarchy levels. A decomposition agent:

1. Receives a goal node with constraints
2. Analyzes whether to decompose further or resolve directly
3. If decomposing: produces child goals with inherited constraints and contracts
4. Spawns child agents (decomposition or domain-specific)
5. Gathers results from children
6. Verifies cross-child consistency (sibling constraints jointly satisfiable)
7. Propagates evidence upward

The recursive structure means the same agent code handles "decompose a foundry into subsystems" and "decompose a bracket into manufacturing steps."

### Domain-Specific Resolvers

Agents that solve specific types of problems:

| Agent | What it does | Sigma |
|-------|-------------|-------|
| LLM Resolver | Code generation, design generation, semantic tasks | &lt; 1.0 |
| Code Executor | Runs code, tests, scripts | 1.0 (pass/fail) |
| SAT/SMT Solver | Formal constraint checking, abductive repair | 1.0 |
| Simulation Runner | FEA, thermal simulation, signal integrity | Depends on model fidelity |
| CAD Kernel | Geometry operations, B-Rep analysis | 1.0 |
| VLM Inspector | Visual geometry checks, anomaly detection | &lt; 1.0 |
| Knowledge Lookup | Queries ontologies, material databases | 1.0 |

### Verifiers

Agents that check outputs against constraints. Map to the four-layer verification stack from Phase 2:

1. **Ontology verifier**: Runs Datalog rules to discover and check constraints
2. **Formal verifier**: Z3 for abductive repair
3. **Test verifier**: Executes test suites
4. **Semantic verifier**: LLM judgment with structured rubrics

### Aggregators

Agents that roll up evidence from children to parents. Implement the confidence propagation model from Phase 2:

- Collect per-constraint sigma values from child agents
- Compute node sigma (minimum across constraints)
- Compute path sigma (product along root → node)
- Apply gating thresholds
- Generate uncertainty reports (top unknowns, VOI ranking)

### Schedulers

Agents that coordinate execution waves. Handle:

- **Wave-parallel execution**: Independent branches run simultaneously
- **Dependency resolution**: Coupled branches execute in topological order
- **Resource allocation**: Budget distribution across parallel agents
- **Failure recovery**: When a branch fails, determine whether to retry, re-decompose, or escalate

## Wave-Parallel Execution

The goal tree naturally contains both independent and dependent branches. Wave-parallel execution exploits this:

```
Wave 1: [Branch A, Branch B, Branch C]    ← independent, run in parallel
         |            |           |
         v            v           v
Wave 2: [Branch A.1, Branch B.1]           ← A.1 depends on A, B.1 on B
         |            |                       C completed in Wave 1
         v            v
Wave 3: [Integration of A + B + C]         ← depends on all branches
```

**Topological ordering** determines which branches can run in parallel. **Budget allocation** distributes resources across parallel branches proportionally (with reserves for retries).

**Failure handling**: If Branch B fails in Wave 1, Branches A and C continue. Branch B is retried or re-decomposed. Wave 2 starts when all Wave 1 dependencies are met (A and C may proceed to Wave 2 while B is still being resolved).

## Stable Interfaces Enable Evolution

The stable interface design means the system can evolve without re-plumbing:

### Plug In New Verification Layers

Adding FEA verification for structural analysis:

1. Register a new agent type: `fea-verifier` with capabilities `["structural_analysis", "stress_check", "deflection_check"]`
2. It accepts the standard agent context (geometry artifact, material constraints, load constraints)
3. It produces standard verification results (per-constraint sigma, violation traces)
4. The orchestrator routes structural verification requests to it automatically

No changes to the orchestrator, decomposition agents, or any other agent.

### Swap Decomposition Strategies

Different domains may benefit from different decomposition heuristics:

- **Functional decomposition**: Break by function (power, cooling, structural)
- **Physical decomposition**: Break by physical location (left wing, right wing)
- **Process decomposition**: Break by manufacturing process (machined parts, cast parts, purchased parts)

Each strategy is a different decomposition agent. The orchestrator selects based on goal type and domain. Multiple strategies can be tested on the same goal for comparative evaluation.

### Comparative Evaluation

The stable interface enables running the same goal through different configurations:

```
Same Goal → Config A (aggressive decomposition, minimal verification)
          → Config B (conservative decomposition, full verification)
          → Config C (domain-expert decomposition heuristics)
```

Compare: cost, time, final confidence, number of HIRE escalations. This is how the system learns which strategies work for which domains.

## Evidence Propagation

Confidence flows upward through the agent hierarchy:

1. **Leaf agents** produce artifacts with per-constraint sigma
2. **Parent agents** aggregate child sigma values using the noisy-AND model (from Phase 2's hypergraph uncertainty)
3. **Aggregators** compute path sigma and apply gating
4. **Root agent** reports system-level confidence: the minimum path sigma across all leaves

At each aggregation point, the system also computes:
- **Top unknowns**: Which constraints have the highest epistemic uncertainty
- **Top blockers**: Which constraints have the lowest sigma
- **VOI ranking**: Where additional work would most improve system confidence

## Cross-Project Coordination

When multiple projects run concurrently:

### Shared Registry

All projects share the same resolver registry. A thermal analysis verifier built for the foundry project is available to the aircraft project. DFM rules for aluminum machining apply everywhere aluminum is used.

### Shared Manufacturing Graph

Manufacturing resources (machines, facilities, materials) are nodes in a shared graph. The scheduler considers cross-project resource contention:

- CNC machine X is used by foundry (30% capacity) and aircraft (50% capacity) — 20% remains
- Surface treatment facility has a 2-week lead time — schedule projects to avoid conflicts
- Shared material orders reduce per-unit cost

### When to Build Custom Manufacturing Tools

The system evaluates whether building a custom tool (resolver, fixture, test rig) is justified:

- **Single-project use**: Build only if ROI within this project's budget
- **Multi-project use**: Amortize cost across projects; lower threshold for building
- **Registry check**: Does a similar tool already exist? Can it be adapted?

## Background You Need

**DAG (Directed Acyclic Graph)**: A graph where edges have direction and no cycles exist. Topological sort gives a valid execution order. Used here for goal tree scheduling — independent branches can execute in parallel.

**Distributed Systems**: Systems where computation happens across multiple processes/machines. Key concerns: coordination, consistency, failure handling, message passing.

**Message Passing**: Agents communicate by sending structured messages rather than sharing state. This enables loose coupling — agents don't need to know each other's internals.

**Topological Sort**: An ordering of graph nodes such that every edge goes from earlier to later in the ordering. Used to determine which branches can execute in parallel (no edges between them) vs. must execute sequentially (dependency edges).

## Deep Dive

- [Multi-Agent Architecture](/graftonlab/multi-agent-architecture) — full architectural blueprint with interface definitions and scaling analysis
- [Constraint-Driven Problem Solving](/graftonlab/constraint-driven-problem-solving) — the philosophical foundation for how problems are formulated and solved
