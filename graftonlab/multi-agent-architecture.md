---
title: "Multi-Agent Architecture"
description: "DAG-based multi-agent architecture with stable interfaces and recursive decomposition"
---

## TL;DR

Current Grafton's linear pipeline is designed for a fast pass, not for scale to multi-domain, multi-level designs — each new verification layer or domain requires re-plumbing the whole chain. This document proposes a **DAG of agents** behind **stable interfaces** (agent context, typed messages, file-system workspaces). Once the interfaces are in place, decomposition strategies, verification layers, uncertainty models, and scheduling policies become **plug-ins** — swappable without touching the orchestration layer. This also enables **comparative evaluation at scale**: run the same intent through different agent configurations on the eval dataset to measure which structures actually work.

**Core design ideas:**

* **Recursive decomposition** — a single Decomposition Agent type handles all hierarchy levels; it decomposes, oversees children, gathers results, and propagates evidence upward
* **Thin orchestrator** — mostly programmatic (DAG, scheduling, routing); LLM only for skill selection and error triage
* **Stable interfaces** — agent context, typed messages, file-system workspaces define the contract; everything behind them is swappable
* **Four-layer verification** — Physics Ontology → Datalog rules → Z3 abductive repair → LLM neural bridge; deterministic where possible, probabilistic where necessary
* **Wave-parallel execution** — topological ordering with coupling-aware grouping; independent branches run in parallel, coupled branches sequenced

A lightweight **Orchestrator** manages the execution graph and agent lifecycle. **Decomposition Agents** recursively break goals into subgoals (one per hierarchy node, implementing `SOLVE`). Leaf nodes spawn specialized agents (**CAD**, **Verifier**, **Assembly**, **Communication**). Execution proceeds in **topological waves** — independent branches in parallel, coupled branches sequenced.

**Key idea**: the Decomposition Agent is the recursive unit — the Orchestrator manages logistics; all domain reasoning stays inside agents.


## 1. Overview

Current Grafton's linear pipeline (`cli.py` → GRS → contracts → artifact → verify) is designed for a fast pass on single-part designs. It is challenging to scale up to system which requires hierarchical decomposition, cross-subsystem coupling, or domain-specific verification. Adding a new domain or verification strategy means re-plumbing the pipeline end-to-end.

We propose a DAG-based multi-agent architecture with **stable interfaces** — agent context, typed messages, file-system workspaces — so that everything behind those interfaces becomes a **swappable plug-in**:

* **Decomposition heuristics** — how goals break into subgoals (domain-specific, swappable per skill)
* **Verification layers** — which checks run and how failures feed back (Datalog rules, Z3 repair, VLM scoring)
* **Uncertainty propagation** — how confidence rolls up from leaves to root (min, Bayesian, weighted)
* **Scheduling policies** — how the DAG is parallelized (coupling-aware waves, budget-aware ordering)

This separation enables running **controlled experiments**: same intent, different agent configurations, measured on the eval benchmark at scale.

### Design Principles

