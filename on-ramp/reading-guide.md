---
title: "Reading Guide"
description: "Which design documents to read based on your role and current work, plus the document dependency graph"
---

## By Phase

Each phase has essential reading (required for contribution) and optional deep dives (for full context).

### Phase 1: Building the Kernel

| Priority | Document | What you'll learn |
|----------|----------|------------------|
| Essential | [Minimum Recursive Kernel](/graftonlab/minimum-recursive-kernel) | Kernel axioms, base interfaces, growth engine (TRY/BUILD/HIRE) |
| Essential | [Kernel Boot Specification](/graftonlab/kernel-boot-specification) | Full operational spec — every step, data model, state machine |
| Essential | [Kernel Build: Task Breakdown](/graftonlab/kernel-build-task-breakdown) | Implementation units, dependency layers, build checklist |
| Deep dive | [Hierarchical Constraint-Driven Planning](/graftonlab/hierarchical-constraint-planning) | The formal math behind every algorithm |

### Phase 2: Verification & Reasoning

| Priority | Document | What you'll learn |
|----------|----------|------------------|
| Essential | [Symbolic Reasoning for Verification](/graftonlab/symbolic-reasoning-verification) | Four-layer verification (Datalog, Z3, LLM), domain ontology examples |
| Essential | [Hypergraph Uncertainty](/graftonlab/hypergraph-uncertainty) | Clause-level uncertainty, evidence fusion, VOI-driven decisions |
| Deep dive | [Domain-Agnostic Goal Decomposition](/graftonlab/domain-agnostic-goal-decomposition) | Autonomous capability building, Datalog ontology construction |
| Deep dive | [Constraint-Driven Problem Solving](/graftonlab/constraint-driven-problem-solving) | Philosophical foundation — CSPs, HTNs, constraint relaxation |

### Phase 3: Physical Design & Manufacturing

| Priority | Document | What you'll learn |
|----------|----------|------------------|
| Essential | [Joint Step-Level Verification](/graftonlab/joint-step-level-verification) | Atomic design loop, three-level checking, manufacturing context |
| Essential | [Hierarchical Decomposition Design](/graftonlab/hierarchical-decomposition-design) | System-to-part decomposition, contracts cascade, manufacturing planning |
| Deep dive | [Hierarchical Decomposition Outline](/graftonlab/hierarchical-decomposition-outline) | Document structure and scope decisions for the decomposition design |

### Phase 4: Multi-Agent Architecture & Scaling

| Priority | Document | What you'll learn |
|----------|----------|------------------|
| Essential | [Multi-Agent Architecture](/graftonlab/multi-agent-architecture) | DAG orchestration, stable interfaces, wave-parallel execution |
| Deep dive | [Constraint-Driven Problem Solving](/graftonlab/constraint-driven-problem-solving) | Problem formulation, algorithm selection, solver strategies |

## By Role

### Software Engineer (building the platform)

**Start with**: [On-Ramp Phase 1](/on-ramp/phase-1-kernel) → [Kernel Build: Task Breakdown](/graftonlab/kernel-build-task-breakdown) → [Kernel Boot Specification](/graftonlab/kernel-boot-specification)

**Then**: Pick your assigned Layer 1 unit and read the corresponding section in the Task Breakdown. The specification has the detailed data model and state transitions you'll implement.

**For verification work**: [Symbolic Reasoning for Verification](/graftonlab/symbolic-reasoning-verification) for the Datalog/Z3 integration patterns.

### Systems Engineer (decomposition, contracts, architecture)

**Start with**: [On-Ramp Phase 3](/on-ramp/phase-3-physical-design) → [Hierarchical Decomposition Design](/graftonlab/hierarchical-decomposition-design) → [Joint Step-Level Verification](/graftonlab/joint-step-level-verification)

**Then**: [Multi-Agent Architecture](/graftonlab/multi-agent-architecture) for how decomposition agents work at scale.

**For formal grounding**: [Hierarchical Constraint-Driven Planning](/graftonlab/hierarchical-constraint-planning) — the contract cascade and confidence propagation math.

### Researcher (reasoning, verification, uncertainty)

**Start with**: [On-Ramp Phase 2](/on-ramp/phase-2-verification) → [Hierarchical Constraint-Driven Planning](/graftonlab/hierarchical-constraint-planning) → [Symbolic Reasoning for Verification](/graftonlab/symbolic-reasoning-verification)

