---
title: "Hierarchical Decomposition Outline"
description: "Document outline for hierarchical decomposition from atomic parts to complex multi-project systems"
---

## Context

This outlines a team-facing design document showing how the Grafton platform scales from atomic parts (mounting bracket) to complex multi-project systems (semiconductor foundry, hypersonic jet, microscope, tractor). Focus: decomposition process, hierarchical contracts, manufacturing planning, cross-project optimization.

**Not code implementation.** Workflows, schema interfaces, database concepts. No Python in main body.

## Inputs

* **Grafton Project Jan 2025 doc**: high-level project requirements, 27 items to build, dense verifiable hypergraph spec, contract elaboration, deconstruction strategies, microscope example
* **Previous doc** (joint-step-level-design-verification): atomic-level design loop, three-level constraint checking, manufacturing context, generalizable framework
* **Codebase exploration**: current hypergraph model (13 node types, 14 edge types), single-level decomposition, no subsystem nodes, flat assembly model

## Key Decisions

* **Running example**: 2nm semiconductor foundry
* **Depth**: breadth at top 2-3 levels + one deep vertical slice to atomic part
* **Secondary examples**: plane, microscope, tractor (for cross-project commonality)
* **Self-contained**: enough atomic-level context to stand alone, link to previous doc for deep dive
* **Cross-project manufacturing**: full section with workflow diagrams, shared DB schema, scheduling concepts
* **Audience**: development team, no deep prior knowledge of mech verification assumed
* **3-level evaluation**: unify atomic-level (parameter/VLM/B-Rep) and system-level (screening/bounds/simulation) as one universal pattern
* **Graph vocabulary**: introduce incrementally as the foundry example demands nodes (start minimal, add budget/experiment/coupling when needed)
* **Scheduling**: semi-formal (simplified constraint formulation, mention specific algorithms, keep readable)

---

## Document Outline

### 1. Introduction

* **1.1 Purpose**: what this doc covers, who it's for
* **1.2 The Problem**: how to go from "build a 2nm semiconductor foundry" to actionable, verifiable, manufacturable subproblems, autonomously
* **1.3 Core Thesis**: the same recursive pattern (intent → decompose → GRS → contracts → artifacts → verify) applies at every level, from system down to individual part. Manufacturing, uncertainty, and verification are first-class at all levels, not downstream afterthoughts.
* **1.4 Relationship to Existing Work**: brief pointer to atomic-level doc and Grafton Project requirements

### 2. The Decomposition Pattern

Visual-first section. Show the recursive structure before details.

* **2.1 Overview Diagram**: mermaid showing recursive intent → decompose → verify loop at multiple levels
* **2.2 What Happens at Each Level**: intent → GRS tree → contracts → artifacts → verification, but at system level the "artifact" is a subsystem decomposition, not a STEP file
* **2.3 When Decomposition Stops**: hitting atomic components (single-part, single-manufacturing-process units, or purchasable off-the-shelf items)
* **2.4 Bidirectional Propagation**: changes at leaf level propagate up (a manufacturing constraint on a mirror mount affects the optical subsystem budget, which affects the lithography system spec)

### 3. Running Example: 2nm Semiconductor Foundry

#### 3.1 Top-Level Intent and Requirements

* Intent: "Build a semiconductor fabrication facility capable of producing 2nm-node chips at target yield, cost, and throughput"
* Top-level GRS sketch: goals (yield > X%, throughput Y wafers/month, cost < $Z), requirements (cleanroom class, vibration budget, thermal stability), specs (quantitative thresholds)

#### 3.2 Level 1: Major Subsystems (Breadth)

Table/diagram of ~8-10 top-level subsystems:

* Lithography system
* Deposition systems (CVD, PVD, ALD, ECD)
* Etch systems (RIE, wet etch)
* Ion implantation
* CMP (chemical mechanical planarization)
* Metrology and inspection
* Wafer handling and transport
* Cleanroom and facilities (HVAC, vibration isolation, ultrapure water, gas delivery)
* Process control and automation

For each: one-line description, key contracts (what it assumes, what it guarantees), key uncertainties.

#### 3.3 Level 2: Subsystem Decomposition (Lithography Focus)

Expand lithography into sub-subsystems:

* EUV light source (plasma generation, collector optics, spectral filtering)
* Illumination optics (condenser, pupil shaping)
* Reticle/mask stage (precision positioning, particle protection)
* Projection optics (reduction optics, wavefront control)
* Wafer stage (nm-precision positioning, vibration isolation)
* Alignment and overlay measurement
* Dose control

Contracts at this level: e.g., EUV source guarantees X watts at 13.5nm; projection optics assumes < Y nm wavefront error from illumination.

#### 3.4 Deep Vertical Slice: EUV Source → Collector Mirror → Mirror Mount

Trace one path all the way to atomic:

1. **EUV Source** (subsystem): generates 13.5nm EUV radiation
2. **Collector Optics** (sub-subsystem): collects and focuses EUV from plasma
3. **Collector Mirror** (component): multilayer Mo/Si coated reflector
4. **Mirror Substrate** (part): precision-machined Zerodur blank
5. **Mirror Mount** (atomic part): kinematic mount with thermal compensation

