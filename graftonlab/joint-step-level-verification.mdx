---
title: "Joint Step-Level Verification"
description: "Simplifying design verification through joint step-level resolution with manufacturing-aware design"
---

## 1. Introduction

### 1.1 Purpose

This document proposes a fundamental restructuring of how the Grafton platform handles design verification and optimization. Today, design generation (Steps 1-3) runs to completion before verification (Steps 4-6) begins, creating a long sequential pipeline with multiple iteration loops between stages. This proposal shifts verification and manufacturing assessment *into* the design generation loop, resolving constraints at each atomic design step rather than after the full design is committed.

### 1.2 Motivation: Design Alone Doesn't Deliver Value

A geometrically valid CAD model is necessary but not sufficient. The real value lies in producing a design that can be **manufactured** within cost, lead time, and tooling constraints.

Today, the pipeline treats manufacturing feasibility as a downstream concern. By the time a manufacturing issue surfaces ("this pocket requires 5-axis CNC but the budget only allows 3-axis"), the entire design has been committed and rework requires cycling through multiple pipeline stages.

### 1.3 What This Proposal Enables

* **Simplified pipeline**: eliminate redundant verification/optimization stages
* **Early manufacturing visibility**: surface cost, tooling, and lead time implications at each design step
* **Joint resolution**: treat manufacturing viability as a first-class constraint alongside geometry and DFM
* **Generalizable pattern**: the same step-level resolution architecture applies across mechanical, electrical, and other engineering domains

### 1.4 Limitations

Here we are focused on **one** atomic unit of design, such as a mounting bracket. How multiple atomic units come together and how requirements are propagated will be shared in a separate doc.

---

## 2. Workflow Comparison: Current vs Proposed

### 2.3 Key Differences

| Aspect | Current Pipeline | Proposed |
|--------|------------------|----------|
| When violations are detected | After full design complete | At the step that introduces them |
| How violations are fixed | NLopt per-parameter optimization + LLM refinement cycle | LLM rewrites the code directly with error context |
| Manufacturing awareness | None in verification loop | First-class constraint at every step |
| VLM role | Applied in cad generation loop | Triage layer for visual checks + escalation decisions |
| Format boundary crossings | 5+ per iteration (JSON, STEP, MDS, suggestions, etc.) | 0: code is the artifact, checks run on live workpiece |
| Heavy B-Rep analysis | Runs on full geometry every iteration | Targeted, only for VLM-flagged regions |
| Optimization approach | NLopt solving trivial 1D bound problems | LLM with full design + manufacturing context |

---

## 3. The Proposal: Joint Step-Level Resolution

### 3.1 Core Idea

Every engineered design can be decomposed into an ordered sequence of atomic operations. The proposal: execute each operation individually, and before proceeding to the next, jointly verify:

1. **Design constraints**: does this operation satisfy DFM rules and structural requirements?
2. **Visual integrity**: does the geometry look correct? Are there emergent issues?
3. **Manufacturing feasibility**: can this operation be performed within cost, tooling, and time constraints?

If any check fails, the LLM regenerates that single operation with the specific failure context.

### 3.2 Worked Example: Mounting Bracket

Intent: *"Design a mounting bracket for a 5kg load, CNC machined from aluminum 6061."*

Decomposed into 4 steps:

---

**Step 1: Create base plate** (100mm x 50mm x 3mm)

| Check | Result |
|-------|--------|
| Parameter: thickness 3mm >= 1mm min | Pass |
| Parameter: aspect ratio 33:1 <= 50:1 | Pass |
| Manufacturing: cut from 3mm Al 6061 sheet stock | Laser cutter, ~$2, 3 min |
| VLM: "rectangular plate, correct proportions" | Pass |

Cumulative manufacturing state: 1 setup (laser), $2, 3 min. Proceed to Step 2.

---

**Step 2: Drill 4 mounting holes** (M6 through-hole, 15mm edge inset)

| Check | Result |
|-------|--------|
| Parameter: diameter 6mm >= 1mm | Pass |
| Parameter: L/D = 3/6 = 0.5 <= 10 | Pass |
| Parameter: edge distance 15mm >= 12mm (2x diameter) | Pass |
| Manufacturing: 6mm drill bit, drill press | +$1.50, 2 min |
| VLM: "4 holes symmetrically placed, good edge margins" | Pass |

Cumulative: 2 setups (laser + drill), $3.50, 5 min. Proceed to Step 3.

---

**Step 3: Add stiffening ribs** (height=20mm, thickness=2mm)

| Check | Result |
|-------|--------|
| Parameter: rib aspect ratio 20/2 = 10:1 <= 8:1 | **FAIL** |

Immediate feedback to LLM:

> "Rib aspect ratio 10:1 exceeds maximum 8:1. Either reduce height to 16mm or increase thickness to 2.5mm."

LLM regenerates with height=16mm. Manufacturing cost jump (+$15 for CNC) surfaced immediately. Cumulative: 3 setups, $18.50, 15 min.

---

**Step 4: Add vertical plate** (50mm x 30mm x 3mm, welded perpendicular)

| Check | Result |
|-------|--------|
| VLM: "vertical plate attached, but fillet at base looks small" | **Escalate** |
| Targeted B-Rep analysis: fillet radius at base joint = 0.15mm | **FAIL** (min 0.2mm) |