**Then**: [Hypergraph Uncertainty](/graftonlab/hypergraph-uncertainty) for the evidence fusion model and [Domain-Agnostic Goal Decomposition](/graftonlab/domain-agnostic-goal-decomposition) for autonomous capability building.

### Product / Program Manager

**Start with**: [On-Ramp Project Overview](/on-ramp/project-overview) → [On-Ramp Core Concepts](/on-ramp/core-concepts)

**Then**: Skim each phase doc's introduction and "Background You Need" sections. Read [Kernel Build: Task Breakdown](/graftonlab/kernel-build-task-breakdown) for the build checklist and v1 scope.

## Document Dependency Graph

Read from top to bottom. Documents at the same level have no dependencies on each other.

```
                    Hierarchical Constraint-Driven Planning
                         (formal foundation for everything)
                                      |
                    +-----------------+-----------------+
                    |                 |                 |
            Minimum Recursive    Constraint-Driven   Domain-Agnostic
               Kernel            Problem Solving     Goal Decomposition
                    |                                      |
          +---------+---------+                    Symbolic Reasoning
          |                   |                    for Verification
   Kernel Boot         Kernel Build                      |
   Specification      Task Breakdown              Hypergraph Uncertainty
          |
          +---------------------------+
          |                           |
   Joint Step-Level            Multi-Agent
   Verification                Architecture
          |
   Hierarchical Decomposition
   Design
          |
   Hierarchical Decomposition
   Outline
```

## Key Open Questions

These are unresolved design questions collected from across the design documents. Each links to the document where it's discussed. These are areas where your input can shape the system's direction.

### Kernel & Growth

- **Baseline knowledge store**: How are failures and successes brought to context for the resolver and builder? ([Minimum Recursive Kernel](/graftonlab/minimum-recursive-kernel))
- **Kernel axioms minimality**: What is truly the irreducible set? ([Minimum Recursive Kernel](/graftonlab/minimum-recursive-kernel))
- **Token budget / run cost enforcement**: How to enforce overall cost bounds across recursive subgoal creation? ([Minimum Recursive Kernel](/graftonlab/minimum-recursive-kernel))
- **Tiered logging**: What's the right granularity for the reference store? ([Minimum Recursive Kernel](/graftonlab/minimum-recursive-kernel))

### Verification & Reasoning

- **Automated constraint elicitation**: How to surface implicit constraints without human domain experts? ([Constraint-Driven Problem Solving](/graftonlab/constraint-driven-problem-solving))
- **VLM calibration**: How to tune the VLM's conservatism (false positive vs false negative trade-off)? ([Joint Step-Level Verification](/graftonlab/joint-step-level-verification))
- **Evidence conflict resolution**: When two evidence sources disagree, how to resolve without strong credibility priors? ([Hypergraph Uncertainty](/graftonlab/hypergraph-uncertainty))

### Physical Design

- **Backtracking strategy**: If Step 5 reveals Step 2 was wrong, how far do we unwind? ([Joint Step-Level Verification](/graftonlab/joint-step-level-verification))
- **Dynamic constraint discovery**: Some constraints only emerge at later steps — how to handle retroactive constraints? ([Joint Step-Level Verification](/graftonlab/joint-step-level-verification))
- **Cost model fidelity**: What's the minimum viable manufacturing cost model? ([Joint Step-Level Verification](/graftonlab/joint-step-level-verification))
- **Backtracking across hierarchy levels**: When a leaf constraint invalidates a parent contract, how to propagate efficiently? ([Hierarchical Decomposition Design](/graftonlab/hierarchical-decomposition-design))

### Architecture & Scaling

- **Decomposition strategy selection**: How to choose between functional, physical, and process decomposition? ([Hierarchical Decomposition Outline](/graftonlab/hierarchical-decomposition-outline))
- **Scheduling complexity**: How formal should resource scheduling be? ([Hierarchical Decomposition Outline](/graftonlab/hierarchical-decomposition-outline))
- **Cross-project security**: How to share capabilities without leaking proprietary design data between projects? ([Hierarchical Decomposition Outline](/graftonlab/hierarchical-decomposition-outline))