At level 5, we're at the bracket-equivalent: a single part designed and manufactured using the atomic loop from the previous doc. Show how the atomic-level constraints (DFM, manufacturing) connect back up to the system-level contract (EUV source must deliver X watts).

#### 3.5 Contracts at Each Level

Show a concrete example of how contracts cascade:

* System: "Lithography resolution <= 2nm"
* Subsystem: "EUV source power >= 250W at 13.5nm"
* Component: "Collector reflectivity >= 65% across aperture"
* Part: "Mirror surface roughness <= 0.1nm RMS"
* Atomic: "Mount thermal drift <= 0.5nm/K over operating range"

Each guarantee at level N becomes an assumption at level N-1.

### 4. The Universal Three-Level Evaluation Pattern

Unify the atomic-level and system-level evaluation into one framework.

* **4.1 One Pattern, All Scales**: the same 3-level escalation applies everywhere
  * Level 0 / Parameter: instant checks from values
  * Level 1 / Visual+Bounds: fast triage with conservative estimates
  * Level 2 / Analysis: expensive, targeted
* **4.2 Escalation Logic**: same at all levels. Level 0 everywhere. Level 1 broadly. Level 2 only where margins are tight, coupling is complex, or novelty is present.
* **4.3 The Three Actions**: for every guarantee at every level: prove, bound, or test. If none is possible, redesign.

### 5. The Atomic Design Loop (Self-Contained Summary)

Condensed version of the previous doc for readers who haven't seen it.

* **5.1 Joint Step-Level Resolution**: generate one design step → verify constraints → assess manufacturing → proceed or fix
* **5.2 Constraint Checking at Atomic Level**: parameter (instant) → visual/VLM (fast) → analysis/B-Rep (slow) — an instance of the universal pattern
* **5.3 Manufacturing as First-Class Constraint**: cumulative tools, cost, time tracked at each step
* **5.4 How Atomic Outputs Feed Parent Contracts**: the completed part artifact, its verified properties, and manufacturing context become evidence nodes that validate (or violate) the parent contract

### 6. Combining Atomic Elements: Assembly and Integration

* **6.1 Emergent Constraints at Assembly**: individual parts pass all checks, but together create new failure modes
* **6.2 Interface Verification**: contracts between sibling parts
* **6.3 Manufacturing Context Aggregation**: individual part costs + assembly costs + integration testing costs roll up
* **6.4 Integration Testing Patterns**: what verification looks like at assembly level vs part level
* **6.5 Example**: collector mirror + mount + alignment mechanism → collector assembly

### 7. Manufacturing Planning Across the Hierarchy

* **7.1 Manufacturing as a Graph Problem**: tools, materials, processes, facilities form their own subgraph
* **7.2 Make vs Buy Decisions**: at each decomposition level
* **7.3 Tool and Facility Requirements Propagation**
* **7.4 Building Tools to Build Tools**: recursive manufacturing
* **7.5 The Manufacturing Process Database**: schema for linking components to processes to equipment
* **7.6 Workflow Diagram**: manufacturing planning from system-level down to facility layout

### 8. Cross-Project Optimization

* **8.1 The Multi-Project Scenario**: foundry, jet, microscope, tractor built concurrently
* **8.2 Common Manufacturing Motifs**: shared processes/tools across projects
* **8.3 Shared Component Library**: atomic components that appear across projects
* **8.4 The Global Manufacturing Graph**: linking multiple project hypergraphs
* **8.5 Scheduling and Resource Allocation** (semi-formal)
* **8.6 When to Build Custom Manufacturing Tools**
* **8.7 Workflow Diagram**: cross-project optimization loop

### 9. Handling Novelty vs Reuse

* **9.1 The Principle**: minimize novelty
* **9.2 Starting from Existing Products**: decomposition begins by surveying what already exists
* **9.3 The Novelty Frontier**: where existing products end and new design begins
* **9.4 Novelty as Uncertainty**: novel designs carry higher uncertainty

### 10. Data Architecture

* **10.1 Extended Node Types**
* **10.2 Extended Edge Types**
* **10.3 Hierarchical Contract Schema**
* **10.4 Manufacturing Graph Schema**
* **10.5 Cross-Project Linking**
* **10.6 The Three Databases**: Project DB, Manufacturing DB, Knowledge DB

### 11. Open Questions

* Decomposition strategy selection
* Backtracking across levels
* Autonomy vs handcrafting
* Scheduling complexity
* Uncertainty calibration
* Cross-project security

### Appendices

* **A**: Foundry Level 1-2 subsystem decomposition table (full detail)
* **B**: Cross-project common motifs matrix (processes x projects)
* **C**: Schema definitions (node types, edge types, contract hierarchy, manufacturing graph)
* **D**: Contract cascade example (foundry → lithography → EUV → collector → mount, full contracts at each level)
