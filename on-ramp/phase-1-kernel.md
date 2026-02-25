---
title: "Phase 1: Building the Kernel"
description: "Everything you need to understand and contribute to the MVP kernel — the minimum system that can accept goals, solve them, and grow"
---

## What the Kernel Is

The kernel is the minimum system that can:

1. Accept a goal with constraints
2. Decompose it into subgoals
3. Select a resolver for each subgoal
4. Resolve (produce output)
5. Verify the output satisfies constraints
6. Learn from the experience
7. Grow its own capabilities when stuck

It boots from two resolvers — Claude Code (reasoning, code generation, semantic verification) and Codex Tools (code execution, file ops, testing) — and a self-model that tracks what the system can do. Everything else is built on demand.

**MVP scope**: Software-only goals. 10-minute wall-time cap. Confidence thresholds: tau_local = 0.70 (per-node), tau_global = 0.50 (path minimum). Steps 1-9 of 13 total system steps.

## The Six Axioms

These are what MUST exist at boot. Everything else grows from them.

| Axiom | What it is | Why it can't be removed |
|-------|-----------|------------------------|
| At least one resolver | Claude Code at boot | Can't solve anything without a solver |
| Goal interface | Accepts goals in any modality (text, sketch, spec) | System needs to ingest problems |
| Constraint typing | Every constraint is typed as logic or semantic | Verification confidence depends on type |
| Verification function | VERIFY(input, output, constraints) returns verdict + sigma | Can't confirm outputs satisfy constraints without this |
| Persistent registry | Stores all resolvers, constraints, artifacts | Without persistence, every goal restarts from zero |
| Self-model | System knows its own capabilities, costs, state | Needed for resolver selection and grounding |

**Not axioms** (built on demand): domain ontologies, physical resolvers, decomposition heuristics, the full module stack. The kernel is the seed.

## Boot State

At system start, the registry contains exactly three resolvers:

**Self-Model** (`self-model-0`): Reports system state, enumerates capabilities, describes architecture, estimates costs. Zero cost per invocation.

**Claude Code** (`claude-code-0`): The primary reasoning resolver. Handles decomposition, code generation, constraint extraction, semantic verification, feedback generation, orchestration. Cost: ~$0.015/token. Quality: high on representation and generalization, moderate on completeness and executability, lower on optimality and efficiency.

**Codex Tools** (`codex-tools-0`): The execution resolver. Runs code, manages files, executes tests, installs packages. Cost: ~$0.01/invocation. Quality: high on executability and efficiency, lower on representation and generalization.

## The Algorithm: Step by Step

Here's what happens when a goal enters the system, traced through a concrete example.

**Example goal**: "Build a Python CLI tool that reads a CSV file and outputs the top 10 rows sorted by a user-specified column."

### Step 1: Goal Extraction

The system (via Claude Code in interactive mode) dialogues with the human to extract:
- The goal description (any modality)
- The irreducible core constraints — the minimum the human must provide

For our example:
- **Goal**: CLI tool, CSV sorting
- **Core constraints**: Python 3.10+, must handle files up to 1GB, must validate column exists, output to stdout

### Step 2: Constraint Identification

Claude Code identifies ALL potentially relevant constraints using domain knowledge:
- Logic: "CLI accepts --file and --column arguments" (verifiable by test)
- Logic: "exits with error code 1 if column not found" (verifiable by test)
- Semantic: "error messages should be helpful" (LLM judgment)
- Logic: "handles 1GB file without memory error" (verifiable by test)
- Derived: "must use streaming/chunked reads for large files" (derived from 1GB constraint)

### Step 3: Parsimonious Selection

Select the MINIMUM set of constraints that makes the problem decidable. Drop redundant or overly specific constraints. Keep only those where removing any one would make the goal underdetermined.

### Step 4: Resolver Selection

