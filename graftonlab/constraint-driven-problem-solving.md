---
title: "Constraint-Driven Problem Solving Architecture"
description: "Framework for identifying constraints, exploring solutions, and handling recursive problem decomposition"
---

A comprehensive framework for identifying constraints, exploring solutions, and handling recursive problem decomposition in complex domains.

---

## 1. Constraint Identification & Representation

### Formal Constraint Modeling

**Constraint Satisfaction Problems (CSPs)** are a well-established formalism where you define variables, domains, and constraints. Solvers like SAT/SMT solvers (Z3, MiniSat) can find satisfying assignments or prove infeasibility.

**Ontology-driven constraint elicitation** uses knowledge graphs to systematically discover constraints. Many constraints are implicit — they live in domain expertise, physics, regulations, and economics. A structured ontology forces them to be made explicit.

### Typed Constraint Taxonomy

Classify constraints into the following types, since the architecture needs to know which constraints can be relaxed and which cannot:

* **Hard** (Physical laws) — Cannot be relaxed → Feasibility Boundary
* **Soft** (Preferences) — Can be relaxed → Optimization Space
* **Derived** — Emerge from other constraints, depends on Hard and Soft
* **Conditional** — Active only in certain configurations, triggered by Configuration State

---

## 2. Solution Exploration

### For Well-Defined Search Spaces

| Approach | Best For | Tools |
|----------|----------|-------|
| **Bayesian Optimization** | Expensive evaluations; builds a surrogate model (typically Gaussian Process) and uses an acquisition function to balance exploration vs. exploitation | Custom implementations |
| **Constraint Programming (CP)** | Combinatorial constraints; propagates constraints to prune the search space before enumerating | Google OR-Tools, Gecode, IBM CP Optimizer |
| **Mixed-Integer Programming (MIP)** | Linearizable constraints and objectives | Gurobi, CPLEX |

### For Ill-Defined or Emergent Search Spaces

| Approach | Best For |
|----------|----------|
| **Evolutionary / Genetic Algorithms** | Irregular, multi-modal, or hard-to-formalize solution spaces; encode arbitrary constraints as fitness penalties |
| **Monte Carlo Tree Search (MCTS)** | Sequential decision problems with a tree of choices — relevant to cascading-challenges scenarios |

---

## 3. The Cascading / Recursive Decomposition Problem

This is the hardest and most interesting part. The architecture must handle:

> *"To solve A, I discovered I need to solve B, which requires solving C..."*

### Hierarchical Task Networks (HTNs)

Decompose high-level goals into sub-tasks, each with their own constraints. The key property: decomposition is recursive, and constraints flow **both downward** (parent imposes on child) **and upward** (child's infeasibility forces parent to replan).

### Goal-Directed Dependency Graphs (DAGs with Backpressure)

Model the cascade as a directed graph where each node is a sub-problem with its own constraint set. Edges represent "enables" relationships. Critically, you need **backpropagation of infeasibility** — if a sub-problem is unsolvable, that information must propagate back to reframe the parent problem.

### Constraint Relaxation & Reformulation

When a sub-problem is infeasible, the system should automatically attempt the following sequence:

1. **Constraint Relaxation** — Loosen soft constraints
2. **Problem Reformulation** — Redefine the sub-problem (if still infeasible)
3. **Lateral Search** — Find entirely different decomposition path (if still infeasible)
4. **Escalate to Parent** — Backpropagate infeasibility

> **TRIZ Connection:** The theory of inventive problem solving (TRIZ) emphasizes that contradictions between constraints are where innovation happens. This directly maps to the reformulation step.

---

## 4. The "Truly General / Unbiased" Requirement

This is the philosophical crux. Rather than committing to a single solver, the system should be adaptive.

### Meta-Learning / Algorithm Selection

Use a meta-layer that characterizes the sub-problem and selects the appropriate solver. This is studied in Algorithm Selection (Rice's framework) and AutoML.

### Program Synthesis

Instead of searching within a fixed solution space, synthesize the solution procedure itself. Systems like **DreamCoder** or neural program synthesis can learn to construct novel solution strategies.

### Active Inference / Free Energy Principle

A framework from neuroscience where an agent minimizes surprise by either updating its model (perception) or acting on the world (action). It naturally handles the "set up an experimental rig" move — when the model is too uncertain, the rational action is to **gather more information**.

---

## 5. Candidate Architecture

The full end-to-end architecture integrating all components:

**Phase 1: Problem Intake / Framing**
* Constraint Elicitation
* Ontology Mapping

**Phase 2: Constraint Graph Builder**
* Hard / Soft / Derived / Conditional Typing
* Feasibility Pre-check via SAT/SMT

**Phase 3: Hierarchical Decomposition Engine**
* HTN Planning
* Recursive Sub-problem Generation
* Backpropagation of Infeasibility

**Phase 4: Solving**
* Direct Solve — CP, MIP, BO, Evolutionary
* Meta-Problem Solve — Reformulation, Experimental Design, Program Synthesis

**Phase 5: Solution Validation & Integration**
* Verify All Constraints Satisfied
* Compose Sub-solutions into Whole

---

## 6. Key Insight: Fluid Boundary Between Solving and Meta-Solving

The deepest idea is that the boundary between "solving" and "meta-solving" should be fluid. The system needs to:

1. **Attempt** to solve the problem directly
2. **Recognize** when it's stuck (infeasibility detection)
3. **Reformulate** the problem — which itself is a constraint-satisfaction problem
4. **Recurse** arbitrarily deep

This is essentially what researchers call **reflective architectures** or **self-improving systems**.

### Related Frameworks

| Framework | Key Idea |
|-----------|----------|
| **SOAR** | Cognitive architecture with impasse-driven learning |
| **OpenNARS** | Non-Axiomatic Reasoning System — handles uncertain, open-world reasoning |
| **Category Theory** | Mathematically rigorous composability of solutions and constraint transformations across levels of abstraction |

---

## The Hardest Unsolved Piece

**Automated constraint elicitation** — reliably identifying all relevant constraints for a novel problem domain — remains the critical open challenge. Three approaches work in combination:

1. **Human expertise** — domain specialists surface implicit constraints
2. **Domain ontologies** — structured knowledge graphs make constraints explicit
3. **Adversarial constraint discovery** — actively stress-testing proposed solutions to find missing constraints
