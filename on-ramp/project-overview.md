---
title: "Project Overview"
description: "What Grafton is, why it exists, and what it does — your starting point"
---

## What is Grafton?

Grafton is a **hierarchical constraint-driven planning and execution platform**. It takes a goal — stated in any form (text, sketch, partial spec) — and recursively breaks it down into a tree of solvable subproblems. For each subproblem, it selects the best available solver, verifies the output against constraints, tracks confidence, and assembles the results back up into a complete solution.

The same framework applies whether the goal is "build a CLI tool that sorts CSV files" or "design a 2nm semiconductor fabrication facility."

## The Problem Grafton Solves

Complex engineering projects — semiconductor fabs, aircraft, precision instruments — require thousands of interconnected decisions across multiple domains (structural, thermal, electrical, manufacturing, cost). Today these decisions are made manually, sequentially, and expensively:

1. A design gets committed across many parameters
2. Downstream verification discovers a violation (e.g., a pocket requires 5-axis CNC but the budget only allows 3-axis)
3. The entire design cycles back through multiple stages
4. Manufacturing feasibility surfaces too late to influence design cheaply

This sequential commit-then-verify-then-rework loop is where most time and money is lost. Grafton eliminates it by verifying constraints **at every step**, growing its own capabilities when it encounters domains it hasn't seen before, and treating manufacturing as a first-class constraint alongside physics and geometry.

## The Core Loop

Every operation in Grafton follows the same recursive pattern:

```
Goal + Constraints
       |
       v
   DECOMPOSE into subgoals
       |
       v
   For each subgoal:
       SELECT best resolver
       RESOLVE (produce output)
       VERIFY against constraints
       |
       +---> SAT (constraints satisfied) --> proceed or assemble
       |
       +---> UNSAT --> feedback loop:
                1. TRY:   retry with violation context
                2. BUILD: construct a new resolver as a subgoal
                3. HIRE:  ask a human to build the resolver
```

This pattern applies at every level of the hierarchy — from the top-level system goal down to an individual part feature.

## Key Properties

**Domain-agnostic.** The framework doesn't know or care whether it's solving a software problem or a mechanical design problem. Domain knowledge enters through constraints and resolvers, not through the framework itself.

**Self-growing.** When the system encounters a problem it can't solve with existing capabilities, it doesn't just fail — it asks "what capability am I missing?" and tries to build that capability as a subgoal. Each solved problem expands the system's resolver set. Capabilities only grow; they never shrink.

**Confidence-tracked.** Every output carries a quantified confidence value (sigma). Logic checks (formal proofs, test passes) give sigma = 1.0. Semantic checks (LLM judgment) give calibrated sigma between 0 and 1. Confidence multiplies along the path from root to leaf. If a path becomes too uncertain, the system backtracks to the weakest ancestor and re-resolves.

**Manufacturing-aware.** Cost, tooling, lead time, and manufacturing process constraints are tracked at every design step — not discovered after the design is committed.

**Parsimonious.** At every level, the system selects the *minimum* set of constraints that makes the problem decidable. This avoids over-constraining (which leads to infeasibility) and under-constraining (which leads to underdetermined solutions).

## Where We're Headed

The project is structured in phases that progressively expand the system's scope:

- **Phase 1 — The Kernel**: A minimum viable system that handles software-only goals. Boots from Claude Code + Codex, grows its resolver set through use.
- **Phase 2 — Verification and Reasoning**: Deterministic verification layers (Datalog, Z3) that replace LLM judgment where possible, plus rigorous uncertainty quantification.
- **Phase 3 — Physical Design and Manufacturing**: Extend to mechanical/hardware design with step-level verification and manufacturing constraint tracking.
- **Phase 4 — Multi-Agent Architecture**: Scale to multi-domain, multi-project execution with parallel agents and a DAG-based orchestrator.

Each phase is covered in its own on-ramp document.

## How to Use This On-Ramp

1. **Start here** — you've done that
2. **Read [Core Concepts and Glossary](/on-ramp/core-concepts)** — learn the vocabulary
3. **Read the phase that matches your current work** — each phase doc contains the background you need to contribute
4. **Use the [Reading Guide](/on-ramp/reading-guide)** — maps your role and work to the deeper design documents in the Grafton Lab section
