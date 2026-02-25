---
title: "Phase 3: Physical Design & Manufacturing"
description: "How Grafton handles physical engineering — from individual parts to complex systems, with manufacturing as a first-class constraint"
---

## The Shift from Software to Physical

In Phases 1 and 2, the system handles software goals: code artifacts verified by tests, logic solvers, and LLM judgment. Phase 3 extends the same framework to physical engineering — designing parts, verifying geometry, tracking manufacturing feasibility, and scaling to multi-level system hierarchies.

The framework doesn't change. The resolvers, constraints, and verification methods do:

| Aspect | Software (Phases 1-2) | Physical (Phase 3) |
|--------|----------------------|-------------------|
| Artifact | Code, config, docs | CAD geometry, STEP files, manufacturing plans |
| Resolvers | LLM, code executor, test runner | LLM, CAD kernel, VLM, FEA, simulation |
| Constraints | Test pass, performance bounds, style | DFM rules, material properties, thermal limits, cost, tooling |
| Verification | Z3, test suite, LLM judgment | Parameter checks, visual inspection (VLM), B-Rep analysis |

## The Atomic Design Loop

Every physical part — a bracket, a mirror mount, a heat sink — can be decomposed into an ordered sequence of **atomic design steps**. Each step is a single manufacturing operation: cut a plate, drill holes, add ribs, attach a flange.

The core idea: **execute each step individually, and before proceeding to the next, jointly verify design constraints, visual integrity, and manufacturing feasibility.**

If any check fails, the LLM regenerates that single step with specific failure context. No full-design rework.

### Worked Example: Mounting Bracket

**Goal**: "Design a mounting bracket for a 5kg load, CNC machined from aluminum 6061."

Decomposed into 4 steps:

---

**Step 1: Create base plate** (100mm x 50mm x 3mm)

| Check | Result |
|-------|--------|
| Parameter: thickness 3mm >= 1mm min | Pass |
| Parameter: aspect ratio 33:1 &lt;= 50:1 | Pass |
| Manufacturing: cut from 3mm Al 6061 sheet stock | Laser cutter, ~$2, 3 min |
| VLM: "rectangular plate, correct proportions" | Pass |

Cumulative manufacturing state: 1 setup (laser), $2, 3 min. Proceed.

---

**Step 2: Drill 4 mounting holes** (M6 through-hole, 15mm edge inset)

| Check | Result |
|-------|--------|
| Parameter: diameter 6mm >= 1mm | Pass |
| Parameter: L/D = 3/6 = 0.5 &lt;= 10 | Pass |
| Parameter: edge distance 15mm >= 12mm (2x diameter) | Pass |
| Manufacturing: 6mm drill bit, drill press | +$1.50, 2 min |
| VLM: "4 holes symmetrically placed, good edge margins" | Pass |

Cumulative: 2 setups (laser + drill), $3.50, 5 min. Proceed.

---

**Step 3: Add stiffening ribs** (height=20mm, thickness=2mm)

| Check | Result |
|-------|--------|
| Parameter: rib aspect ratio 20/2 = 10:1 &lt;= 8:1 | **FAIL** |

Immediate feedback to LLM: "Rib aspect ratio 10:1 exceeds maximum 8:1. Either reduce height to 16mm or increase thickness to 2.5mm."

LLM regenerates with height=16mm. Now requires CNC machining (+$15). Cumulative: 3 setups, $18.50, 15 min.

---

**Step 4: Add vertical plate** (50mm x 30mm x 3mm, welded perpendicular)

| Check | Result |
|-------|--------|
| VLM: "vertical plate attached, but fillet at base looks small" | **Escalate** |
| Targeted B-Rep analysis: fillet radius at base joint = 0.15mm | **FAIL** (min 0.2mm) |

The VLM caught an emergent geometry issue that parameter checks couldn't see. Only that specific region gets full analysis — not the entire part.

Final: 4 setups, $26.50, 20 min.

## Three-Level Constraint Checking

Not all constraints require the same effort to verify. The system uses a graduated approach that catches most violations cheaply:

### Level 1: Parameter Checks (instant)

Check values directly from the code/design specification. No geometry processing needed. Covers the majority of DFM rules:

- Hole diameter, length-to-diameter ratio
- Wall thickness minimums
- Fillet radius minimums
- Slot width, depth ratios
- Edge distances
- Aspect ratios

**~80% of violations are caught here.**

### Level 2: Visual Checks — VLM (fast)

A Vision-Language Model inspects rendered views of the geometry. Catches spatial relationships and emergent issues that parameter checks miss:

| Emergent Issue | How VLM Detects It | If Flagged, Route To |
|---------------|-------------------|---------------------|
| Thin walls at boolean intersections | Cross-section renders | Ray-cast at flagged location |
| Features too close to edges | Top/side orthographic views | Edge distance computation |
| Disconnected/floating geometry | Isometric view | Topology validation |
| Draft angle issues | Silhouette from pull direction | Face normal analysis |
| Internal voids | Cross-section slices | Solid validation |
| Sharp internal corners | Highlighted edge render | Stress concentration check |

The VLM acts as a **triage layer** — it doesn't need to precisely measure; it flags regions that look problematic, and those specific regions get routed to expensive analysis.

### Level 3: Full Analysis — B-Rep (slow, targeted)

Geometry operations on the actual solid model. Only invoked for specific regions flagged by the VLM:

- Ray-casting through the solid for wall thickness measurement
- Assembly interference detection (loading and comparing separate STEP solids)
- Tolerance stack-up analysis across multiple features
- PMI/GD&T compliance (parsing STEP annotations)

**~20% of real verification needs require this level.** By targeting only VLM-flagged regions, the system avoids running expensive analysis on the entire part.

### Why This Order Matters

The graduated approach saves significant computation:

```
All constraints
    |
    v
Parameter checks (instant) → catches ~80%
    |
    v
VLM inspection (fast) → catches ~15% of remainder, flags regions
    |
    v
B-Rep analysis (slow, targeted) → resolves flagged regions only (~5% of total)
```

Running full B-Rep analysis on every constraint at every step would be orders of magnitude more expensive.

## Manufacturing as a First-Class Constraint

Traditional design pipelines assess manufacturability after the design is complete. By then, changing anything requires cycling back through multiple stages. Grafton tracks manufacturing at every step.

### Cumulative Manufacturing Context

At each design step, the system tracks:

| Field | What it tracks |
|-------|---------------|
| Tools committed | Which manufacturing equipment is needed (laser, drill, CNC, welder) |
| Setup count | How many physical setups / fixturings |
| Cumulative cost | Running total of manufacturing cost |
| Cumulative time | Running total of manufacturing time |
| Material state | What material is in use, stock dimensions, remaining material |

### Manufacturing Constraint Types

Manufacturing constraints function just like any other constraint, but they carry additional weight because they affect cost and feasibility:

- **Blocking**: "This operation requires EDM, which is outside the tooling budget." The step can't proceed until the design is changed.
- **Informing**: "This operation adds $3 and one setup change." The system reports but doesn't block.
- **Trade-off enabling**: "Two approaches satisfy DFM constraints; option A costs $2 more but eliminates a setup change."

### Manufacturing Cost Jumps

Some design decisions trigger disproportionate cost increases. The system surfaces these at the step where they occur:

- Adding stiffening ribs → requires CNC machining (jumped from $3.50 to $18.50 in the bracket example)
- Tight tolerances → requires grinding or EDM instead of standard machining
- Multi-axis features → requires 5-axis CNC instead of 3-axis

By surfacing these at the step level, the designer can make informed trade-offs immediately rather than discovering them after the full design is committed.

## Scaling to Hierarchical Systems

The atomic design loop handles single parts. Real systems have thousands of parts organized in hierarchies. Phase 3 extends the framework to multi-level decomposition.

### The Recursive Pattern

The same pattern applies at every level of the hierarchy:

```
Intent → Decompose into subgoals → Solve each → Verify → Assemble
```

At the system level, "solve" means decomposing further. At the atomic level, "solve" means designing a part. The framework is the same; only the artifacts differ:

| Level | What "artifact" means |
|-------|----------------------|
| System | Subsystem decomposition + interface contracts |
| Subsystem | Component list + assembly plan |
| Component | Part designs + integration spec |
| Part | Manufacturing-ready geometry |
| Atomic step | Single design operation on a workpiece |

### Contracts Cascade

At each level, a subsystem makes guarantees given assumptions. The parent's guarantee becomes the child's assumption:

**Running example: 2nm Semiconductor Foundry**

| Level | What | Guarantee |
|-------|------|-----------|
| System | Foundry | Produce 2nm chips at target yield, cost, throughput |
| Subsystem | Lithography | Resolution &lt;= 2nm |
| Sub-subsystem | EUV Source | Power >= 250W at 13.5nm |
| Component | Collector Optics | Reflectivity >= 65% across aperture |
| Part | Collector Mirror | Surface roughness &lt;= 0.1nm RMS |
| Atomic part | Mirror Mount | Thermal drift &lt;= 0.5nm/K |

Each guarantee at level N becomes an assumption at level N-1. If the mirror mount can't achieve 0.5nm/K thermal drift, that propagates upward: collector mirror roughness may not hold, collector reflectivity drops, EUV source power is insufficient, lithography resolution exceeds 2nm. The system detects this cascade and backtracks to the point where the constraint can be relaxed or an alternative approach found.

### The Three-Level Evaluation — Universal

The parameter → visual → analysis pattern from atomic design applies at ALL scales:

| Level | Parameter (instant) | Visual/Bounds (fast) | Analysis (slow) |
|-------|-------------------|---------------------|----------------|
| Atomic part | DFM rule check | VLM geometry inspection | B-Rep ray-cast |
| Component | Interface dimension check | Assembly render review | Interference detection |
| Subsystem | Performance envelope check | Block diagram review | System simulation |
| System | Budget/schedule check | Architecture review | End-to-end simulation |

The principle is the same: cheap checks first, expensive checks targeted at flagged areas only.

## Cross-Project Optimization

When multiple projects run concurrently (e.g., a foundry, an aircraft, a microscope, and a tractor), they share manufacturing capabilities:

- **Shared tools**: A CNC machine used by one project can serve others
- **Shared components**: A standard mounting bracket design used across projects
- **Shared processes**: Surface treatment, heat treatment, testing rigs
- **Shared knowledge**: DFM rules learned on one project apply to others

The registry (from Phase 1) accumulates these shared capabilities. A resolver built for one project's thermal analysis applies to any project with thermal constraints. Manufacturing ontology rules built for aluminum machining apply wherever aluminum is machined.

## Background You Need

**DFM (Design for Manufacturing)**: Rules that ensure a design can actually be manufactured. Examples: minimum wall thickness, minimum hole diameter, maximum aspect ratios, minimum fillet radii, draft angles for molding.

**B-Rep (Boundary Representation)**: A way of representing 3D geometry as a collection of surfaces (faces) that bound a solid. Used for precise geometric analysis: wall thickness measurement, interference detection, surface area computation.

**CNC Machining**: Computer-controlled cutting. 3-axis (X, Y, Z motion) is cheaper; 5-axis (adds rotation) handles complex geometries but costs more. The number of axes needed is a manufacturing constraint.

**VLM (Vision-Language Model)**: An AI model that can analyze images and answer questions about them. Used here to inspect rendered geometry and flag potential issues.

**Systems Engineering Hierarchy**: Complex systems decompose into subsystems, which decompose into components, which decompose into parts. Each level has interfaces (what connects to what) and contracts (what each part guarantees).

## Deep Dive

- [Joint Step-Level Verification](/graftonlab/joint-step-level-verification) — the atomic design loop, three-level checking, manufacturing context
- [Hierarchical Decomposition Design](/graftonlab/hierarchical-decomposition-design) — full system decomposition with the foundry example
- [Hierarchical Decomposition Outline](/graftonlab/hierarchical-decomposition-outline) — document structure for the decomposition design