Query the registry for resolvers that can handle these constraints. At boot, Claude Code is the only candidate for code generation. The system scores it against the six quality criteria and selects it (trivially, since it's the only option).

### Step 5: Resolution

Claude Code generates the code artifact. The output is wrapped in an Artifact envelope with full lineage (which resolver, which constraints, which inputs).

### Step 6: Verification

Each constraint is checked individually:
- **Logic constraints**: Codex Tools runs the test suite. Pass = sigma 1.0.
- **Semantic constraints**: Claude Code evaluates "are error messages helpful?" with a rubric. Returns sigma 0.85.
- **Node sigma**: min(1.0, 1.0, 0.85, 1.0) = 0.85. Above tau_local (0.70). Pass.

### Step 7: Feedback Loop

If verification returns UNSAT, the violation trace (which constraints failed, why, suggested repairs) goes back to the resolver. The resolver retries with this specific context. Up to T_max retries.

### Step 8: Decomposition

If the goal is too complex for direct resolution, decompose into subgoals. For our CLI tool, this might not be needed. For a larger goal like "build a web application with auth, database, and API," decomposition would produce subgoals for each component.

Decomposition is itself a resolver call — Claude Code takes the goal + constraints and produces child goals with inherited constraints.

### Step 9: Leaf Termination

When a goal is resolved and verified (SAT, sigma above threshold), it terminates. The artifact is registered. The resolution history is logged for the knowledge graph.

## The Growth Engine

What happens when the system gets stuck?

### Level 1: TRY

Use existing resolvers. The feedback loop gives specific violation traces: "test_large_file failed — MemoryError at line 42." The resolver retries with this context. Up to T_max attempts.

### Level 2: BUILD

When retries are exhausted and no resolver in the registry can handle the problem, construct a new resolver as a subgoal. This is recursive — the same kernel that solves user goals also solves "build me a resolver for X."

**Example**: The system needs to verify that generated code handles edge cases. No edge-case verifier exists. It creates a subgoal: "Build a property-based test generator for Python CLI tools." Claude Code generates the test generator. Codex Tools runs it to verify it works. The new resolver is persisted in the registry.

### Level 3: HIRE

When building a resolver exceeds the compute budget, route to a human. The human is presented with:
- What capability is needed
- What was tried and why it failed
- Budget remaining
- Similar past resolutions (from the knowledge graph)

The human-built resolver is registered and persists. The system deduplicates capability requests — it never asks a human for the same thing twice.

## Build Layers

The kernel is organized into dependency layers. This is how it gets implemented:

### Layer 0: Storage Foundation (build first)

SQLite schema with 11 tables: nodes, constraints, resolution_attempts, node_dependencies, logs, artifacts, capability_requests, resolvers, kg_nodes, kg_edges, budget_ledger. Plus a content-addressed artifact store.

### Layer 1: Core Units (all parallelizable — no dependencies between them)

| Unit | What it does |
|------|-------------|
| Node State Machine | Lifecycle management: pending → identifying → selecting → resolving → verifying → ... → terminated |
| Constraint Engine | CRUD, inheritance propagation (child inherits parent's resolved constraints), signature computation |
| Budget Enforcer | Per-node cost + time enforcement. Ledger ops (allocate/spend/refund). Parent→child allocation with 20% reserve |
| Dependency Resolver | DAG edges between sibling subgoals, topological sort for execution order |
| Knowledge Graph | Typed nodes + edges for learning. Four boot queries: resolver history, capability lookup, failure patterns, decomposition history |
| Logging | Three tiers: human-readable flow, structured decision JSON, I/O metadata |

### Layer 2: Composition (depends on Layer 1)

| Unit | What it does |
|------|-------------|
| Resolver Registry + Adapters | Tool library, capability matching, quality vectors. Claude Code adapter (6 prompt types), Codex Tools adapter |
| Verification Engine | Per-constraint evaluation (Z3 for logic, LLM for semantic), sigma composition, path confidence, gating |

### Layer 3: Integration (depends on everything above)

| Unit | What it does |
|------|-------------|
| Dispatch Loop | Main loop: pop node → check dependencies → route to action → budget-wrap every resolver call |
| Human Interface | Goal input (Step 1), HIRE escalation, capability request deduplication |

### Critical Path

```
Storage → [State Machine, Constraints, Budget, Dependencies, Logging]
       → [Registry + Adapters, Verification]
       → Dispatch Loop
       → Run validation examples
```

Layer 1 units can all be built in parallel.

## v1 Scope

What's in v1 and what's deferred:

| Capability | v1 Status |
|-----------|-----------|
| Storage (all 11 tables) | Full |
| Node State Machine | Full (minus gated/infeasible states) |
| Constraint Engine | Full |
| Budget Enforcer | Partial (wall-time hard cap, basic feasibility) |
| Dependency Resolver | Full |
| Knowledge Graph | Deferred (no value until reasoning loop validated) |
| Logging | Full (all 3 tiers) |
| Registry + Adapters | Partial (hardcoded 2 resolvers, 6 prompt types) |
| Verification | Partial (Z3 for logic, LLM for semantic) |
| Dispatch Loop | Partial (steps 1-9 only, no BUILD/HIRE) |
| Human Interface | Minimal (goal input only) |

## Validation Goals

Three test scenarios for v1:

1. **Autonomous SWE Agent**: Given a codebase + bug description, produce a fix that passes tests
2. **Quantitative Trading Strategy**: Given market data constraints, produce a backtested strategy
3. **Document Classification Deployment**: Given document types + accuracy constraints, produce a classifier pipeline

## Background You Need

If these concepts are unfamiliar, brief primers:

- **Constraint satisfaction**: problems defined by variables, domains, and constraints. A solution assigns values to all variables satisfying all constraints.
- **Tree data structures**: nodes with parent/child relationships. Root has no parent. Leaves have no children. Depth = distance from root.
- **State machines**: entities that transition between defined states via defined transitions. Invalid transitions are rejected.
- **DAGs**: directed acyclic graphs. Topological sort gives a valid execution order respecting all dependencies.

## Deep Dive

For full formal details, read these design documents:

- [Hierarchical Constraint-Driven Planning](/graftonlab/hierarchical-constraint-planning) — the mathematical formalization
- [Minimum Recursive Kernel](/graftonlab/minimum-recursive-kernel) — kernel axioms, interfaces, growth engine
- [Kernel Boot Specification](/graftonlab/kernel-boot-specification) — complete operational specification
- [Kernel Build: Task Breakdown](/graftonlab/kernel-build-task-breakdown) — implementation units and build order