The VLM caught an emergent geometry issue that parameter checks couldn't see. Only that specific region gets analyzed.

Final cumulative: 4 setups, $26.50, 20 min.

### 3.3 Three Levels of Constraint Checking

Not all constraints require the same effort to verify:

**Parameter level** (instant): Checks values directly from the code. Covers the majority of DFM rules: hole diameter, L/D ratio, wall thickness, fillet radius, slot width.

**Visual level** (fast): VLM inspects rendered views. Catches spatial relationships, symmetry, obvious topology issues, and flags regions that might have emergent geometry problems.

**Analysis level** (slow): Full B-Rep analysis using OCCT or equivalent. Only invoked for specific regions flagged by the VLM, not the entire part.

### 3.4 The VLM as Triage Layer

| Emergent Property | VLM Detection | Reliability | If Flagged, Route To |
|-------------------|---------------|-------------|----------------------|
| Thin walls at boolean intersections | Cross-section renders | High | Ray-cast at flagged location |
| Features too close to edges | Top/side orthographic | High | Edge distance computation |
| Disconnected/floating geometry | Isometric view | High | Topology validation |
| Draft angle issues | Silhouette from pull direction | Medium | Face normal analysis |
| Internal voids | Cross-section slices | Medium | Solid validation |
| Sharp internal corners | Highlighted edge render | Medium | Stress concentration check |
| Sub-mm tolerance issues | N/A | Low | Always route to analysis |

### 3.5 When Heavy Analysis Is Still Needed

Some properties genuinely cannot be checked from code parameters or visual inspection:

* **Geometry-emergent wall thickness**: requires ray-casting through the actual B-Rep solid
* **Assembly interference**: requires loading and comparing separate STEP solids
* **Tolerance stack-up**: cumulative effects across multiple features
* **PMI/GD&T compliance**: requires parsing STEP annotations

These represent roughly 20% of real verification needs.

---

## 4. Manufacturing-Aware Design

### 4.1 Why Manufacturing Belongs at Each Step

The traditional approach of designing first and assessing manufacturability later creates a fundamental problem: by the time a manufacturing constraint is discovered, the full design is committed. By assessing manufacturing at each step, the designer can make an informed decision locally.

### 4.2 Cumulative Manufacturing Context

| After Step | Tools Committed | Setups | Cumulative Cost | Cumulative Time |
|------------|-----------------|--------|-----------------|-----------------|
| 1: Base plate | Laser cutter | 1 | $2.00 | 3 min |
| 2: Mounting holes | + Drill press | 2 | $3.50 | 5 min |
| 3: Stiffening ribs | + 3-axis CNC | 3 | $18.50 | 15 min |
| 4: Vertical plate | + Welding | 4 | $26.50 | 20 min |

### 4.3 Manufacturing as a Constraint, Not a Separate Pipeline

Manufacturing is a constraint with the same status as DFM or structural requirements:

* **Blocking**: "this operation requires EDM which is outside the tooling budget"
* **Informing**: "this operation adds $3 and one setup change"
* **Trade-off enabling**: "two approaches satisfy DFM constraints; option A costs $2 more but saves a setup change"

---

## 5. Generalizable Framework

### 5.1 The Pattern

> **Any design = an ordered sequence of atomic steps, where each step must satisfy design constraints, visual integrity, and manufacturing feasibility before the next step begins.**

### 5.2 Core Abstractions

* **DesignStep**: an atomic unit of design work
* **StepArtifact**: the output of a single step
* **Constraint**: a condition that must hold, with a check level
* **DesignContext**: accumulated state across all completed steps
* **ManufacturingContext**: cumulative manufacturing state
* **StepOrchestrator**: runs the generate-verify-assess loop for a single step
* **DesignPipeline**: sequences steps with context propagation

### 5.3 Cross-Domain Application

| Concept | Mechanical | Electrical |
|---------|------------|------------|
| **DesignStep** | Drill hole, add fillet, cut pocket | Place component, route trace, add via |
| **StepArtifact** | CadQuery Workpiece (live 3D solid) | KiCad Board state (layout + netlist) |
| **Parameter constraints** | Hole diameter >= 1mm, wall >= 1mm | Trace width for current, clearance between nets |
| **Visual constraints (VLM)** | Thin walls, edge proximity, symmetry | Component overlap, thermal zones |
| **Analysis constraints** | B-Rep ray-cast, FEA | DRC engine, signal integrity simulation |
| **Manufacturing context** | CNC setup, drill bits, laser time | PCB layer count, min trace/space |

### 5.4 Constraint Escalation Model

The three-level escalation (parameter → visual → analysis) is universal because:

1. **Most violations are parameter-level** (cheap)
2. **Emergent issues are usually visually apparent** (moderate)
3. **Precise quantification is rarely needed upfront** (expensive)

---

## 6. Open Questions

* **Backtracking strategy**: If Step 5 reveals Step 2 was wrong, how far do we unwind?
* **Dynamic constraint discovery**: Some constraints only emerge at later steps
* **Manufacturing alternatives**: Should the system propose alternative operations?
* **VLM calibration**: How do we tune the VLM's conservatism?
* **Cost model fidelity**: What's the minimum viable cost model?