**From** [**Kimi K2.5**](https://www.kimi.com/blog/kimi-k2-5.html)**:**

* Orchestrator dynamically creates subagents — no predefined rigid roles (Fig 1)
* Critical path metric: `latency = orchestration_overhead + slowest_subagent_per_wave`
* Only parallelize when it shortens the critical path (Fig 8, Fig 9)

>  ![Kimi K2.5 Orchestrator Architecture — orchestrator dynamically creates and assigns tasks to typed subagents for parallel execution](https://statics.moonshot.cn/blogs/k2-5/orchestrator-1.png) *Fig 1. Kimi K2.5 agent swarm: orchestrator creates typed subagents (researchers, fact-checkers, etc.), assigns up to 100 parallel tasks in waves, gathers results. Source:* [*Kimi K2.5 Tech Blog*](https://www.kimi.com/blog/kimi-k2-5.html)

**From** [**Manus Context Engineering**](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)**:**

* File system as externalized unlimited memory (Fig 2)
* Keep errors in context — model learns from mistakes implicitly (Fig 3)
* `todo.md` trick: push global plan into model's recent attention span to prevent goal drift (Fig 4)
* KV-cache hit rate is the #1 optimization target — append-only context preserves cache (Fig 5)
* Mask tools instead of removing them to preserve KV-cache prefix (Fig 6)

**From** [**Anthropic Multi-Agent Systems**](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them)**:**

* Decompose by context boundaries, not by work type
* Three valid reasons for multi-agent: context protection, parallelization, specialization
* Verification subagent pattern: main agent works → verifier blackbox-tests → iterate
* Start simple; add agents only when evidence shows need

### Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                          ORCHESTRATOR                                 │
│                                                                       │
│  Responsibilities:                                                    │
│    - Agent lifecycle (create, monitor, kill)                          │
│    - DAG state management                                             │
│    - Wave scheduling (topological order + coupling awareness)         │
│    - Global σ tracking + budget enforcement                           │
│    - Message routing between agents                                   │
│                                                                       │
│  Tools: create_agent, assign_task, kill_agent,                        │
│         route_message, query_dag, check_global_σ                      │
└────────────┬──────────────────────────────────────────────────────────┘
             │
             │ create agents + assign tasks (in waves)
             │
  ┌──────────┼───────────┬──────────────┬──────────────┐
  │          │           │              │              │
┌─▼────┐ ┌──▼───┐ ┌─────▼─────┐ ┌─────▼────┐ ┌──────▼──────┐
│Decomp│ │Decomp│ │  Decomp   │ │   CAD    │ │  Verifier   │
│Agent │ │Agent │ │  Agent    │ │  Agent   │ │  Agent      │
│(Sys) │ │(Lith)│ │  (Depo)   │ │  (leaf)  │ │  (leaf)     │
│      │ │      │ │           │ │          │ │             │
│GRS,  │ │GRS,  │ │  GRS,     │ │CadQuery, │ │DFM rules,   │
│Z3,   │ │Z3,   │ │  Z3,      │ │Render,   │ │VLM scoring, │
│Rsch  │ │Rsch  │ │  Rsch     │ │Screensht,│ │Geometry     │
│      │ │      │ │           │ │DFM       │ │backend      │
└──────┘ └──────┘ └───────────┘ └──────────┘ └─────────────┘
```

### Worked Example: Single-Engine Aircraft

Walk through `--intent "Design a single-engine propeller aircraft for 4 passengers"` to see how the architecture handles two levels of decomposition, parallel wave execution, coupling between subsystems, and leaf-level verification.

#### Level 0 → Level 1: Aircraft → Subsystems

```
User Intent: "Design a single-engine propeller aircraft for 4 passengers"
                              │
                     ┌────────▼────────┐
                     │   ORCHESTRATOR   │  creates root Decomp Agent
                     └────────┬────────┘
                              │
                   ┌──────────▼────────────┐
                   │  Decomp Agent (root)  │
                   │                       │
                   │  run_grsc() →         │
                   │    G: 4-pax, 800nm    │
                   │       range, 150kt    │
                   │    R: MTOW≤1200kg,    │
                   │       stall≤55kt      │
                   │    S: wing_area 14m²  │
                   │       engine ≥180hp   │
                   │  verify_csp() → SAT   │
                   │  is_leaf()? NO        │
                   │                       │
                   │  request_children →   │
                   │    5 child Decomp     │
                   │    Agents with        │
                   │    coupling groups    │
                   └──────────┬────────────┘
                              │
         ┌──────────┬────────┼─────────┬─────────┐
         │          │        │         │         │
    ┌────▼───┐ ┌───▼────┐ ┌──▼───┐ ┌───▼───┐ ┌───▼────┐
    │ Decomp │ │ Decomp │ │Decomp│ │Decomp │ │ Decomp │
    │ Wing   │ │Fuselage│ │Engine│ │ Tail  │ │Landing │
    │        │ │        │ │Mount │ │       │ │ Gear   │
    └────┬───┘ └───┬────┘ └──┬───┘ └───┬───┘ └───┬────┘
         │         │         │         │         │
         coupling: ─────────────────────
         wing↔fuselage (structural frame)
         wing↔tail (aero balance)
         engine_mount↔fuselage (load path)
```

The orchestrator schedules these as **Wave 1** but respects coupling: Landing Gear (independent) can run fully in parallel with everything else. Wing ↔ Fuselage ↔ Engine Mount share coupling groups and exchange `COUPLING_ALERT` messages (§4.3.3) as they resolve shared constraints (e.g., wing-fuselage attachment loads propagate to fuselage frame sizing).

#### Level 1 → Level 2: Wing → Subcomponents

Each Level-1 Decomp Agent further decomposes. Here is the Wing subtree:

```
                   ┌──────────────────────┐
                   │  Decomp Agent (Wing)  │
                   │                       │
                   │  run_grsc() →         │
                   │    G: generate lift   │
                   │       ≥ 1200kg @ 55kt │
                   │    R: spar must carry │
                   │       3.8g ultimate   │
                   │    S: NACA 2412,      │
                   │       span 10.5m,     │
                   │       Al-2024-T3      │
                   │  verify_csp() → SAT   │
                   │  is_leaf()? NO        │
                   │                       │
                   │  request_children →   │
                   │    4 leaf nodes       │
                   └──────────┬───────────┘
                              │
           ┌──────────┬───────┴───────┬──────────┐
           │          │               │          │
      ┌────▼───┐ ┌───▼────┐    ┌─────▼───┐ ┌───▼─────┐
      │  Spar  │ │  Skin  │    │  Flap   │ │ Aileron │
      │ (leaf) │ │ (leaf) │    │ (leaf)  │ │ (leaf)  │
      └────┬───┘ └───┬────┘    └────┬────┘ └────┬────┘
           │         │              │            │
           ▼         ▼              ▼            ▼
       CAD Agent  CAD Agent     CAD Agent    CAD Agent
       + Verify   + Verify      + Verify     + Verify
```

Spar and Skin share coupling (skin transfers loads to spar), so the orchestrator sequences them. Flap and Aileron are independent — they run in **Wave 2b** in parallel.

#### Leaf-Level: CAD → Verify Loop (Spar)

At the leaf, the Decomp Agent spawns a CAD Agent then a Verifier:

```
Decomp Agent (Spar)           Orchestrator           CAD Agent         Verifier Agent
       │                           │                      │                   │
       │  REQUEST_AGENT(cad, {     │                      │                   │
       │    specs: Al-2024 I-beam, │                      │                   │
       │    3.8g ultimate,         │                      │                   │
       │    span 5.25m })          │                      │                   │
       ├──────────────────────────►│  create + assign     │                   │
       │                           ├─────────────────────►│                   │
       │                           │                      │ cadquery_exec()   │
       │                           │                      │ screenshot()      │
       │                           │                      │ dfm_check()       │
       │                           │                      │ export_step()     │
       │                           │   TASK_COMPLETE      │                   │
       │                           │◄─────────────────────┤                   │
       │                           │                      │                   │
       │                           │  create verifier     │                   │
       │                           │  (gets ONLY .step    │                   │
       │                           │   + contracts)       │                   │
       │                           ├─────────────────────────────────────────►│
       │                           │                                          │ DFM rules
       │                           │                                          │ VLM scoring
       │                           │                                          │ geometry check
       │                           │                                          │
       │                           │            TASK_COMPLETE(FAIL,           │
       │                           │◄─────────────────────────────────────────┤
       │                           │             wall=1.5mm < 2mm,            │
       │   CHILD_RESULT(FAIL,      │             feedback: "increase          │
       │    feedback: ...)         │              flange thickness")          │
       │◄──────────────────────────┤                                          │
       │                           │                                          │
       │  ── retry: new CAD Agent with feedback ──                            │
       │  REQUEST_AGENT(cad, {     │                      │                   │
       │    specs: ...,            │                      │                   │
       │    feedback: "increase    │  create + assign     │                   │
       │     flange thickness" })  │─────────────────────►│                   │
       │                           │                      │ regenerates       │
       │                           │                      │ thicker flanges   │
       │                           │   TASK_COMPLETE      │                   │
       │                           │◄─────────────────────┤                   │
       │                           │  create verifier     │                   │
       │                           ├─────────────────────────────────────────►│
       │                           │            TASK_COMPLETE(PASS, σ=0.91)   │
       │                           │◄─────────────────────────────────────────┤
       │   CHILD_RESULT(PASS)      │                                          │
       │◄──────────────────────────┤                                          │
       │                           │                                          │
       │  merges evidence,         │                                          │
       │  returns to parent Wing   │                                          │
```

#### Bottom-Up Rollup

Results propagate upward through two levels back to the root:

```
Leaf Agents (spar, skin, flap, aileron)
    │ each returns evidence + σ
    ▼
Decomp Agent (Wing)
    │ gathers 4 child results
    │ check_consistency(): spar-skin load path OK
    │ merge_evidence(): σ_wing = min(0.91, 0.88, 0.93, 0.90) = 0.88
    │ TASK_COMPLETE → orchestrator
    ▼
... (same for Fuselage, Engine Mount, Tail, Landing Gear)
    ▼
Decomp Agent (root)
    │ gathers 5 subsystem results
    │ check_consistency(): wing-fuselage attachment loads match,
    │   engine mount loads within fuselage frame capacity,
    │   tail moment arm balances wing center of pressure
    │ merge_evidence(): σ_aircraft = 0.85
    │ TASK_COMPLETE → orchestrator
    ▼
Orchestrator: DAG complete → SystemResult
```

#### Wave Execution Summary

| Wave | Agents running | Notes |
|------|----------------|-------|
| **1** | Root Decomp    | GRSC → decompose into 5 subsystems |
| **2** | Wing, Fuselage, Engine Mount, Tail, Landing Gear (Decomps) | Coupled groups exchange `COUPLING_ALERT`; Landing Gear fully independent |
| **3a** | Wing Spar, Skin (CAD+Verify, sequenced — coupled) | Spar resolves first, Skin uses spar attachment interface |
| **3b** | Flap, Aileron (CAD+Verify, parallel — independent) | Run simultaneously with 3a |
| **3c** | Fuselage subparts, Tail subparts, etc. (parallel with 3a/b) | Other subtrees execute concurrently |
| **4** | Wing Decomp gathers children, Fuselage gathers, ... | Each Level-1 agent merges evidence |
| **5** | Root Decomp gathers all 5 subsystems | Cross-subsystem consistency check, final σ |


## 2. Agent Types

### 2.1 Orchestrator

Lightweight process manager inspired by Kimi's orchestrator (Fig 1). Dynamically creates subagents in topological waves. Mostly deterministic code (state machine, graph algorithms, file I/O) with LLM calls limited to semantic coordination tasks.

**State:** `DAG` (execution graph), `agent_registry`, `global_σ`, `budget_tracker`, `message_queue`

**Scope:**

| Orchestrator | Decomp Agent |
|--------------|--------------|
| DAG state + wave scheduling | Goal decomposition (+ LLM) |
| Agent lifecycle (create/kill) | Spec exploration + selection (+ LLM) |
| Message routing | GRSC generation (+ LLM) |
| Budget enforcement | Constraint verification (Z3 + LLM) |
| Skill selection (+ LLM) | Evidence merging + rollup (+ LLM) |
| Error triage (+ LLM) | Cross-child consistency (+ LLM) |

Boundary heuristic: **semantic coordination** (understanding the task) → Orchestrator. **Domain-specific reasoning** (understanding the engineering) → Decomp Agent. The boundary is intentionally soft.

**Skills:** Domain prompt packs on disk (`shared/skills/{domain}/prompt.md`). The Orchestrator resolves the Decomp Agent's domain hint to a skill file and injects it into the child's system prompt prefix (KV-cache friendly — stable prefix).

**Tools:** `create_agent`, `assign_task`, `kill_agent`, `route_message`, `query_dag`, `check_global_σ`, `load_skill` (+ LLM), `triage_error` (+ LLM)

**Does NOT do:** domain reasoning, direct tool use (no CadQuery, Z3, LLM judge), decomposition decisions.

### 2.2 Decomposition Agent

Recursive unit — one per hierarchy node, implementing `SOLVE(G_v, C_v, F)`. Owns all domain reasoning: GRSC, specs, constraints, evidence, consistency. Evolves frequently as new domains are added; the Orchestrator structure remains stable.

**Lifecycle:** lives until its entire subtree completes.

**Role:**



1. Takes goal + parent contracts + constraint budget
2. Explores spec alternatives (may spawn Research Agent)
3. Runs GRSC → commits specs + contracts
4. Verifies via CSP (Z3 + LLM judge)
5. Leaf → spawns CAD/Buy agent; non-leaf → spawns child Decomps with **domain hints** (e.g., `"domain": "aero"`) for skill resolution
6. Oversees subgraph: gathers child results, checks cross-child consistency, merges evidence upward
7. Handles budget violations (re-decompose or escalate)

**Tools:** `run_grsc`, `explore_specs`, `verify_csp`, `request_children` (includes domain hints), `gather_results`, `check_consistency`, `send_message`, `escalate`

### 2.3 Research Agent

Short-lived. Spawned by Decomp Agent for spec exploration. Proposes competing spec-decomposition pairs with cost/feasibility estimates.

**Tools:** `web_search`, `knowledge_lookup`, `llm_reason`, `cost_estimate`

### 2.4 CAD Agent

Leaf-level tool user. Implements the Atomic Design Loop — generates CadQuery code, renders geometry, runs DFM checks per operation.

**Tools:** `cadquery_exec`, `render`, `screenshot`, `dfm_check`, `export_step`

### 2.5 Verifier Agent

Blackbox tester (Anthropic verification pattern). Receives only artifact + spec — never implementation context. Internally organized as four layers per the [Symbolic Reasoning](./Symbolic%20Reasoning%20for%20First-Principles%20Verification.md) design:



1. **Physics Ontology** — structured knowledge base populating the universal constraint set C
2. **Deductive Rules** (Datalog via Scallop/Souffle) — implements the relevance function ρ to surface *all* applicable checks deterministically (σ = 1.0); LLMs cannot serve as ρ (non-deterministic, incomplete, no audit trail)
3. **Abductive Repair** (Z3 Optimize, ASP) — on UNSAT, computes minimum-cost parameter changes rather than just reporting "constraint failed"; feeds enhanced feedback back to the Decomp Agent's retry loop
4. **Neural Bridge** (LLM) — semantic-typed constraints (σ &lt; 1.0): fact extraction, hypothesis generation, gap-filling for novel situations

Returns evidence nodes (known knowns), Unknown nodes (known unknowns — detected gaps in C), and a novelty margin ε for unknown unknowns.

**Tools:** `datalog_eval`, `dfm_rules`, `vlm_score`, `geometry_backend`, `z3_check`, `z3_repair`, `screenshot`, `render`

### 2.6 Assembly Agent

Spawned when child parts must integrate. Handles interference checks, tolerance stack-up, coupling analysis, assembly sequencing.

**Tools:** `boolean_intersection`, `tolerance_montecarlo`, `modal_analysis`, `assembly_sequence`

### 2.7 Communication Agent

Spawned for external component procurement. Searches supplier databases, validates against contracts, produces acceptance test plans.

**Tools:** `supplier_search`, `datasheet_parse`, `acceptance_test_plan`


## 3. Context Management

Organized by implementation priority. §3.1 is the minimum viable context system; §3.2 adds operational maturity; §3.3 is premature at small scale.

### 3.1 Core — Build First

Five items. Three need implementation; two are free disciplines.

**AgentContext dataclass** — immutable boundary conditions set by the orchestrator at agent creation. This is the contract between orchestrator and agent:

```python
@dataclass
class AgentContext:
    """Immutable context provided by orchestrator at agent creation."""

    # Identity
    agent_id: str               # unique, e.g. "decomp_litho_002"
    agent_type: str             # "decomposition", "cad", "verifier", etc.
    dag_node_id: str            # which DAG node this agent serves

    # From parent (read-only boundary conditions)
    goal: str                   # G_v — what this agent must achieve
    parent_contracts: list[dict]  # contracts this agent must satisfy
    resolved_constraints: set[str]  # C_resolved from ancestors (don't re-verify)
    budget: BudgetAllocation    # cost, time, token, σ budgets

    # Workspace (agent-owned, file-system externalized)
    workspace: Path             # ./agent_runs/{agent_id}/

    # Model configuration
    model: str                  # which LLM to use for this agent
    max_turns: int              # max tool-call rounds before forced stop
    temperature: float          # LLM temperature


@dataclass
class BudgetAllocation:
    """Resource budget allocated by parent Decomposition Agent."""

    cost_usd: float             # max spend on LLM + tools
    time_seconds: float         # wall-clock deadline
    token_budget: int           # max total tokens (input + output)
    sigma_threshold: float      # min confidence to pass (τ_local)
    max_retries: int            # T_max for feedback loop
```

**Workspace + file tools** — each agent gets a workspace directory as externalized memory (Fig 2). Three file tools sandboxed to the agent's workspace: `read_file(path)`, `write_file(path, content)`, `list_files(path)`. After context compression, `list_files` lets the agent rediscover its own state.

>  ![Manus file system as externalized agent context/memory](https://d1oupeiobkpcny.cloudfront.net/user_upload_by_module/markdown/310708716691272617/sBITCOxGnHNUPHTD.png) *Fig 2. "Use the File System as Context": Left (bad) — large observations bloat the context window. Right (good) — observations are offloaded to named files; the context stays slim and the agent reads files on demand. Source:* [*Manus Context Engineering*](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)

```
agent_runs/
├── orchestrator/
│   ├── dag.json                # global DAG state
│   ├── agent_registry.json     # all agents + status
│   ├── waves.jsonl             # wave execution history
│   └── trace.jsonl             # orchestrator decisions
│
├── decomp_system_001/
│   ├── context.md              # append-only working memory
│   ├── todo.md                 # current plan (Manus trick)
│   ├── grsc_output.json        # GRS + contracts produced
│   ├── spec_alternatives.json  # research results
│   ├── subgraph_state.json     # child status tracking
│   ├── inbox.jsonl             # messages received
│   ├── outbox.jsonl            # messages sent
│   └── trace.jsonl             # full event log
│
├── cad_mirror_mount_015/
│   ├── context.md
│   ├── todo.md
│   ├── artifacts/
│   │   ├── mirror_mount.py     # CadQuery code
│   │   ├── mirror_mount.step   # exported STEP
│   │   ├── render_iso.png      # rendered views
│   │   └── ops_program.json    # operations program
│   ├── inbox.jsonl
│   ├── outbox.jsonl
│   └── trace.jsonl
│
└── verifier_mount_016/
    ├── context.md
    ├── evidence/
    │   ├── dfm_results.json
    │   └── vlm_scores.json
    ├── inbox.jsonl
    ├── outbox.jsonl
    └── trace.jsonl
```

**Context isolation** — each agent has isolated context; cannot read another agent's conversation or workspace:

| Rule | Rationale |
|------|-----------|
| Agent CANNOT read another agent's `messages` | Context protection (Anthropic) |
| Agent CAN read its own `workspace/` files | Externalized memory (Manus) |
| Agent CANNOT write to another agent's `workspace/` | No side-channel mutation |
| Agent CAN send structured messages via orchestrator | Controlled communication |
| Agent CAN read shared read-only resources | Tools, not context |
| Orchestrator CAN read any agent's `workspace/` | Monitoring, not injection |
| Orchestrator CANNOT modify agent's `messages` | Preserve append-only invariant |

**Append-only context discipline** — within a session, context only grows. Never mutate or delete existing messages. Free to implement: just don't call splice/edit on the message list. Prevents subtle bugs where downstream reasoning references deleted context.

**Errors stay in context** — failed tool calls, verification failures, and constraint violations remain visible in the message history. The model sees what didn't work and adjusts (Fig 3). Free to implement: just don't filter them out.

>  ![Manus error retention in context — failed actions remain visible for implicit learning](https://d1oupeiobkpcny.cloudfront.net/user_upload_by_module/markdown/310708716691272617/dBjZlVbKJVhjgQuF.png) *Fig 3. "Keep the Wrong Stuff In": Left (bad) — failed attempts are discarded, so the model retries the same approach. Right (good) — failed actions (red) remain inline in context, so the model sees what didn't work and tries something different. Source:* [*Manus Context Engineering*](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)

### 3.2 Extended — Build Second

[**todo.md**](http://todo.md) **recitation** — agent maintains a running todo list in its context window, preventing goal drift on long (50+ tool call) sequences. Re-inject after every context compression (Fig 4).

>  ![Manus attention recitation — re-injecting objectives between action steps to keep them in recent context](https://d1oupeiobkpcny.cloudfront.net/user_upload_by_module/markdown/310708716691272617/OYpTzfPZaBeeWFOx.png) *Fig 4. "Manipulate Attention Through Recitation": Left (bad) — objectives only appear at the top; after many steps they fall out of the model's attention window. Right (good) — objectives are re-injected between action steps, keeping them in recent context. In our architecture, each agent's* `*todo.md*` *serves this role — re-injected after every context compression to prevent goal drift. Source:* [*Manus Context Engineering*](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)

**Context lifecycle** — agent creation → fixed system prompt → append-only loop → final result:

```
                        Agent Created
                             │
                    ┌────────▼────────┐
                    │ System prompt    │  ← FIXED: never changes
                    │ + AgentContext   │
                    │ + parent_contracts│
                    │ + tool definitions│
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │ Initial task    │  ← From orchestrator's assign_task
                    │ message         │
                    └────────┬────────┘
                             │
              ┌──────────────▼──────────────┐
              │       APPEND-ONLY LOOP       │
              │                              │
              │  1. Agent reasons + tool call │
              │  2. Tool result appended      │
              │  3. Agent updates todo.md     │
              │  4. Agent writes to workspace │
              │  5. Repeat                    │
              │                              │
              │  On inbox message:            │
              │    → appended as user message │
              │    → agent processes it       │
              │                              │
              │  On context window pressure:  │
              │    → summarize old messages   │
              │    → write summary to context.md
              │    → keep recent N messages   │
              │    → re-read context.md       │
              └──────────────┬──────────────┘
                             │
                    ┌────────▼────────┐
                    │ Agent returns   │
                    │ final result    │  → structured output to orchestrator
                    │ + workspace     │  → persisted for post-mortem
                    └─────────────────┘
```

**Per-agent trace.jsonl** — every agent writes structured events (tool calls, LLM requests, decisions, errors) to `trace.jsonl`. Enables post-mortem debugging without replaying the agent's context. Cheap: just append one JSON line per event.

**Budget tracking** — `BudgetAllocation` enforced at runtime: orchestrator checks token/cost/time spend after each tool call. Agent receives `BUDGET_WARNING` at 80% and `BUDGET_EXCEEDED` at 100%. Prevents runaway agents from draining the entire run's budget.

**Read-only shared resources** — materials, processes, standards, calibration data shared across agents but immutable. Any agent can `read_shared(path)` but never write. See §10 for full directory layout.

**Cross-agent file sharing** — agents share **files** via the orchestrator, not context. Orchestrator copies a file from one agent's workspace to another's `shared_from/` directory and sends a `FILE_SHARED` notification. The receiving agent reads the exact file by path.

### 3.3 Scale Optimization — Build When Needed

These matter when running 10+ agents or 100+ tool calls per agent. At small scale (2-5 agents, &lt;50 tool calls), skip.

**KV-cache engineering** — stable prompt prefix (system prompt + contracts + tool defs) never changes mid-session (Fig 5). Append-only context preserves KV-cache; modifying or removing earlier messages invalidates everything downstream. Tool masking over tool removal (Fig 6): when available tools change between phases, keep all tool definitions but use logit masking to disable unavailable ones.

>  ![Manus KV-cache design — append-only context preserves cache hits across steps](https://d1oupeiobkpcny.cloudfront.net/user_upload_by_module/markdown/310708716691272617/OhdKxGRSXCcuqOvz.png) *Fig 5. "Design Around the KV-Cache": Left (bad) — replacing earlier context between steps causes full cache miss. Right (good) — append-only context preserves the prefix, so only new tokens at the end are cache misses. Source:* [*Manus Context Engineering*](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)

>  ![Manus tool masking — keep tool definitions stable, mask unavailable ones instead of removing](https://d1oupeiobkpcny.cloudfront.net/user_upload_by_module/markdown/310708716691272617/cWxINCvUfrmlbvfV.png) *Fig 6. "Mask, Don't Remove": Left (bad) — changing which tool definitions appear in the prompt between steps causes KV-cache miss on the entire prefix. Right (good) — all tool definitions stay in the prompt; unavailable tools are greyed out via logit masking. The prefix remains identical, preserving cache. Source:* [*Manus Context Engineering*](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)

**ContextManager compression class** — manages automatic context summarization when approaching window limit. Summarize-and-externalize: write older turns to `context.md`, keep summary + recent N turns, re-inject `todo.md`. Never summarize the system prompt prefix. Preserve file paths and constraint IDs as cheap tokens for on-demand re-reading.

```python
class ContextManager:
    """Manages context window for a single agent."""

    def __init__(self, agent_id: str, workspace: Path, max_tokens: int):
        self.agent_id = agent_id
        self.workspace = workspace
        self.max_tokens = max_tokens
        self.messages: list[Message] = []
        self.system_prompt: str = ""  # fixed, never compressed

    def append(self, message: Message) -> None:
        """Append-only. Never mutate existing messages."""
        self.messages.append(message)
        self._check_pressure()

    def _check_pressure(self) -> None:
        """If approaching limit, compress older messages."""
        total = self._estimate_tokens()
        if total > self.max_tokens * 0.8:  # 80% threshold
            self._compress()

    def _compress(self) -> None:
        """Summarize old messages, externalize to context.md."""
        old = self.messages[:-self.keep_recent]
        summary = self._summarize(old)

        self._append_to_file(
            self.workspace / "context.md",
            f"\n## Context Summary (turn {len(self.messages)})\n{summary}\n"
        )

        self.messages = [
            Message(role="system", content=f"[Previous context summarized in context.md]\n{summary}"),
            *self.messages[-self.keep_recent:]
        ]

        # Re-inject todo.md into recent attention
        todo = (self.workspace / "todo.md").read_text()
        self.messages.append(
            Message(role="system", content=f"[Current plan]\n{todo}")
        )

    def inject_inbox_message(self, msg: AgentMessage) -> None:
        """Route an inter-agent message into this agent's context."""
        self.append(Message(
            role="user",
            content=f"[Message from {msg.sender_id}]\n{msg.content}"
        ))
```

**Compression → externalization cycle** — full diagram in §10.5. Key idea: at 80% capacity, summarize old turns → write structured data to workspace files → keep recent N turns → re-inject `context.md` summary + `todo.md`.

**Repetitive pattern prevention** — when an agent processes many similar subtasks, it can get "few-shotted by its own history" (Fig 7). Agent isolation partially solves this — each child starts with a clean context. For sequential children, context compression between iterations naturally varies the context.

>  ![Manus diversity injection — varied observation phrasing prevents repetitive pattern lock-in](https://d1oupeiobkpcny.cloudfront.net/user_upload_by_module/markdown/310708716691272617/IIyBBdwwuMDJUnUc.png) *Fig 7. "Don't Get Few-Shotted": Left (bad) — identical action/observation patterns across k iterations cause the model to mechanically repeat the same behavior. Right (good) — observation phrasing is varied to break the repetitive pattern. In our architecture, agent isolation naturally prevents this — each CAD agent starts with a clean context. Source:* [*Manus Context Engineering*](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)


## 4. Inter-Agent Communication

### 4.1 Communication Patterns

All agent communication flows through the orchestrator's message router. No direct agent-to-agent connections.

```
Agent A                 Orchestrator              Agent B
   │                        │                        │
   │  send_message(B, msg)  │                        │
   ├───────────────────────►│                        │
   │                        │  route + validate      │
   │                        │  write to B's inbox    │
   │                        ├───────────────────────►│
   │                        │                        │ processes msg
   │                        │                        │ in next turn
   │                        │  send_message(A, reply)│
   │                        │◄───────────────────────┤
   │  inbox: reply from B   │                        │
   │◄───────────────────────┤                        │
```

### 4.2 Message Types

```python
@dataclass
class AgentMessage:
    """Structured message between agents."""

    id: str                     # unique message ID
    sender_id: str              # agent_id of sender
    receiver_id: str            # agent_id of receiver (or "orchestrator")
    msg_type: MessageType       # see enum below
    content: dict               # typed payload (schema depends on msg_type)
    timestamp: datetime
    reply_to: str | None        # message ID this replies to
    priority: int               # 0=normal, 1=urgent, 2=critical


class MessageType(str, Enum):
    # Orchestrator → Agent
    TASK_ASSIGNMENT = "task_assignment"      # initial task
    CHILD_RESULT = "child_result"           # child agent completed
    BUDGET_UPDATE = "budget_update"         # budget reallocation
    KILL = "kill"                           # terminate

    # Agent → Orchestrator
    REQUEST_CHILDREN = "request_children"   # spawn child agents
    REQUEST_AGENT = "request_agent"         # spawn a specific agent type
    TASK_COMPLETE = "task_complete"         # agent finished
    ESCALATE = "escalate"                   # can't resolve, need parent help
    STATUS_UPDATE = "status_update"         # progress report

    # Agent → Agent (via orchestrator)
    QUERY = "query"                         # ask another agent for info
    QUERY_RESPONSE = "query_response"       # response to query
    CONSTRAINT_UPDATE = "constraint_update" # bottom-up propagation
    EVIDENCE = "evidence"                   # share evidence
    COUPLING_ALERT = "coupling_alert"       # coupling hub notification
```

### 4.3 Communication Flows

#### 4.3.1 Parent → Child: Decomposition

```python
# Decomposition Agent decides to decompose into children
# Step 1: Agent sends REQUEST_CHILDREN to orchestrator

msg = AgentMessage(
    sender_id="decomp_litho_002",
    receiver_id="orchestrator",
    msg_type=MessageType.REQUEST_CHILDREN,
    content={
        "children": [
            {
                "goal": "Provide EUV light source >= 250W at 13.5nm",
                "agent_type": "decomposition",
                "constraints": ["power_250w", "wavelength_13.5nm", "lifetime_30khrs"],
                "parent_contracts": [{"id": "ct_euv_01", "guarantees": [...]}],
                "budget": {"cost_usd": 500, "sigma_threshold": 0.85},
                "coupling_groups": ["litho_frame"],  # for parallelism decisions
            },
            {
                "goal": "Project EUV image with wavefront error < 0.3nm RMS",
                "agent_type": "decomposition",
                "constraints": ["wavefront_0.3nm", "magnification_4x"],
                "parent_contracts": [{"id": "ct_proj_01", "guarantees": [...]}],
                "budget": {"cost_usd": 400, "sigma_threshold": 0.90},
                "coupling_groups": ["litho_frame"],
            },
            # ... more children
        ],
        "independence_matrix": {
            # Which children can run in parallel
            # Children sharing coupling_groups may need sequencing
            "euv_source": {"shares_coupling_with": ["projection_optics"]},
            "dose_control": {"independent": True},
        }
    }
)
```

```python
# Step 2: Orchestrator creates child agents and assigns tasks

for child_spec in msg.content["children"]:
    child_ctx = AgentContext(
        agent_id=generate_id(child_spec["agent_type"]),
        agent_type=child_spec["agent_type"],
        goal=child_spec["goal"],
        parent_contracts=child_spec["parent_contracts"],
        resolved_constraints=parent_agent.resolved_constraints | parent_agent.own_resolved,
        budget=BudgetAllocation(**child_spec["budget"]),
        workspace=AGENT_RUNS / child_ctx.agent_id,
    )
    orchestrator.create_agent(child_ctx)
    orchestrator.assign_task(child_ctx.agent_id, child_spec["goal"])
```

```python
# Step 3: When child completes, orchestrator routes result to parent

child_result_msg = AgentMessage(
    sender_id="orchestrator",
    receiver_id="decomp_litho_002",  # parent
    msg_type=MessageType.CHILD_RESULT,
    content={
        "child_id": "decomp_euv_source_003",
        "status": "complete",
        "sigma": 0.88,
        "evidence": [...],
        "unknowns": [...],
        "budget_actual": {"cost_usd": 420},
        "artifacts": ["agent_runs/decomp_euv_source_003/artifacts/"],
    }
)
```

#### 4.3.2 Child → Parent: Bottom-Up Propagation

When a child discovers a constraint violation or budget overrun, it escalates to its parent Decomposition Agent.

```python
# CAD Agent discovers manufacturing cost exceeds allocation
escalation = AgentMessage(
    sender_id="cad_mirror_mount_015",
    receiver_id="orchestrator",  # routed to parent decomp agent
    msg_type=MessageType.ESCALATE,
    content={
        "reason": "budget_violation",
        "detail": "Single-point diamond turning required for 0.1nm roughness, +$50K",
        "current_budget": 30000,
        "required_budget": 80000,
        "alternatives": [
            "Accept lower surface quality (0.3nm) — may violate parent contract",
            "Switch material from Zerodur to fused silica — different CTE",
            "Use conventional polishing + ion beam figuring — $45K, 0.15nm achievable",
        ],
    },
    priority=2,  # critical — blocks progress
)
```

```python
# Orchestrator routes to the appropriate parent Decomposition Agent
# Parent agent receives this in its inbox and must decide:
#   1. Reallocate budget from siblings
#   2. Accept alternative and re-verify parent contract
#   3. Escalate further up the hierarchy
```

#### 4.3.3 Sibling → Sibling: Coupling Notification

When one agent's decisions affect a sibling through a shared coupling hub.

```python
# EUV Source decomp agent discovers high thermal output
coupling_alert = AgentMessage(
    sender_id="decomp_euv_source_003",
    receiver_id="decomp_projection_optics_004",  # sibling
    msg_type=MessageType.COUPLING_ALERT,
    content={
        "coupling_hub": "litho_frame",
        "parameter": "thermal_output",
        "value": "2.5kW radiated into frame",
        "affected_contracts": ["ct_proj_01"],
        "recommendation": "Account for 0.15nm additional thermal drift in projection budget",
    },
)

# Routed by orchestrator — orchestrator logs this coupling interaction
# Receiving agent incorporates into its constraint set
```

#### 4.3.4 Decomposition Agent → Verifier: Handoff

```python
# Decomposition Agent at leaf level hands off to verifier
# Key: verifier gets ONLY spec + artifact, NOT implementation history

handoff = AgentMessage(
    sender_id="decomp_collector_008",
    receiver_id="orchestrator",
    msg_type=MessageType.REQUEST_AGENT,
    content={
        "agent_type": "verifier",
        "task": "Verify mirror mount artifact against contracts",
        "context": {
            # ONLY these — no CAD agent conversation, no design rationale
            "artifact_path": "agent_runs/cad_mirror_mount_015/artifacts/mirror_mount.step",
            "contracts": [{"id": "ct_mount_01", "guarantees": [...]}],
            "specs": [{"id": "spec_drift", "value": "0.5nm/K", "tolerance": "±0.1nm/K"}],
        },
        "return_to": "decomp_collector_008",  # send results back here
    },
)
```

### 4.4 Message Router

```python
class MessageRouter:
    """Routes messages between agents. All communication flows through here."""

    def __init__(self, agent_registry: dict[str, AgentInfo], dag: DAG):
        self.registry = agent_registry
        self.dag = dag
        self.message_log: list[AgentMessage] = []

    def route(self, msg: AgentMessage) -> None:
        """Route a message to its destination."""

        # Log every message for observability
        self.message_log.append(msg)

        # Validate sender exists and is alive
        if msg.sender_id != "orchestrator":
            assert msg.sender_id in self.registry
            assert self.registry[msg.sender_id].status == "running"

        # Validate receiver exists
        if msg.receiver_id == "orchestrator":
            self._handle_orchestrator_message(msg)
            return

        assert msg.receiver_id in self.registry

        # Validate communication is allowed
        self._check_communication_policy(msg)

        # Write to receiver's inbox
        inbox_path = self.registry[msg.receiver_id].workspace / "inbox.jsonl"
        self._append_jsonl(inbox_path, msg.to_dict())

        # Inject into receiver's context (if agent is running)
        receiver = self.registry[msg.receiver_id]
        if receiver.status == "running":
            receiver.context_manager.inject_inbox_message(msg)

    def _check_communication_policy(self, msg: AgentMessage) -> None:
        """Enforce communication rules."""

        sender = self.registry.get(msg.sender_id)
        receiver = self.registry.get(msg.receiver_id)

        # Decomp agents can message: their parent, their children, siblings
        if sender and sender.agent_type == "decomposition":
            allowed = (
                self.dag.is_parent(msg.receiver_id, msg.sender_id) or
                self.dag.is_child(msg.receiver_id, msg.sender_id) or
                self.dag.is_sibling(msg.receiver_id, msg.sender_id)
            )
            if not allowed and msg.msg_type != MessageType.STATUS_UPDATE:
                raise CommunicationPolicyError(
                    f"{msg.sender_id} cannot message {msg.receiver_id}: not parent/child/sibling"
                )

        # Verifier agents can ONLY message orchestrator (return results)
        if sender and sender.agent_type == "verifier":
            if msg.receiver_id != "orchestrator":
                raise CommunicationPolicyError(
                    "Verifier agents can only communicate with orchestrator"
                )
```

### 4.5 Communication Policy Matrix

| Sender → Receiver | Allowed? | Message Types |
|-------------------|----------|---------------|
| Orchestrator → Any agent | Yes      | TASK_ASSIGNMENT, CHILD_RESULT, BUDGET_UPDATE, KILL |
| Decomp → Orchestrator | Yes      | REQUEST_CHILDREN, REQUEST_AGENT, TASK_COMPLETE, ESCALATE, STATUS_UPDATE |
| Decomp → Parent Decomp | Yes (via orchestrator) | ESCALATE, EVIDENCE, CONSTRAINT_UPDATE |
| Decomp → Child Decomp | Yes (via orchestrator) | BUDGET_UPDATE, QUERY |
| Decomp → Sibling Decomp | Yes (via orchestrator) | COUPLING_ALERT, QUERY, QUERY_RESPONSE |
| CAD → Orchestrator | Yes      | TASK_COMPLETE, ESCALATE |
| Verifier → Orchestrator | Yes      | TASK_COMPLETE (evidence + verdict) |
| CAD → CAD         | No       | N/A           |
| Verifier → Any non-orchestrator | No       | N/A           |
| Any → Random agent | No       | N/A           |


## 5. DAG Execution

The orchestrator executes the DAG in topological waves, paralleling independent branches while respecting coupling constraints. Kimi's PARL training (Fig 8) demonstrates that RL can teach the orchestrator to balance accuracy and parallelism simultaneously, while Fig 9 shows the resulting wall-clock speedup of 3-4.5x as task complexity grows.

### 5.1 DAG Structure

```python
@dataclass
class DAGNode:
    """One node in the execution DAG."""

    id: str
    agent_type: str              # "decomposition", "cad", "verifier", etc.
    goal: str                    # what this node must achieve
    parent_id: str | None        # parent DAG node
    children: list[str]          # child DAG nodes
    dependencies: list[str]      # must complete before this starts
    coupling_groups: list[str]   # shared coupling hubs
    status: Literal["pending", "ready", "running", "done", "failed", "blocked"]
    agent_id: str | None         # assigned agent (once created)
    result: dict | None          # output (once complete)
    sigma: float                 # confidence
    budget: BudgetAllocation


class DAG:
    """Global execution graph managed by orchestrator."""

    nodes: dict[str, DAGNode]
    edges: list[tuple[str, str]]  # dependency edges

    def get_ready_nodes(self) -> list[DAGNode]:
        """Nodes whose dependencies are all 'done'."""
        return [
            n for n in self.nodes.values()
            if n.status == "pending"
            and all(self.nodes[dep].status == "done" for dep in n.dependencies)
        ]

    def group_by_coupling(self, nodes: list[DAGNode]) -> list[list[DAGNode]]:
        """Group ready nodes for parallel execution.

        Nodes with no shared coupling groups → parallel.
        Nodes sharing a coupling group → same group (may need sequencing).

        Kimi's PARL training (Fig 8) shows that naive parallelism causes
        two failure modes: serial collapse (not parallelizing enough) and
        spurious parallelism (creating useless parallel tasks). The coupling
        group mechanism prevents spurious parallelism by only parallelizing
        truly independent branches.
        """
        independent = []
        coupled_groups: dict[str, list[DAGNode]] = {}

        for node in nodes:
            if not node.coupling_groups:
                independent.append([node])
            else:
                for group in node.coupling_groups:
                    coupled_groups.setdefault(group, []).append(node)

        return independent + list(coupled_groups.values())

    def critical_path_length(self) -> int:
        """Kimi metric: orchestration overhead + slowest node per wave."""
        # Topological sort → waves → sum of max(duration) per wave
        ...
```

>  ![Kimi PARL training — accuracy and parallelism increasing during RL](https://statics.moonshot.cn/blogs/k2-5/20260126-225846.png) *Fig 8. Kimi PARL training: reward shaping simultaneously increases task accuracy AND average parallelism. The orchestrator learns to parallelize correctly — avoiding both serial collapse (not enough parallelism) and spurious parallelism (useless parallel tasks). Source:* [*Kimi K2.5*](https://www.kimi.com/blog/kimi-k2-5.html)

### 5.2 Wave Execution

The wave scheduler groups ready nodes by coupling and launches independent groups in parallel. As shown in Fig 9, this wave-based approach yields significant speedup over serial execution, particularly for complex multi-subsystem designs like our hierarchical decomposition trees.

>  ![Kimi execution time comparison — Agent Swarm reduces wall-clock time by 3x-4.5x vs single agent](https://statics.moonshot.cn/blogs/k2-5/complpexity_of_tasks.png) *Fig 9. Kimi critical path speedup: wave-based parallel execution reduces wall-clock time 3-4.5x as task complexity grows. Speedup is measured by critical path (orchestration overhead + slowest subagent per wave), not total steps. Source:* [*Kimi K2.5*](https://www.kimi.com/blog/kimi-k2-5.html)

```python
class WaveScheduler:
    """Execute DAG in topological waves with parallelism."""

    def __init__(self, orchestrator: Orchestrator, dag: DAG):
        self.orchestrator = orchestrator
        self.dag = dag
        self.wave_count = 0

    def run(self) -> None:
        """Execute all waves until DAG is complete."""

        while self.dag.has_pending():
            self.wave_count += 1
            ready = self.dag.get_ready_nodes()

            if not ready:
                # Deadlock detection
                blocked = [n for n in self.dag.nodes.values() if n.status == "blocked"]
                if blocked:
                    self._handle_deadlock(blocked)
                continue

            groups = self.dag.group_by_coupling(ready)

            # Launch all independent groups in parallel
            wave_futures = []
            for group in groups:
                if len(group) == 1 or self._all_independent(group):
                    # Parallel: create agents + assign tasks concurrently
                    for node in group:
                        future = self._launch_agent(node)
                        wave_futures.append((node, future))
                else:
                    # Coupled: run sequentially within group
                    # but group itself runs in parallel with other groups
                    future = self._launch_coupled_group(group)
                    wave_futures.extend([(n, future) for n in group])

            # Wait for wave completion
            for node, future in wave_futures:
                result = future.result()
                self.dag.complete(node.id, result)

                # Check for escalations
                if result.get("escalation"):
                    self._handle_escalation(node, result["escalation"])

            # Post-wave: global consistency check
            if self.wave_count % self.consistency_check_interval == 0:
                self._check_global_consistency()

    def _launch_agent(self, node: DAGNode) -> Future:
        """Create agent + assign task for a single DAG node."""
        ctx = self._build_context(node)
        agent_id = self.orchestrator.create_agent(node.agent_type, ctx)
        node.agent_id = agent_id
        node.status = "running"
        return self.orchestrator.assign_task_async(agent_id, node.goal)
```


## 6. Observability & Logging

### 6.1 Event Model

Every agent action produces an event. Events flow to a central bus for aggregation.

```python
@dataclass
class AgentEvent:
    """Single observable event from any agent."""

    timestamp: datetime
    agent_id: str
    agent_type: str
    dag_node_id: str
    event_type: str             # see below
    data: dict                  # event-specific payload
    parent_event_id: str | None # causal chain
    duration_ms: float | None   # for timed events


class EventType(str, Enum):
    # Lifecycle
    AGENT_CREATED = "agent_created"
    AGENT_STARTED = "agent_started"
    AGENT_COMPLETED = "agent_completed"
    AGENT_FAILED = "agent_failed"
    AGENT_KILLED = "agent_killed"

    # LLM
    LLM_CALL_START = "llm_call_start"
    LLM_CALL_COMPLETE = "llm_call_complete"
    LLM_CALL_ERROR = "llm_call_error"

    # Tool use
    TOOL_CALL_START = "tool_call_start"
    TOOL_CALL_COMPLETE = "tool_call_complete"
    TOOL_CALL_ERROR = "tool_call_error"

    # Domain
    GRSC_GENERATED = "grsc_generated"
    CONSTRAINT_SELECTED = "constraint_selected"
    VERIFICATION_RESULT = "verification_result"
    DECOMPOSITION_DECIDED = "decomposition_decided"
    EVIDENCE_PRODUCED = "evidence_produced"

    # Communication
    MESSAGE_SENT = "message_sent"
    MESSAGE_RECEIVED = "message_received"
    ESCALATION = "escalation"

    # Context
    CONTEXT_COMPRESSED = "context_compressed"
    TODO_UPDATED = "todo_updated"

    # Budget
    BUDGET_CHECK = "budget_check"
    BUDGET_EXCEEDED = "budget_exceeded"

    # DAG
    WAVE_STARTED = "wave_started"
    WAVE_COMPLETED = "wave_completed"
    DAG_NODE_STATUS_CHANGE = "dag_node_status_change"
```

### 6.2 Observability Bus

```python
class ObservabilityBus:
    """Central event bus. All agents emit here via their trace file."""

    def __init__(self, agent_runs_dir: Path):
        self.agent_runs = agent_runs_dir
        self.global_log = agent_runs_dir / "global_trace.jsonl"

    def emit(self, event: AgentEvent) -> None:
        """Write event to agent's trace + global trace."""
        # Per-agent trace
        agent_trace = self.agent_runs / event.agent_id / "trace.jsonl"
        self._append_jsonl(agent_trace, event.to_dict())

        # Global trace (for cross-agent analysis)
        self._append_jsonl(self.global_log, event.to_dict())

    # --- Query API ---

    def get_agent_trace(self, agent_id: str) -> list[AgentEvent]:
        """Full history of one agent."""
        ...

    def get_dag_timeline(self) -> list[AgentEvent]:
        """All events sorted by time — the complete execution story."""
        ...

    def get_critical_path(self) -> list[AgentEvent]:
        """Events on the critical path (orchestration + slowest per wave)."""
        ...

    def get_sigma_evolution(self) -> dict[str, list[tuple[datetime, float]]]:
        """Confidence over time per DAG node."""
        ...

    def get_message_flow(self) -> list[AgentMessage]:
        """All inter-agent messages in order."""
        ...

    def get_budget_usage(self) -> dict[str, BudgetUsage]:
        """Token/cost/time usage per agent."""
        ...

    def get_error_chain(self, agent_id: str) -> list[AgentEvent]:
        """Trace from error back through causal chain."""
        events = self.get_agent_trace(agent_id)
        errors = [e for e in events if e.event_type.endswith("_error")]
        # Follow parent_event_id chain to find root cause
        ...
```

### 6.3 Per-Agent Trace Format

Each agent's `trace.jsonl` is append-only, one JSON object per line:

```jsonl
{"ts":"2025-01-15T10:00:01Z","agent":"decomp_litho_002","type":"agent_started","data":{"goal":"Provide EUV lithography","budget":{"cost_usd":500}}}
{"ts":"2025-01-15T10:00:02Z","agent":"decomp_litho_002","type":"llm_call_start","data":{"model":"claude-sonnet-4-5-20250929","prompt_tokens":2400,"purpose":"grsc_generation"}}
{"ts":"2025-01-15T10:00:05Z","agent":"decomp_litho_002","type":"llm_call_complete","data":{"output_tokens":1800,"duration_ms":3200,"cost_usd":0.02}}
{"ts":"2025-01-15T10:00:05Z","agent":"decomp_litho_002","type":"grsc_generated","data":{"goals":7,"requirements":12,"specs":18,"contracts":5}}
{"ts":"2025-01-15T10:00:06Z","agent":"decomp_litho_002","type":"tool_call_start","data":{"tool":"verify_csp","constraints":["power_250w","wavelength_13.5nm"]}}
{"ts":"2025-01-15T10:00:06Z","agent":"decomp_litho_002","type":"verification_result","data":{"verdict":"SAT","sigma":0.91,"logic_constraints":8,"semantic_constraints":3}}
{"ts":"2025-01-15T10:00:07Z","agent":"decomp_litho_002","type":"decomposition_decided","data":{"children":["euv_source","illumination","reticle","projection","wafer_stage","alignment","dose"]}}
{"ts":"2025-01-15T10:00:07Z","agent":"decomp_litho_002","type":"message_sent","data":{"to":"orchestrator","type":"request_children","children_count":7}}
```

### 6.4 Dashboard Views

The observability data supports these views:

| View | Source | Purpose |
|------|--------|---------|
| **DAG View** | `dag.json` + node statuses | See which nodes are running/done/blocked |
| **Wave Timeline** | `waves.jsonl` | See execution waves, parallelism, critical path |
| **Agent Deep-Dive** | `{agent_id}/trace.jsonl` | Full history of one agent's decisions |
| **Message Flow** | All `inbox.jsonl` + `outbox.jsonl` | See inter-agent communication graph |
| **σ Dashboard** | `verification_result` events | Confidence evolution across the hierarchy |
| **Budget Burndown** | `budget_check` events | Cost/token usage over time |
| **Error Trace** | `*_error` events + causal chain | Debug failures across agent boundaries |


## 7. Orchestrator Implementation

The orchestrator (Fig 1) is the only component with a global view of the DAG. It creates agents, routes messages, and drives wave execution (Fig 8, Fig 9).

### 7.1 Core Loop

```python
class Orchestrator:
    """Meta-level manager. Creates agents, manages DAG, routes messages."""

    def __init__(self, config: OrchestratorConfig):
        self.dag = DAG()
        self.registry: dict[str, AgentInfo] = {}
        self.router = MessageRouter(self.registry, self.dag)
        self.scheduler = WaveScheduler(self, self.dag)
        self.bus = ObservabilityBus(config.agent_runs_dir)
        self.budget = GlobalBudget(config.total_budget)

    def run(self, intent: str) -> SystemResult:
        """Main entry point. Decomposes intent into DAG and executes."""

        # Create root Decomposition Agent
        root_ctx = AgentContext(
            agent_id="decomp_root_001",
            agent_type="decomposition",
            dag_node_id="root",
            goal=intent,
            parent_contracts=[],  # root has no parent
            resolved_constraints=set(),
            budget=self.budget.allocate_root(),
            workspace=self.config.agent_runs_dir / "decomp_root_001",
        )

        root_node = DAGNode(
            id="root",
            agent_type="decomposition",
            goal=intent,
            parent_id=None,
            children=[],
            dependencies=[],
            coupling_groups=[],
            status="ready",
            agent_id=None,
            result=None,
            sigma=0.0,
            budget=root_ctx.budget,
        )
        self.dag.add_node(root_node)

        # Launch root agent
        self.create_agent(root_ctx)
        self.assign_task(root_ctx.agent_id, intent)

        # Run wave scheduler
        self.scheduler.run()

        # Collect final results
        return self._build_system_result()

    def create_agent(self, ctx: AgentContext) -> str:
        """Spawn a new agent with isolated context."""
        # Create workspace directory
        ctx.workspace.mkdir(parents=True, exist_ok=True)
        (ctx.workspace / "inbox.jsonl").touch()
        (ctx.workspace / "outbox.jsonl").touch()
        (ctx.workspace / "trace.jsonl").touch()
        (ctx.workspace / "context.md").write_text(f"# {ctx.agent_id}\n\nGoal: {ctx.goal}\n")
        (ctx.workspace / "todo.md").write_text(f"# TODO\n\n- [ ] {ctx.goal}\n")

        # Register agent
        self.registry[ctx.agent_id] = AgentInfo(
            agent_id=ctx.agent_id,
            agent_type=ctx.agent_type,
            status="created",
            context=ctx,
            context_manager=ContextManager(ctx.agent_id, ctx.workspace, ctx.max_tokens),
            workspace=ctx.workspace,
        )

        self.bus.emit(AgentEvent(
            timestamp=datetime.now(),
            agent_id=ctx.agent_id,
            agent_type=ctx.agent_type,
            dag_node_id=ctx.dag_node_id,
            event_type=EventType.AGENT_CREATED,
            data={"goal": ctx.goal, "budget": ctx.budget.__dict__},
            parent_event_id=None,
            duration_ms=None,
        ))

        return ctx.agent_id

    def handle_request_children(self, msg: AgentMessage) -> None:
        """Handle a Decomposition Agent requesting child agents."""
        parent_node = self.dag.get_node_by_agent(msg.sender_id)

        for child_spec in msg.content["children"]:
            child_node_id = generate_id("dag")
            child_agent_id = generate_id(child_spec["agent_type"])

            # Add to DAG
            child_dag_node = DAGNode(
                id=child_node_id,
                agent_type=child_spec["agent_type"],
                goal=child_spec["goal"],
                parent_id=parent_node.id,
                children=[],
                dependencies=[],  # independent of siblings by default
                coupling_groups=child_spec.get("coupling_groups", []),
                status="pending",
                agent_id=None,
                result=None,
                sigma=0.0,
                budget=BudgetAllocation(**child_spec["budget"]),
            )
            self.dag.add_node(child_dag_node)
            parent_node.children.append(child_node_id)

        # Scheduler will pick up new ready nodes in next wave
```


## 8. Decomposition Agent Implementation

### 8.1 Core Logic

```python
class DecompositionAgentLogic:
    """Implements the formalization's SOLVE for a single node.

    This runs inside the agent's isolated context.
    The agent interacts with the orchestrator via messages.
    """

    def __init__(self, ctx: AgentContext):
        self.ctx = ctx
        self.goal = ctx.goal
        self.constraints = ctx.resolved_constraints
        self.parent_contracts = ctx.parent_contracts
        self.budget = ctx.budget
        self.workspace = ctx.workspace

    def run(self) -> DecompResult:
        """Main agent loop."""

        # Update todo (Manus trick: keep plan in recent attention, Fig 4)
        self.update_todo([
            "Explore spec alternatives",
            "Run GRSC",
            "Verify constraints (CSP)",
            "Decide: leaf or decompose",
            "If decompose: request children + gather results",
            "Check cross-child consistency",
            "Return evidence to parent",
        ])

        # §2.2: Explore specification alternatives
        spec_options = self.explore_specs()
        committed_specs = self.select_best(spec_options)
        self.mark_todo_done("Explore spec alternatives")

        # §3-5: GRSC + verify
        grsc = self.run_grsc(committed_specs)
        verdict, sigma = self.verify_csp(grsc)
        self.mark_todo_done("Run GRSC")

        # §7: Feedback loop (errors stay visible per Fig 3)
        attempts = 0
        while verdict != "SAT" or sigma < self.budget.sigma_threshold:
            attempts += 1
            if attempts >= self.budget.max_retries:
                return self.escalate("max_retries", grsc, verdict, sigma)

            feedback = self.get_constraint_trace(grsc)
            grsc = self.run_grsc(committed_specs, feedback=feedback)
            verdict, sigma = self.verify_csp(grsc)

        self.mark_todo_done("Verify constraints (CSP)")

        # §9: Leaf check
        if self.is_leaf():
            self.mark_todo_done("Decide: leaf or decompose")
            return self.handle_leaf(grsc, sigma)

        # §8: Decompose
        children = self.plan_children(grsc)
        independence = self.assess_coupling(children)
        self.request_children(children, independence)
        self.mark_todo_done("Decide: leaf or decompose")

        # Wait for children (orchestrator sends CHILD_RESULT messages)
        child_results = self.gather_results()
        self.mark_todo_done("If decompose: request children + gather results")

        # §11: Cross-child consistency
        consistency = self.check_consistency(child_results)
        if not consistency.ok:
            # May need to re-plan specific children
            child_results = self.handle_inconsistency(consistency, child_results)
        self.mark_todo_done("Check cross-child consistency")

        # Merge evidence upward
        merged = self.merge_evidence(child_results)
        self.mark_todo_done("Return evidence to parent")

        return DecompResult(
            evidence=merged.evidence,
            unknowns=merged.unknowns,
            sigma=merged.sigma,
            artifacts=merged.artifacts,
            budget_actual=self.budget_used(),
        )

    def handle_leaf(self, grsc, sigma) -> DecompResult:
        """At leaf: spawn CAD or Buy agent, then Verifier."""

        if self.is_purchasable():
            result = self.request_agent("buy", {
                "specs": grsc.specs,
                "contracts": grsc.contracts,
            })
        else:
            # Spawn CAD agent
            cad_result = self.request_agent("cad", {
                "specs": grsc.specs,
                "contracts": grsc.contracts,
                "ops_sequence": grsc.ops_sequence,
            })

            # Spawn Verifier (gets ONLY artifact + spec, not CAD context)
            verify_result = self.request_agent("verifier", {
                "artifact_path": cad_result["artifact_path"],
                "contracts": grsc.contracts,
                "specs": grsc.specs,
            })

            if not verify_result["passed"]:
                # Iterate: send feedback to CAD agent
                for attempt in range(self.budget.max_retries):
                    cad_result = self.request_agent("cad", {
                        "specs": grsc.specs,
                        "contracts": grsc.contracts,
                        "feedback": verify_result["feedback"],
                    })
                    verify_result = self.request_agent("verifier", {
                        "artifact_path": cad_result["artifact_path"],
                        "contracts": grsc.contracts,
                        "specs": grsc.specs,
                    })
                    if verify_result["passed"]:
                        break

            result = {**cad_result, **verify_result}

        return DecompResult(
            evidence=result.get("evidence", []),
            unknowns=result.get("unknowns", []),
            sigma=result.get("sigma", 0.0),
            artifacts=result.get("artifacts", []),
        )
```


## 9. Mapping to Existing Codebase

### Current → New

| Current (linear pipeline) | New (DAG multi-agent) |
|---------------------------|-----------------------|
| `cli.py` sequential steps | Orchestrator wave scheduler (Fig 1, Fig 9) |
| `agents/grs_refinement.py` | Decomposition Agent's `run_grsc()` tool |
| `agents/contract_extraction.py` | Part of Decomposition Agent's GRSC phase |
| `verification/pipeline.py` (V0-V4) | Verifier Agent tools  |
| `agents/ops_generator.py` | CAD Agent             |
| `MechanicalVerificationAgent` | Verifier Agent tools (DFM subset) |
| `OptimizationAgent`       | CAD Agent feedback loop |
| `cad/intent_decomposition/observability/trace_collector.py` | Per-agent `trace.jsonl` + ObservabilityBus |
| `HypergraphEngine` (single shared state) | Per-agent subgraph views + orchestrator's global DAG |
| `HypergraphStore` (disk persistence) | Agent workspace directories (Fig 2) + global DAG state |

### Migration Path



1. **Phase 1**: Extract `AgentContext` + `ContextManager` + `MessageRouter` as core primitives
2. **Phase 2**: Wrap existing agents (`grs_refinement`, `contract_extraction`) as Decomposition Agent tools
3. **Phase 3**: Implement Orchestrator + DAG + Wave Scheduler
4. **Phase 4**: Implement inter-agent communication (inbox/outbox)
5. **Phase 5**: Add observability bus, replace `TraceCollector`
6. **Phase 6**: Enable parallelism (start with independent leaf parts)
7. **Phase 7**: Enable recursive decomposition (multi-level hierarchy)


## 10. File-System Context (Reference)

Detail layouts referenced from §3. Core concepts (Fig 2, isolation, file tools) are in §3.1.

### 10.1 Detailed Workspace Structure

A single decomposition agent's full workspace (beyond the compact view in §3.1):

```
agent_runs/decomp_litho_002/
├── context.md              # Running summary of conversation so far
├── todo.md                 # Current plan — always in recent attention
├── grsc_output.json        # Structured GRSC output
│
├── specs/
│   ├── euv_source.json     # Committed spec for EUV source
│   ├── projection.json     # Committed spec for projection optics
│   └── alternatives.json   # Rejected alternatives + rationale
│
├── constraints/
│   ├── active.json         # C_v* — current active constraints
│   ├── resolved.json       # Satisfied by ancestors
│   └── violated.json       # Failed verification
│
├── evidence/
│   ├── child_euv.json      # Evidence from child agents
│   └── merged.json         # Fused evidence for parent
│
├── decisions/
│   └── spec_selection.md   # Decision rationale
│
├── inbox.jsonl             # Messages received
├── outbox.jsonl            # Messages sent
└── trace.jsonl             # Full event log
```

### 10.2 Read-Only Shared Resources

```
shared/
├── knowledge_base/
│   ├── materials/
│   │   ├── aluminum_7075.json    # Material properties
│   │   ├── zerodur.json
│   │   └── ...
│   ├── processes/
│   │   ├── cnc_3axis.json        # Manufacturing process capabilities
│   │   ├── wire_edm.json
│   │   └── ...
│   └── standards/
│       ├── iso_2768.json         # Tolerance standards
│       └── ...
│
├── supplier_db/
│   └── components.json           # Purchasable component catalog
│
├── templates/
│   ├── grsc_prompt.md            # GRSC generation prompt template
│   ├── verification_prompt.md
│   └── ...
│
└── calibration/
    ├── tool_calibrations.json    # VLM/DFM calibration data
    └── prior_distributions.json  # Default priors for UQ
```

### 10.3 Context Compression → File Externalization Cycle

Referenced from §3.3. Full diagram of the compression mechanism:

```
┌─────────────────────────────────────────────────────┐
│ Agent's Active Context Window                        │
│                                                      │
│ [System prompt] [Parent contracts] [Tool defs]       │
│ [Turn 1] [Turn 2] ... [Turn 40] [Turn 41]          │
│                                                      │
│ ⚠️ Approaching 80% capacity                          │
└─────────────────────┬───────────────────────────────┘
                      │
            ┌─────────▼─────────┐
            │ COMPRESS           │
            │                    │
            │ 1. Summarize turns │
            │    1-30 into       │
            │    context.md      │
            │                    │
            │ 2. Write structured│
            │    data to files:  │
            │    specs/*.json    │
            │    decisions/*.md  │
            │    evidence/*.json │
            │                    │
            │ 3. Keep turns 31+  │
            │    in active window│
            │                    │
            │ 4. Re-inject:      │
            │    - context.md    │
            │      summary       │
            │    - todo.md       │
            └─────────┬─────────┘
                      │
┌─────────────────────▼───────────────────────────────┐
│ Agent's Active Context Window (after compression)    │
│                                                      │
│ [System prompt] [Parent contracts] [Tool defs]       │
│ [Summary from context.md] [todo.md]                  │
│ [Turn 31] ... [Turn 41]                             │
│                                                      │
│ ✅ Back to ~50% capacity                             │
│                                                      │
│ Agent can read_file("specs/euv_source.json")        │
│ anytime it needs details that were compressed away   │
└─────────────────────────────────────────────────────┘
```

## 11. Open Questions



 1. **Agent lifecycle**: keep agents alive across waves (reuse context, cheaper) vs destroy+recreate (cleaner isolation)?
 2. **Coupling detection**: how does the orchestrator know which branches share coupling hubs before GRSC runs? Decomp agent must annotate in REQUEST_CHILDREN
 3. **Model routing**: orchestrator=Opus, decomp=Sonnet, CAD=Sonnet, verifier=Haiku? Or same model everywhere?
 4. **Max parallelism**: Kimi does 100 agents; practical limit given API rate limits + cost?
 5. **State persistence**: DAG + agent workspaces must survive restarts. Is file-system sufficient or need a DB?
 6. **Context window per agent**: different agents need different window sizes. CAD agent needs large context for code generation; verifier needs small context
 7. **Deadlock handling**: what if two coupled siblings each wait for the other's coupling data?
 8. **Re-planning cost**: when bottom-up propagation triggers re-plan, does the parent re-plan the whole subtree or just the affected branch?
 9. **Skill pack spec**: what's the minimal viable skill? Just `prompt.md`, or also `constraints.json`, `examples.json`, calibration data?
10. **Handoffs**: should agents ever transfer state directly peer-to-peer, or must everything route through the orchestrator?
11. **Stateful agent reuse**: keep warm agents for repeated similar tasks (cheaper, preserves learned context) vs fresh isolation (cleaner, no cross-contamination)?
12. **Mid-execution user interaction**: allow users to steer/approve at intermediate decomposition nodes, or only at root entry and final output?

## 12. References

* [Kimi K2.5: Agent Swarm Architecture](https://www.kimi.com/blog/kimi-k2-5.html)
* [Manus: Context Engineering for AI Agents](https://manus.im/blog/Context-Engineering-for-AI-Agents-Lessons-from-Building-Manus)
* [Anthropic: Building Multi-Agent Systems](https://claude.com/blog/building-multi-agent-systems-when-and-how-to-use-them)
* [LangChain: Choosing the Right Multi-Agent Architecture](https://blog.langchain.com/choosing-the-right-multi-agent-architecture/)
* [Agentic Reasoning for Large Language Models](https://arxiv.org/pdf/2601.12538)