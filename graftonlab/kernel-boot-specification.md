---
title: "Kernel Boot Specification"
description: "Complete specification for booting the hierarchical constraint-driven planning kernel"
---

What must exist at boot for the system to follow the [System Flow §7](https://graftonsciences.getoutline.com/doc/minimum-recursive-kernel-Etdc4x1HAi#h-7-system-flow) and recursively get better. This document traces every step, maps the data that passes between steps, specifies the minimum infrastructure, and shows how the kernel improves as humans resolve edge cases and the system accumulates experience.

**Refs**: [Minimum Recursive Kernel](https://graftonsciences.getoutline.com/doc/minimum-recursive-kernel-Etdc4x1HAi), [Hierarchical Constraint-Driven Planning: Formalization](https://graftonsciences.getoutline.com/doc/hierarchical-constraint-driven-planning-formalization-8c6MpeON72)

## 0. MVP Scope + Conventions (Decision-Complete)

This spec is written to be implementable as an MVP that follows the Formalization's SOLVE loop and the Minimum Recursive Kernel's TRY→BUILD→HIRE growth mechanism.

**MVP scope (locked):**

* Domain: software-only goals (code, config, docs). Physical/hybrid resolver types may be registered but are out of scope for MVP execution.
* Budget enforcement: cost (float, USD-like) + wall time (seconds) enforced per node for *every* resolver call (including Claude Code operations).
* Confidence: per-constraint σ_i (logic=1.0, semantic∈\[0,1\]); node σ_v = min(σ_i); path σ_path(v) = ∏ σ_u along root→v; system σ_system = min over leaf σ_path.
* Thresholds (config defaults): τ_local=0.70, τ_global=0.50. Both are stored on each node at creation time for auditability.
* IDs: all primary IDs are TEXT (UUID/ULID). Knowledge Graph IDs are namespaced to avoid collisions: `resolver:<id>`, `node:<id>`, `attempt:<id>`, `artifact:<id>`, `constraint:<id>`, `capability:<tag>`, `failure:<slug>`, `goal_pattern:<id>`, `actor:human`.

> MVP testing should include non-software goal inputs (e.g., hardware design, logistics) to validate that constraint identification, decomposition, and resolver selection reason correctly across domains — even though resolution will bottom out at HIRE for non-software capabilities.

**Tool-first model:**

* A **tool** is anything that can be used to advance a goal: resolvers, verifiers, decomposers, schedulers, web/AWS adapters, test runners, retrieval systems, lab rigs, and (temporarily) humans.
* In this spec, **resolver == tool** (a resolver is a callable tool with metadata + cost profile). The kernel treats every operation as "select a tool, run it, verify, record outcome".
* Tools are **composable**: a higher-level tool can internally call other tools (toolchains/pipelines). Composition may be explicit (stored as a tool plan artifact) or implicit (a composite resolver implementation).
* Tools are **buildable artifacts**: the output of BUILD/HIRE is not a one-off answer; it is a *new tool* (or verifier/rule/checker) persisted in the library for reuse.
* Tools can build tools: the system may build a tool that enables building another tool (meta-tools), and both persist.
* Some tools are **kernel-axiom tools** (dispatcher/state machine/budget enforcer/stores). They must exist at boot and are deterministic (logic-typed, σ=1.0).

**"Claude Code" in this spec:**

* Boot-time default: headless Claude API calls (Opus) wrapped by the kernel's budget enforcer with structured I/O (JSON envelopes). The `claude-code-0` resolver adapter formats prompts with KG context, calls the API, parses responses, and returns artifacts.
* Interactive Claude Code (CLI with human-in-the-loop) is used only for Step 1 (goal extraction) and Step 11 (HIRE).
* Claude Agent SDK is a growth-path option for building composite resolvers.
* CLI default is headless (auto mode). An `--interactive` flag enables human-in-the-loop at any step for debugging and exploration.

**Human role + scalability requirement:**

* Humans (the operators) provide the root goal + irreducible core constraints + optional suggestions/ideas.
* Human suggestions must be evaluated **on merit without privilege**: ideas are treated as candidate artifacts and scored/filtered against constraints and historical outcomes; author identity must not directly affect selection.
* **Infinite-scalability requirement**: after sufficient bootstrapping in a domain distribution, expected HIRE calls should converge to 0. To enforce compounding:
  * Any HIRE outcome must be toolized (registered tool/verifier/rule), not a one-off patch.
  * Edge-case handling must itself become a toolchain (failure detection → diagnosis → repair/tool-build → regression verification).

**Canonical envelopes**

Everything persisted or passed between steps is an envelope with stable identity and lineage.

```json
{
  "id": "…",
  "kind": "goal|constraint|artifact|tool_plan|verification|attempt|capability_gap",
  "format": "text|json|code|binary|…",
  "content": "inline or reference",
  "metadata": {},
  "lineage": {
    "node_id": "…",
    "resolver_id": "…",
    "constraint_ids": ["…"],
    "input_artifact_ids": ["…"]
  }
}
```

**Constraint (minimum shape)**

```json
{
  "id": "…",
  "type": "logic|semantic",
  "domain": "software|cost|schedule|…",
  "title": "short human label",
  "payload": {},
  "provenance": {
    "source": "human|resolver|derived",
    "resolver_id": "…",
    "derived_from_constraint_ids": ["…"]
  }
}
```

`payload` is intentionally open-ended. MVP requires only:

* Logic constraints must be verifiable deterministically (σ=1.0) by code, a test runner, or a simple checker.
* Semantic constraints must be verifiable by an evaluator that emits σ∈\[0,1\] plus an interpretable rubric trace.

**Verification result (minimum shape)**

```json
{
  "verdict": "SAT|UNSAT",
  "sigma_v": 0.0,
  "checks": [
    {
      "constraint_id": "…",
      "passed": true,
      "sigma_i": 1.0,
      "detail": {},
      "repair_suggestion": {}
    }
  ],
  "trace_summary": "one-paragraph summary for humans"
}
```

`sigma_v = min(checks[].sigma_i)` (formalization §5). `sigma_path` multiplies node sigmas (formalization §6).

**Tool plan artifact (minimum shape)**

A `tool_plan` is an artifact with `format="json"` that specifies a 1..N step toolchain. MVP only needs sequential plans.

```json
{
  "steps": [
    {
      "resolver_id": "codex-tools-0",
      "capability": "test_execution",
      "inputs": {"input_artifact_ids": ["artifact:…"]},
      "params": {}
    }
  ],
  "policy": {"parallel": false}
}
```

## 1. System Flow — Step-by-Step Data Trace

Every step of §7's SOLVE loop traced with the exact data that passes between steps and the infrastructure required to execute it.

### Step 1: Goal Arrives

|     |     |
|-----|-----|
| **What happens** | Human provides raw intent + irreducible core constraints. System creates root node. |
| **Input** | Raw intent (any modality), human-seeded core constraints, optional human suggestions/ideas (treated as unprivileged candidate artifacts) |
| **Output** | \`Node \{ id, parent_id:null, goal_description |
| **Infra required** | Node Store (create node), Constraint typing (classify each constraint as logic\|semantic) |
| **At boot** | Claude Code runs the iterative extraction dialogue (formalization §14 M1): proposes candidate constraints, human confirms/rejects, checks for irreducible core |
| **Growth path** | Domain-specific goal synthesizers that know which constraints to probe (e.g., hardware synthesizer always asks thermal/structural/cost/timeline) |

### Step 2: Identify Constraints

|     |     |
|-----|-----|
| **What happens** | Given goal $G_v$, determine which constraints from the universe apply. This is the relevance function $\rho(c, G_v, d)$ from formalization §3. |
| **Input** | \`Node.goal_description |
| **Output** | `constraints_eligible_ids` — list of applicable constraint IDs (each constraint typed logic\|semantic in the constraint store) |
| **Infra required** | Node Store (read node + ancestors), Knowledge Graph (query: "what constraints have been relevant for similar goals?") |
| **Context injected** | `KG.query("constraints applied to goals similar to {goal_summary}")` → structured block in prompt |
| **At boot** | Claude Code reads goal + parent context + KG results, proposes relevant constraints |
| **Growth path** | Datalog forward-chaining implements $\rho$ with completeness guarantees ($\sigma = 1.0$). Per Symbolic Reasoning doc §4.2. |

### Step 3: Select Constraints (Parsimony)

|     |     |
|-----|-----|
| **What happens** | From $C_v^{eligible}$, pick minimum set $C_v^*$ such that removing any one makes the problem underdetermined. Parsimony test: adding $c$ to $S$ only justified if $\text{Sol}(G, S \cup \{c\}) \subset \text{Sol}(G, S)$. |
| **Input** | `constraints_eligible_ids` (from step 2), \`Node.goal_description |
| **Output** | `constraints_selected_ids` (the IDs of $C_v^*$) |
| **Infra required** | Node Store (persist selected IDs), Constraint Store (fetch full constraint objects by ID for prompt/verification) |
| **At boot** | Claude Code selects. Imperfect but workable — may over- or under-select. |
| **Growth path** | Z3/SAT-based parsimony checker that can formally verify "removing $c_i$ makes problem underdetermined" |

### Step 4: Select Resolver

|     |     |
|-----|-----|
| **What happens** | Given $C_v^*$ and $G_v$, pick (or compose) the best **tool** from the library. At boot this is "select one resolver"; in growth it becomes routing + composition over a toolchain. |
| **Input** | `constraints_selected_ids`, \`Node.goal_description |
| **Output** | `execution_plan_artifact_id` (a `tool_plan` artifact: 1..N tool steps) + `{estimated_cost, estimated_time_seconds}`. MVP: plan length = 1 (single resolver step). |
| **Infra required** | Registry (capability-match query), Knowledge Graph (query: "what resolvers have been tried for similar goals? what succeeded/failed and why?"), Budget Tracker (remaining budget for this node) |
| **Context injected** | `KG.query("resolution_attempts WHERE goal_similar_to {goal} ORDER BY success DESC")` → structured block: `[Previous attempts: resolver_A failed (reason: X), resolver_B succeeded (cost: $Y, sigma_v: Z)]` |
| **At boot** | Registry.query(required_capabilities) returns candidates. Selection is effectively standard tool routing: capability match + cost/latency metadata. Quality vectors start at defaults (0.5) and are uninformative until KG accumulates outcome data. Claude Code picks from candidates using these coarse signals — usually `claude-code-0`, sometimes `codex-tools-0`. |
| **Growth path** | Learned router/composer trained on (tool_plan, goal, outcome) history. Capability tags are treated as coarse hints; empirical success/failure dominates selection over time. |

### Step 5: Resolve

|     |     |
|-----|-----|
| **What happens** | Call the selected resolver with constraints and input. This is the actual work. |
| **Input** | `execution_plan_artifact_id`, `constraints_selected_ids`, `Node.input_artifact_ids` (inputs; may include parent's output) |
| **Output** | `output_artifact_id` (the artifact produced by the resolver) |
| **Side effects** | Budget Tracker: `pre_check(estimated_cost, estimated_time)` before call, `record(actual_cost, actual_time)` after. Attempt Store: create `resolution_attempt` record with `{plan_artifact_id, output_artifact_id}`. Tiered Log: record run metadata; tier-3 raw I/O is optional/redacted by default. |
| **Infra required** | Budget Tracker (pre-check gate + record), Resolution Logger, Tiered Logger, the resolver execution itself |
| **At boot** | Execute the tool plan (usually a single step: Claude Code API call or Codex tool execution). Budget enforcer wraps the call — if `pre_check` returns DENIED, skip to HIRE (step 11). |
| **Growth path** | Specialized resolvers replace Claude Code per domain. Each is registered with capability tags and cost profiles. |

**Budget gate detail**: Before every resolver call:

```
remaining_cost = budget.remaining_cost(node_id)
remaining_time = budget.remaining_time(node_id)
(estimated_cost, estimated_time) = resolver.estimate_cost_time(...)
if estimated_cost > remaining_cost or estimated_time > remaining_time:
    return DENIED → escalate to HIRE
```

### Step 6: Verify

|     |     |
|-----|-----|
| **What happens** | Check output against constraints. Produce per-constraint verdict, confidence, and violation trace. |
| **Input** | `Node.input_artifact_ids`, `output_artifact_id`, `constraints_selected_ids` |
| **Output** | `Verification { verdict, sigma_v, checks[] }` (checks reference `constraint_id`, not embedded objects) |
| **Infra required** | Attempt Store (attach verification to the attempt), Node Store (update node status + `constraints_resolved_ids` + `sigma_v` + `sigma_path`), Knowledge Graph (record attempt outcome), Budget Tracker (verification itself may cost tokens/time) |
| **At boot** | Claude Code evaluates each constraint against output. Logic-typed constraints with code outputs: run tests via Codex ($\sigma = 1.0$ if test passes). Semantic constraints: Claude Code judgment ($\sigma < 1.0$). |
| **Growth path** | Datalog rule sets for logic verification ($\sigma = 1.0$). Z3 for constraint optimization. Test suites generated as subgoals. Per Symbolic Reasoning doc. |

> **Tuning note:** tau_global may need to be depth-dependent to avoid gating loops at higher levels where σ is inherently low (semantic verification of decomposition plans). Consider tau_effective = tau_global \* (1 - decay^depth) so gating is loose at root and tightens toward leaves. Flagged for empirical tuning.

### Step 7: SAT + Leaf → Register Artifact

|     |     |
|-----|-----|
| **What happens** | Output satisfies constraints and node is a leaf (atomic, off-the-shelf, contracted, or constraint-exhausted). Register the artifact. |
| **Input** | `output_artifact_id`, `Node` (verified SAT, leaf=true) |
| **Output** | `Artifact { id, type, format, uri, sha256, lineage: { resolver_id, constraint_ids, node_id, input_artifact_ids }, sigma_v }` |
| **Infra required** | Node Store (mark resolved), Knowledge Graph (register artifact node, add `produced_by→resolver`, `satisfies→constraints` edges), Tier 1 Log |
| **At boot** | Deterministic: store artifact, update node status, write edges. No LLM needed. |

### Step 8: SAT + Not Leaf → Decompose

|     |     |
|-----|-----|
| **What happens** | Output satisfies constraints but goal needs further decomposition. Break into subgoals with dependency ordering. |
| **Input** | \`Node.goal_description |
| **Output** | Child nodes `{G_v1, ..., G_vk}` with: dependency relationships (DAG), inherited constraints (parent's resolved constraints), allocated budgets |
| **Infra required** | Node Store (create children, link to parent), Dependency Resolver (compute topological order), Budget Tracker (allocate parent's remaining budget to children), Knowledge Graph (query decomposition history + record new decomposition) |
| **Context injected** | `KG.query("decompositions of goals similar to {goal}")` → `[Similar goal X was decomposed into: A→B→C, with B depending on A's output]` |
| **At boot** | Claude Code decomposes and estimates dependencies. Budget allocation: Claude Code estimates relative child complexity → proportional allocation with 20% reserve for retries/overhead. Dependency Resolver computes topological sort → children enter queue in dependency order. |
| **Growth path** | Domain-specific decomposers (mechanical assembly hierarchies, software architecture decomposers). Dependency estimation from historical data. Budget allocation from historical cost-per-goal-type. |

**Budget allocation protocol**:

```
parent_remaining = budget.remaining(node_id)
reserve = parent_remaining * 0.20  # for retries, overhead, verification
allocatable = parent_remaining - reserve
child_estimates = claude_code.estimate_relative_complexity(children)
for child, weight in zip(children, normalize(child_estimates)):
    budget.allocate(child.id, allocatable * weight)
```

### Step 9: UNSAT → Feedback / Retry

|     |     |
|-----|-----|
| **What happens** | Output failed verification. Generate feedback from violation trace. Retry with same resolver up to $T_{max}$. |
| **Input** | `output_artifact_id` (failed output), `constraints_selected_ids`, `verification.checks[]` (per-constraint failures), attempt history from Attempt Store |
| **Output** | New `output_artifact_id` (retry output) → back to VERIFY (step 6), OR escalation signal (exhausted) |
| **Infra required** | Attempt Store (read prior attempts, persist feedback + new attempt), Node Store (update `attempt_count`), Knowledge Graph (record failure_mode + attempt outcome), Budget Tracker (retry costs cost/time) |
| **Context injected** | Full attempt history: `[Attempt 1: tried X, failed because constraint C3 violated (detail). Attempt 2: tried Y, failed because ...]` |
| **At boot** | Claude Code generates feedback from violation trace + attempt history. The violation trace is key — not just "UNSAT" but "constraint C3 failed: expected latency &lt; 200ms, got 450ms, likely caused by synchronous DB call in handler." |
| **Termination** | `attempt_count >= T_max` → escalate to BUILD (step 10) |
| **Growth path** | Z3 abductive repair (Symbolic Reasoning doc §4.3): computes minimum-cost parameter changes to achieve SAT. ASP enumeration of valid alternatives. Prescriptive repairs, not just diagnostics.Stagnation detection: if sigma improvement across last N retries falls below epsilon (e.g., delta_sigma &lt; 0.05 over 2 consecutive attempts), short-circuit to selecting_resolver or escalate to BUILD early rather than exhausting T_max. Data available in resolution_attempts. Additionally, consider allowing child resolvers to emit a RESCOPE signal when UNSAT appears to stem from parent decomposition rather than resolver inadequacy. |

### Step 10: Exhausted → BUILD

|     |     |
|-----|-----|
| **What happens** | Current resolver(s) can't solve this within $T_{max}$ retries. Identify the capability gap. Check if a matching resolver already exists. If not, construct one as a subgoal. |
| **Input** | Capability gap (what's needed), Knowledge Graph (existing resolvers, similar past goals), Budget Tracker (can we afford to build?) |
| **Output** | Either: existing resolver found → retry (back to step 4), OR `G_build` created → recursive SOLVE, OR budget exceeded → HIRE (step 11) |
| **Infra required** | Knowledge Graph (**lookup-before-build**: mandatory query for matching resolvers), Registry, Budget Tracker, Node Store (create build-subgoal if needed) |
| **Lookup-before-build query** | `KG.query("resolvers WITH capabilities MATCHING {gap} AND verdict=SAT")` — if match found, use it directly. Also: `KG.query("build_attempts FOR capability {gap}")` — if someone tried to build this before and failed, learn from their failure. |
| **At boot** | Claude Code identifies the gap from the Attempt Store + verification traces. Registry + KG queried for existing matches. If no match and budget allows, create $G_{build}$ as a new goal node with constraints derived from the capability gap. This goal enters SOLVE recursively — same kernel. |
| **Growth path** | Structured gap analysis: extract the specific capability tag needed, the constraints it must satisfy, and the cost ceiling for the build. Historical build attempts inform whether BUILD is likely to succeed within budget. |

### Step 11: Over Budget → HIRE

|     |     |
|-----|-----|
| **What happens** | Can't build the resolver within compute budget. Route to human. Present the gap with full context. |
| **Input** | Capability gap description, attempt history from Attempt Store (what was tried and why), budget/time state, Knowledge Graph context |
| **Output** | Human-built tool (resolver/verifier/checker) → registered in Registry + Knowledge Graph (toolization is mandatory; no one-off answers) |
| **Infra required** | Human interface (present gap + context, accept resolver), Registry (register human resolver), Knowledge Graph (add `built_by→actor:human` edge, `provides→capability:<tag>` edge) |
| **Context presented to human** | Structured summary: "Need: \{capability\}. Tried: \{attempts with outcomes\}. Failed because: \{root causes\}. Budget remaining: \{amount\}. Similar past resolutions: \{KG results\}." |
| **At boot** | The system's first HIRE calls will be frequent — it starts with minimal tools. Each human-built tool persists. The system never asks for the same capability request fingerprint twice. |
| **Growth path** | As the Knowledge Graph accumulates resolvers, HIRE frequency drops. The system may eventually build a "meta-resolver" that predicts when HIRE is inevitable (bypassing futile BUILD attempts). |

### Step 12: Global Consistency Check

|     |     |
|-----|-----|
| **What happens** | All leaves resolved. Check that leaf outputs are mutually consistent — no cross-branch contradictions. |
| **Input** | All leaf artifacts, all resolved constraint sets, cross-branch relationships |
| **Output** | SAT (consistent) → proceed to assembly, OR UNSAT (conflict detected) → generate coupling constraints + reopen the minimal affected subtrees (no history deletion) |
| **Infra required** | Node Store (traverse tree), Constraint Store (persist new coupling constraints), Knowledge Graph (record conflicts + coupling edges), Tiered Log |
| **At boot** | Claude Code reviews leaf outputs for contradictions. |
| **Growth path** | Datalog transitive closure detects coupling automatically (e.g., `thermal_path(X, Z) :- thermal_path(X, Y), thermally_coupled(Y, Z)` finds ALL thermal contributors, not just obvious ones). Per Symbolic Reasoning doc. |

### Step 13: Assemble Artifact

|     |     |
|-----|-----|
| **What happens** | Compose leaf outputs bottom-up into final deliverable $\mathcal{A}(G)$. |
| **Input** | All leaf artifacts (with lineage), tree structure (composition hierarchy) |
| **Output** | `Artifact { type, content, lineage, σ_system }` where $\sigma_{system} = \min_{l \in leaves} \sigma_{path}(l)$ |
| **Infra required** | Node Store, Knowledge Graph (register final artifact with full lineage graph), Tier 1 Log (final summary) |
| **At boot** | Claude Code assembles. For code: concatenation + integration. For designs: composition per tree structure. |
| **Growth path** | Domain-specific assemblers (code integrator, CAD assembler, document merger) built when domains demand it. |

## 2. Minimum Data Structures

One local SQLite database (`kernel.db`) plus a content-addressed artifact directory (`artifact_store/`). SQLite is the system's source of truth; the artifact store holds large blobs (code, archives, images) addressed by hash.

### A. Node Store (M0 Tree + Memory)

The goal tree is the system's state. Every node is a persistent state machine. Nodes store *IDs* for constraints and artifacts; canonical objects live in `constraints` and `artifacts`.

```sql
-- Canonical constraints (typed logic|semantic; payload is open-ended)
CREATE TABLE constraints (
    id          TEXT PRIMARY KEY,
    type        TEXT NOT NULL,               -- logic | semantic
    domain      TEXT,                        -- e.g. software, cost, schedule, security
    title       TEXT NOT NULL,               -- short human label
    payload     JSON NOT NULL,               -- checker spec (logic) or rubric (semantic)
    provenance  JSON NOT NULL,               -- {source, resolver_id, derived_from_constraint_ids[]}
    created_at  TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_constraints_type ON constraints(type);
CREATE INDEX idx_constraints_domain ON constraints(domain);

-- Canonical artifacts (content lives at uri; lineage is required for auditability)
CREATE TABLE artifacts (
    id          TEXT PRIMARY KEY,
    node_id     TEXT NOT NULL,               -- producing node
    type        TEXT NOT NULL,               -- code | doc | data | resolver | tool_plan | ...
    format      TEXT NOT NULL,               -- python | markdown | json | ...
    uri         TEXT NOT NULL,               -- e.g. artifact_store/<sha256> or external ref
    sha256      TEXT NOT NULL,               -- content hash (required for local storage)
    size_bytes  INTEGER,
    metadata    JSON NOT NULL DEFAULT '{}',
    lineage     JSON NOT NULL,               -- {resolver_id, constraint_ids[], input_artifact_ids[]}
    sigma       REAL,                        -- artifact-local confidence (optional; node-level is authoritative)
    created_at  TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_artifacts_node ON artifacts(node_id, created_at);
CREATE INDEX idx_artifacts_sha ON artifacts(sha256);

-- Resolution attempts (the TRY loop history for a node's step-5 resolver runs)
CREATE TABLE resolution_attempts (
    id              TEXT PRIMARY KEY,
    node_id         TEXT NOT NULL,
    attempt_index   INTEGER NOT NULL,        -- 1..T_max
    resolver_id     TEXT NOT NULL,
    plan_artifact_id    TEXT,                -- tool_plan used for this attempt (required once selection is done)
    input_artifact_ids  JSON NOT NULL DEFAULT '[]',
    output_artifact_id  TEXT,                -- set after RESOLVE
    feedback        JSON,                    -- violation trace from prior attempt (if any)
    verification    JSON,                    -- Verification object (set after VERIFY)
    verdict         TEXT,                    -- SAT | UNSAT (denormalized from verification)
    sigma_v         REAL,                    -- denormalized from verification
    failure_mode    TEXT,                    -- failure:<slug> (optional)
    cost            REAL DEFAULT 0,
    time_seconds    REAL DEFAULT 0,
    created_at      TEXT DEFAULT (datetime('now')),
    UNIQUE (node_id, attempt_index)
);

CREATE INDEX idx_attempts_node ON resolution_attempts(node_id, created_at);
CREATE INDEX idx_attempts_resolver ON resolution_attempts(resolver_id, created_at);

-- Capability request dedupe (enables "never ask twice for the same request")
-- request_fingerprint = sha256(canonical_json({capability_tag, build_constraints, budget_class}))
CREATE TABLE capability_requests (
    id                  TEXT PRIMARY KEY,
    root_id             TEXT NOT NULL,
    capability_tag      TEXT NOT NULL,
    request_fingerprint TEXT NOT NULL,
    status              TEXT NOT NULL,       -- requested | delivered | declined
    resolver_id         TEXT,                -- set when delivered
    detail              JSON,                -- human-visible context + reasons
    created_at          TEXT DEFAULT (datetime('now')),
    UNIQUE (root_id, capability_tag, request_fingerprint)
);

CREATE INDEX idx_capreq_lookup ON capability_requests(capability_tag, status);

-- The goal tree (state machine)
CREATE TABLE nodes (
    id              TEXT PRIMARY KEY,
    parent_id       TEXT REFERENCES nodes(id),
    root_id         TEXT NOT NULL,           -- stable root for this run/tree

    goal_description TEXT NOT NULL,
    goal_summary    TEXT NOT NULL,           -- short summary used for KG/prompt injection
    goal_modality   TEXT NOT NULL,           -- text, code, image, sketch, etc.
    goal_artifact_id TEXT,                   -- optional: artifact id when goal is non-text

    status          TEXT NOT NULL DEFAULT 'pending',
        -- pending | identifying_constraints | selecting_constraints |
        -- selecting_resolver | resolving | verifying |
        -- decomposing | gated | resolved | infeasible

    leaf_kind       TEXT,                    -- atomic | off_the_shelf | contracted | constraint_exhausted
    leaf_reason     TEXT,

    -- Constraint sets (JSON arrays of constraint IDs; canonical objects live in constraints table)
    constraints_core_ids        JSON NOT NULL DEFAULT '[]',  -- human-seeded irreducible core
    constraints_eligible_ids    JSON NOT NULL DEFAULT '[]',  -- step 2 output
    constraints_selected_ids    JSON NOT NULL DEFAULT '[]',  -- step 3 output (C_v*)
    constraints_inherited_ids   JSON NOT NULL DEFAULT '[]',  -- ancestors' resolved constraints (assumed true)
    constraints_resolved_ids    JSON NOT NULL DEFAULT '[]',  -- constraints proven SAT at this node

    resolver_id     TEXT,
    execution_plan_artifact_id TEXT,         -- selected tool_plan for step 5
    input_artifact_ids  JSON NOT NULL DEFAULT '[]',
    output_artifact_id  TEXT,
    last_attempt_id     TEXT,                -- points at resolution_attempts.id
    verification        JSON,                -- last Verification (denormalized from last_attempt_id)

    sigma_v         REAL,                    -- last node-level confidence
    sigma_path      REAL,                    -- product along root→node path (stored for auditability)
    tau_local       REAL NOT NULL,           -- copied from config at node creation
    tau_global      REAL NOT NULL,           -- copied from config at node creation

    attempt_count   INTEGER DEFAULT 0,
    max_attempts    INTEGER DEFAULT 5,       -- T_max

    -- Budget cache (authoritative source is budget_ledger)
    budget_allocated REAL DEFAULT 0,
    budget_spent     REAL DEFAULT 0,
    time_allocated   REAL DEFAULT 0,         -- seconds
    time_spent       REAL DEFAULT 0,

    -- Metadata
    created_at      TEXT DEFAULT (datetime('now')),
    resolved_at     TEXT
);

CREATE TABLE node_dependencies (
    node_id         TEXT REFERENCES nodes(id),
    depends_on      TEXT REFERENCES nodes(id),  -- must be resolved before node_id can execute
    PRIMARY KEY (node_id, depends_on)
);

CREATE INDEX idx_nodes_parent ON nodes(parent_id);
CREATE INDEX idx_nodes_root ON nodes(root_id);
CREATE INDEX idx_nodes_status ON nodes(status, created_at);
```

**Valid state transitions** (enforced by state machine code):

```
pending → identifying_constraints
identifying_constraints → selecting_constraints
selecting_constraints → selecting_resolver
selecting_resolver → resolving
resolving → verifying
verifying → decomposing          (SAT + not leaf)
verifying → resolved             (SAT + leaf)
verifying → gated                (SAT + not leaf + σ_path < τ_global)
verifying → selecting_resolver   (UNSAT, retry < T_max — may pick different resolver)
verifying → resolving            (UNSAT, retry < T_max — same resolver, new feedback)
verifying → pending              (BUILD subgoal created; node waits on dependency)
selecting_resolver → infeasible  (BUILD failed + HIRE failed)
decomposing → pending            (children created, node waits for children)
gated → pending                  (ancestor reopened; node invalidated and will re-run)
```

**Invariants (MVP):**

* `constraints_*_ids` columns store IDs only; fetch canonical objects from `constraints`.
* `constraints_resolved_ids` is the last SAT-verified subset of `constraints_selected_ids` for the node's *current* output. It may be cleared on invalidation/backtracking; history is preserved in `resolution_attempts`.
* `constraints_inherited_ids` at child creation time is the union of ancestor `constraints_resolved_ids` (formalization §3/§8).
* `attempt_count` equals the count of `resolution_attempts` for the node; `last_attempt_id` is the highest `attempt_index`.
* `sigma_v` is the last verification's `sigma_v`. `sigma_path` is recomputed deterministically whenever an ancestor's `sigma_v` changes.
* `execution_plan_artifact_id` must reference an `artifacts` row of `type="tool_plan"` (format json). Each `resolution_attempts.plan_artifact_id` records which plan was executed.
* `resolution_attempts.resolver_id` is the plan's final-step `resolver_id` (the step that produced `output_artifact_id`).
* If the tool plan has exactly 1 step, `nodes.resolver_id` is set to that step's `resolver_id` as a denormalized index; otherwise it may be null.

**State machine** `**next_action**` **(MVP; deterministic):**

```
if status == pending:
  if has_unresolved_dependencies(node): WAIT
  elif constraints_eligible_ids is empty: IDENTIFY_CONSTRAINTS
  elif constraints_selected_ids is empty: SELECT_CONSTRAINTS
  elif execution_plan_artifact_id is null: SELECT_RESOLVER
  elif output_artifact_id is null: RESOLVE
  else: VERIFY

if status == identifying_constraints: IDENTIFY_CONSTRAINTS
if status == selecting_constraints:  SELECT_CONSTRAINTS
if status == selecting_resolver:     SELECT_RESOLVER
if status == resolving:              RESOLVE
if status == verifying:              VERIFY
if status == decomposing:            DECOMPOSE
if status == gated:                  WAIT
if status in {resolved, infeasible}: WAIT
```

**Leaf classification (formalization §9; MVP semantics):**

* `atomic`: the node can be satisfied by a single bounded action (for software: a small patch, a small feature, or a single integration step).
* `off_the_shelf`: there exists an existing package/service that satisfies $C_v^*$; artifact is a dependency choice + minimal integration notes + acceptance checks.
* `contracted`: satisfiable via an external contract/SLA; MVP treats this as `off_the_shelf` with a placeholder artifact.
* `constraint_exhausted`: `constraints_eligible_ids` is empty after excluding `constraints_inherited_ids`; nothing remains to decompose against.

At boot, leaf classification is a Claude Code judgment call that must output `{is_leaf, leaf_kind, leaf_reason}` and is recorded in logs/attempt metadata.

### B. Resolver Registry

The library of tools. Every resolver entry is a tool the system can route to. Tools may be:

* atomic (single implementation: LLM call, test runner, AWS adapter, etc.)
* composite (internally orchestrates other tools; composition optionally recorded via KG `composed_of` edges and/or a tool plan artifact)

```sql
CREATE TABLE resolvers (
    id              TEXT PRIMARY KEY,
    label           TEXT NOT NULL,            -- human-readable name
    description     TEXT,                     -- free-text description (non-rigid; used by router at boot)
    type            TEXT NOT NULL,            -- digital | physical | hybrid
    capabilities    JSON NOT NULL,            -- ["decompose", "generate_code", ...]
    cost_per_invocation REAL,
    cost_per_token  REAL,
    latency_estimate REAL,                    -- seconds

    -- Quality vector (formalization §4)
    q_completeness  REAL DEFAULT 0.5,
    q_executability REAL DEFAULT 0.5,
    q_optimality    REAL DEFAULT 0.5,
    q_representation REAL DEFAULT 0.5,
    q_generalization REAL DEFAULT 0.5,
    q_efficiency    REAL DEFAULT 0.5,

    input_types     JSON,                     -- ["text", "code", ...]
    output_types    JSON,                     -- ["text", "code", "structured_data", ...]

    -- Usage stats (updated by resolution logger)
    usage_count     INTEGER DEFAULT 0,
    success_count   INTEGER DEFAULT 0,
    failure_count   INTEGER DEFAULT 0,
    avg_cost        REAL DEFAULT 0,

    -- Provenance
    built_by        TEXT,                     -- "boot" | "system" | "human"
    built_for_node_id TEXT,                   -- node_id that triggered construction
    metadata        JSON NOT NULL DEFAULT '{}',
    created_at      TEXT DEFAULT (datetime('now'))
);
```

**Resolver execution contract (required; applies to all** $f \in \mathcal{F}$**):**

* `estimate_cost_time(node, constraints_selected_ids, input_artifact_ids) -> {estimated_cost, estimated_time_seconds}`
* `resolve(constraints_selected_ids, input_artifact_ids, feedback?) -> {output_artifact_id, actual_cost, actual_time_seconds, metadata}`
* Errors are classified as:
  * `retryable` (network flake, tool timeout): does not consume an attempt index unless a resolver output was produced.
  * `permanent` (capability mismatch, invalid format): recorded as an UNSAT attempt with `failure_mode`.
* Registration time: for each `capability` tag on the resolver, create/refresh `resolver:<id> --provides--> capability:<tag>` edges in the Knowledge Graph.
* Non-rigid routing: `capabilities` are coarse hints, not a complete symbolic spec. Routing may also use `description`, I/O modality compatibility, and empirical outcomes from KG.
* Composite tools: if a resolver is a composition of other resolvers, record `resolver:<id> --composed_of--> resolver:<part>` edges with `{order, role}` properties (optional but recommended).

### C. Knowledge Graph

Typed nodes and edges in SQLite. Not a full graph database — just two tables with indexed queries.

**ID scheme (required for implementability):**

* Every `kg_nodes.id` is namespaced (see §0). Example: `resolver:claude-code-0`, `goal_pattern:…`, `attempt:…`, `constraint:…`, `artifact:…`, `capability:execute_code`, `actor:human`.
* `kg_nodes` is a *secondary index* used for retrieval and ranking. Canonical objects live in the M0 tables:
  * Nodes: `nodes`
  * Constraints: `constraints`
  * Artifacts: `artifacts`
  * Attempts: `resolution_attempts`
  * Resolvers: `resolvers`

```sql
-- Graph nodes (entities the system knows about)
CREATE TABLE kg_nodes (
    id              TEXT PRIMARY KEY,
    type            TEXT NOT NULL,
        -- resolver | goal_pattern | constraint | resolution_attempt |
        -- artifact | capability | failure_mode | actor
    label           TEXT NOT NULL,            -- human-readable summary
    properties      JSON,                     -- type-specific data
    created_at      TEXT DEFAULT (datetime('now'))
);

-- Graph edges (relationships between entities)
CREATE TABLE kg_edges (
    id              TEXT PRIMARY KEY,
    source_id       TEXT NOT NULL REFERENCES kg_nodes(id),
    target_id       TEXT NOT NULL REFERENCES kg_nodes(id),
    relation        TEXT NOT NULL,
        -- tried_for:       resolution_attempt → goal_pattern
        -- succeeded_on:    resolver → goal_pattern
        -- failed_on:       resolver → goal_pattern
        -- failed_because:  resolution_attempt → failure_mode
        -- provides:        resolver → capability
        -- composed_of:     resolver → resolver (properties: {order, role})
        -- built_by:        resolver → resolver | actor:human
        -- replaced_by:     resolver → resolver (better one found)
        -- similar_to:      goal_pattern → goal_pattern
        -- enables:         capability → capability
        -- requires:        goal_pattern → capability
        -- produced:        resolver → artifact
        -- used_constraint: resolution_attempt → constraint
    properties      JSON,                     -- edge-specific data (e.g., sigma_v, cost, time_seconds, timestamp)
    created_at      TEXT DEFAULT (datetime('now'))
);

-- Indexes for the queries the system actually runs
CREATE INDEX idx_kg_edges_source ON kg_edges(source_id, relation);
CREATE INDEX idx_kg_edges_target ON kg_edges(target_id, relation);
CREATE INDEX idx_kg_edges_relation ON kg_edges(relation);
CREATE INDEX idx_kg_nodes_type ON kg_nodes(type);
CREATE INDEX idx_kg_nodes_label ON kg_nodes(type, label);
```

**Boot-time queries** (MVP versions; parameterize by IDs, not keyword LIKEs):

**Goal pattern identity**:

* `goal_pattern_id = "goal_pattern:" + sha256(normalize(goal_summary))`
* `normalize` = lowercase + trim + collapse whitespace

**Constraint signature (secondary similarity signal):**

* `constraint_signature = sorted(set((c.type, c.domain) for c in constraints_selected))` — captures the structural shape of what a goal requires.
* Budget-class constraints (`domain="cost"`, `domain="schedule"`) are excluded from the signature — they are low-signal for resolution strategy similarity.
* Similarity queries use `constraint_signature` as a filter *before* text-based `goal_summary` matching to reduce false positive cache hits (e.g., two goals with similar text but different constraint structures should not share resolution history).
* Stored as a property on `kg_nodes` of type `goal_pattern`: `properties.constraint_signature = [["logic","latency"],["semantic","code_quality"]]`.

**Resolver selection context** (step 4):

```sql
-- Similar goal patterns (at boot: similarity computed semantically and persisted as similar_to edges)
-- similar_to edge properties: { "score": float in [0,1] }
SELECT gp2.id, gp2.label, CAST(e.properties->>'score' AS REAL) AS score
FROM kg_nodes gp1
JOIN kg_edges e ON e.source_id = gp1.id AND e.relation = 'similar_to'
JOIN kg_nodes gp2 ON gp2.id = e.target_id AND gp2.type = 'goal_pattern'
WHERE gp1.id = '{goal_pattern_id}'
ORDER BY score DESC
LIMIT 10;
```

```sql
-- Resolver outcomes on similar patterns
-- succeeded_on/failed_on edge properties: { "sigma_v": float, "cost": float, "time_seconds": float }
SELECT r.id, r.label,
       SUM(CASE WHEN e.relation = 'succeeded_on' THEN 1 ELSE 0 END) AS successes,
       SUM(CASE WHEN e.relation = 'failed_on' THEN 1 ELSE 0 END) AS failures,
       MAX(CAST(e.properties->>'sigma_v' AS REAL)) AS best_sigma_v
FROM kg_nodes gp
JOIN kg_edges sim ON sim.source_id = '{goal_pattern_id}'
               AND sim.relation = 'similar_to'
               AND sim.target_id = gp.id
JOIN kg_edges e ON e.target_id = gp.id AND e.relation IN ('succeeded_on', 'failed_on')
JOIN kg_nodes r ON r.id = e.source_id AND r.type = 'resolver'
GROUP BY r.id
ORDER BY successes DESC, best_sigma_v DESC;
```

```sql
  -- Filter candidates by input profile compatibility
  -- input_constraint_signature is a JSON array stored in edge properties
  SELECT r.id, r.label,
         COUNT(CASE WHEN e.relation = 'succeeded_on'
               AND e.properties->>'input_modality' = '{goal_modality}'
               THEN 1 END) AS modality_successes
  FROM kg_edges e
  JOIN kg_nodes r ON r.id = e.source_id AND r.type = 'resolver'
  WHERE e.relation IN ('succeeded_on', 'failed_on')
    AND e.target_id IN ({similar_goal_pattern_ids})
  GROUP BY r.id
  ORDER BY modality_successes DESC;
```

**Lookup-before-build** (step 10):

```sql
-- Resolvers that claim to provide a capability (capability tags are canonical strings)
SELECT r.id, r.label
FROM kg_nodes cap
JOIN kg_edges p ON p.target_id = cap.id AND p.relation = 'provides'
JOIN kg_nodes r ON r.id = p.source_id AND r.type = 'resolver'
WHERE cap.id = 'capability:{required_capability}';
```

**Failure pattern detection** (step 9, feedback generation):

```sql
-- "What failure modes have we seen for this goal_pattern?"
SELECT fm.label, COUNT(*) as occurrences
FROM kg_nodes gp
JOIN kg_edges e_try ON e_try.target_id = gp.id AND e_try.relation = 'tried_for'
JOIN kg_nodes a ON a.id = e_try.source_id AND a.type = 'resolution_attempt'
JOIN kg_edges e_fail ON e_fail.source_id = a.id AND e_fail.relation = 'failed_because'
JOIN kg_nodes fm ON fm.id = e_fail.target_id AND fm.type = 'failure_mode'
WHERE gp.id = '{goal_pattern_id}'
GROUP BY fm.id
ORDER BY occurrences DESC;
```

**How results get injected into prompts**:

```
[RESOLUTION HISTORY for goal: "{goal_summary}"]

Similar goals attempted: 3
  - "build web scraper for news sites" (sigma_v=0.9, resolved)
    Resolver: claude-code-0 → succeeded after 2 retries
    Key constraint: rate_limit < 1 req/sec (logic, satisfied)

  - "build API data fetcher" (sigma_v=0.7, resolved)
    Resolver: claude-code-0 → failed (timeout on large payloads)
    Resolver: codex-tools-0 → succeeded (async implementation)
    Failure mode: synchronous HTTP calls don't scale

Known failure modes for this goal type:
  - synchronous_io (3 occurrences) — fix: use async/concurrent patterns
  - missing_error_handling (2 occurrences) — fix: add retry with backoff

[END RESOLUTION HISTORY]
```

This block gets prepended to Claude Code's prompt at steps 2, 4, 8, 9, 10. The exact query varies by step (step 4 emphasizes resolver success/failure, step 9 emphasizes failure modes).

**How new edges get created** (automatically, after every operation):

| After step | Edges created |
|------------|---------------|
| Step 2 (identify constraints) | `goal_pattern:<id> --requires--> capability:<tag>` for each required capability tag |
| Step 5 (resolve) | `attempt:<id> --tried_for--> goal_pattern:<id>`, `attempt:<id> --used_constraint--> constraint:<id>` for each $c \in C_v^*$ |
| Step 6 (verify SAT) | `resolver:<id> --succeeded_on--> goal_pattern:<id>` with `{sigma_v, cost, time_secondsinput_constraint_signature, input_modality, input_size_class}` in edge properties |
| Step 6 (verify UNSAT) | `resolver:<id> --failed_on--> goal_pattern:<id>` with `{sigma_v, cost, time_seconds, input_constraint_signature, input_modality, input_size_class}` in edge properties, `attempt:<id> --failed_because--> failure:<slug>` |
| Step 7 (register artifact) | `resolver:<id> --produced--> artifact:<id>` (and optionally `attempt:<id> --produced--> artifact:<id>`) |
| Step 10 (BUILD succeeds) | `resolver:<new> --built_by--> resolver:<builder>`, old `resolver:<old> --replaced_by--> resolver:<new>`, and if composite: `resolver:<new> --composed_of--> resolver:<part>` edges |
| Step 11 (HIRE) | `resolver:<new> --built_by--> actor:human`, plus `resolver:<new> --provides--> capability:<tag>` |

### D. Budget Tracker

Per-node from boot. Parent allocates to children.

```sql
CREATE TABLE budget_ledger (
    id              TEXT PRIMARY KEY,
    node_id         TEXT NOT NULL REFERENCES nodes(id),
    event_type      TEXT NOT NULL,           -- allocate | spend | refund
    amount          REAL NOT NULL,
    time_amount     REAL DEFAULT 0,          -- seconds
    resolver_id     TEXT,
    description     TEXT,
    created_at      TEXT DEFAULT (datetime('now'))
);

-- Materialized view (or computed on query)
-- remaining(node_id) = SUM(allocate) - SUM(spend) for that node
```

**Budget protocol**:

**Source of truth**: `budget_ledger` is authoritative. Any `nodes.budget_*` / `nodes.time_*` fields are caches that must be derived from the ledger (or updated transactionally on every ledger write).

**Applies to every resolver call**: the Budget Enforcer wraps *all* resolver calls (identify/select/decompose/verify/resolve), not just step 5.

```
ALLOCATE(parent_id, children[]):
    parent_remaining = budget.remaining(parent_id)
    reserve = parent_remaining * 0.20
    allocatable = parent_remaining - reserve

    # At boot: Claude Code estimates relative complexity
    # Growth path: historical avg_cost for similar goal patterns from KG
    weights = estimate_complexity(children)

    for child, weight in zip(children, normalize(weights)):
        budget.allocate(child.id, allocatable * weight)

PRE_CHECK(node_id, estimated_cost, estimated_time):
    remaining_cost = budget.remaining_cost(node_id)
    remaining_time = budget.remaining_time(node_id)
    if estimated_cost > remaining_cost or estimated_time > remaining_time:
        # Check if parent has reserve
        parent_remaining_cost = budget.remaining_cost(node.parent_id)
        parent_remaining_time = budget.remaining_time(node.parent_id)
        if parent_remaining_cost >= estimated_cost and parent_remaining_time >= estimated_time:
            # Reallocate from parent reserve (cost + time)
            budget.allocate(node_id, estimated_cost, estimated_time)
            return ALLOWED
        return DENIED  → escalate to HIRE
    return ALLOWED

RECORD(node_id, actual_cost, actual_time, resolver_id):
    budget.spend(node_id, actual_cost, actual_time, resolver_id)
```

### E. Tiered Log

```sql
CREATE TABLE logs (
    id              TEXT PRIMARY KEY,
    tier            INTEGER NOT NULL,        -- 1, 2, or 3
    node_id         TEXT,
    timestamp       TEXT DEFAULT (datetime('now')),
    category        TEXT,                    -- flow | decision | io
    summary         TEXT,                    -- human-readable (tier 1)
    detail          JSON                     -- structured data (tier 2, 3)
);

CREATE INDEX idx_logs_tier ON logs(tier, timestamp);
CREATE INDEX idx_logs_node ON logs(node_id, tier);
```

**Tier examples**:

```
Tier 1 (Flow):
  [10:15:03] Node abc123: RESOLVE using claude-code-0 → UNSAT (sigma_v=0.4)
  [10:15:08] Node abc123: RETRY 2/5 with feedback
  [10:15:15] Node abc123: RESOLVE using claude-code-0 → SAT (sigma_v=0.8)
  [10:15:15] Node abc123: Leaf resolved. Budget: $0.12 spent of $0.50 allocated. Time: 12s/60s.

Tier 2 (Decisions):
  { "node": "abc123", "decision": "resolver_selection",
    "candidates": [{"id": "claude-code-0", "score": 0.7}, {"id": "codex-tools-0", "score": 0.3}],
    "chosen": "claude-code-0",
    "rationale": "higher completeness for semantic tasks",
    "kg_context": "2 prior successes for similar goals" }

Tier 3 (I/O Metadata; raw content optional):
  { "node": "abc123", "resolver": "claude-code-0",
    "input_artifact_ids": ["artifact:…"], "output_artifact_id": "artifact:…",
    "tokens": 4523, "cost": 0.067, "time_seconds": 3.2,
    "raw_io_policy": "redacted_by_default" }
```

## 3. Minimum Boot Infrastructure

What must be **actual code** at boot vs. what Claude Code handles.

### Must Be Code (Deterministic, σ = 1.0)

| #   | Component | What It Does | Why It Can't Be Claude Code |
|-----|-----------|--------------|-----------------------------|
| 1   | **Dispatch Loop** | Main event loop. Pops nodes from queue (priority: dependency-unblocked, then FIFO). Determines next action from node status. Executes. | The system's heartbeat. Must be deterministic and never skip a step. |
| 2   | **Node State Machine** | Validates and executes state transitions. Rejects invalid transitions (e.g., `pending → resolved` skipping verification). | Enforcer axiom B5: the operation sequence is non-negotiable. |
| 3   | **Budget Enforcer** | Wraps every resolver call. Pre-check → execute → record. Blocks on overbudget. Manages per-node allocation. | Axiom B1/B2: cost/time enforcement is logic-typed (σ=1.0). Arithmetic, not judgment. |
| 4   | **Dependency Resolver** | Computes topological sort of sibling subgoals. Blocks nodes whose dependencies aren't resolved. Updates queue priority. | Axiom B3: dependency ordering is a correctness requirement. Topological sort is deterministic. |
| 5   | **Knowledge Graph CRUD** | Create/read/update nodes and edges. Run the boot-time queries (resolver selection, lookup-before-build, failure patterns). | Database operations. Must be reliable and consistent. |
| 6   | **Resolution + Attempt Logger** | Persists `resolution_attempts` rows (attempt_index, resolver_id, feedback, output_artifact_id, verification, verdict, sigma_v, cost, time_seconds). Creates corresponding KG nodes/edges. | The growth flywheel requires lossless attempt history + interpretable traces. |
| 7   | **Tiered Logger** | Writes to 3 tiers. Tier 1 always, Tier 2 on decisions, Tier 3 stores I/O metadata; raw prompts/outputs are opt-in and redacted by default. | Observability without leaking sensitive content. |
| 8   | **Registry CRUD + Query** | Register resolvers, query by capability tags, rank by quality vector. | Must be reliable. Capability matching is string-set operations, not judgment. |
| 9   | **Lookup-Before-Build Gate** | Before BUILD, mandatory cache check (registry capability match + KG history). If matching resolver found, skip BUILD and retry. | Axiom B4: lookup-before-build. Prevents redundant construction. |
| 10  | **Artifact Store** | Content-addressed storage for artifacts (`sha256` → `uri`), plus artifact metadata persistence. | Without a real artifact store, lineage and verification are not reproducible. |

### Claude Code at Boot (Semantic, σ &lt; 1.0)

| #   | Operation | Formalization | Growth Path |
|-----|-----------|---------------|-------------|
| 1   | Constraint identification | §3 ($\rho$)   | Datalog forward-chaining ($\sigma = 1.0$) |
| 2   | Constraint selection (parsimony) | §3            | Z3/SAT parsimony checker |
| 3   | Resolver selection (ranking beyond capability match) | §4            | Learned policy from KG data |
| 4   | Resolution (the actual work) | §4            | Domain-specific resolvers |
| 5   | Verification (semantic) | §5            | Datalog rules, Z3 checks, test suites |
| 6   | Decomposition + dependency estimation | §8            | Domain-specific decomposers |
| 7   | Feedback generation | §7            | Z3 abductive repair, ASP alternatives |
| 8   | Gap detection for BUILD | §4.3          | Structured capability gap analysis |
| 9   | Goal synthesis | §14 M1        | Domain-specific goal synthesizers |
| 10  | Child budget allocation estimation | —             | Historical cost data from KG |

**The split**: deterministic infrastructure is code (reliable, σ=1.0, never wrong about its own operation). Judgment calls are Claude Code (imperfect, σ&lt;1.0, but improves as KG accumulates data).

## 4. Boot Sequence

```
kernel_boot():
    1. db = init_sqlite("kernel.db")
       create_tables(db, [constraints, artifacts, resolution_attempts,
                          capability_requests,
                          nodes, node_dependencies,
                          resolvers, kg_nodes, kg_edges,
                          budget_ledger, logs])
       init_artifact_store("artifact_store/")  # content-addressed by sha256

    2. register_boot_resolvers(db):
       - self-model-0:   label="Self Model", type=digital, capabilities=[system_state, capability_enumeration,
                          architecture_description, cost_estimation], cost=0
       - claude-code-0:  label="Claude Code", type=digital, capabilities=[decompose, reason, generate_code,
                          verify_semantic, identify_constraints, select_constraints,
                          select_resolver, generate_feedback, escalate, gate_confidence,
                          check_global_consistency, assemble_artifact, orchestrate,
                          synthesize_goal, dispatch, enforce_invariants],
                          cost_per_token=0.015
       - codex-tools-0:  label="Codex Tools", type=digital, capabilities=[execute_code, file_ops, web_search,
                          package_install, shell_execution, test_execution],
                          cost_per_invocation=0.01
       - (registration side effect): for each resolver capability tag X,
         KG edge `resolver:<id> --provides--> capability:X` is created/refreshed

    3. budget = BudgetEnforcer(db, total_budget=config.budget,
                                    total_time=config.time_limit)

    4. logger = TieredLogger(db)

    5. registry = ResolverRegistry(db)
       kg = KnowledgeGraph(db)
       state_machine = NodeStateMachine(valid_transitions)
       queue = NodeQueue()  # in-memory is acceptable for MVP

    6. # Await goal input
       (goal_description, constraints_core) = await_human_input()
       core_ids = upsert_constraints(db, constraints_core)
       root_id = create_node(db, {
           "id": new_id(),
           "root_id": "<same as id>",
           "parent_id": null,
           "goal_description": goal_description,
           "goal_summary": summarize(goal_description),  # at boot: Claude Code or a simple truncation
           "goal_modality": "text",
           "constraints_core_ids": core_ids,
           "constraints_inherited_ids": [],
           "tau_local": config.tau_local,
           "tau_global": config.tau_global
       })
       budget.allocate(root_id, config.budget, config.time_limit_seconds)
       queue.enqueue(root_id)
       dispatch_loop(queue, db, state_machine, registry, budget, kg, logger)
```

## 5. The Dispatch Loop

The heart of the kernel. Pseudocode with all infrastructure hooks integrated.

```python
def dispatch_loop(queue, db, state_machine, registry, budget, kg, logger):
    """
    Deterministic dispatcher for the Formalization §13 loop.
    Every resolver call is budget-gated (cost + wall time).
    """

    while not queue.empty():
        node_id = queue.pop()  # priority: dependency-unblocked, then FIFO
        node = db.nodes.get(node_id)

        # Skip blocked nodes (re-enqueue)
        if has_unresolved_dependencies(node_id, db):
            queue.enqueue(node_id, priority="LOW")
            continue

        action = state_machine.next_action(node)
        logger.tier1(node_id, f"Action: {action}")

        def call_with_budget(operation, resolver_id, estimate_fn, run_fn):
            """Budget wrapper used for *every* resolver call (B1/B2)."""
            est = estimate_fn()  # {cost, time_seconds}
            if budget.pre_check(node_id, est["cost"], est["time_seconds"]) == "DENIED":
                return {"status": "DENIED", "estimate": est}
            out = run_fn()       # {result, cost, time_seconds, io_metadata}
            budget.record(node_id, out["cost"], out["time_seconds"], resolver_id)
            logger.tier3(node_id, operation, resolver_id, out.get("io_metadata", {}))
            return {"status": "OK", "out": out}

        # ── STEP 2: Identify Constraints ──
        if action == "IDENTIFY_CONSTRAINTS":
            ctx = kg.query_constraints_context(node.goal_summary)
            res = call_with_budget(
                "identify_constraints",
                "claude-code-0",
                lambda: claude_code.estimate_identify_constraints(node, ctx),
                lambda: claude_code.identify_constraints(
                    node.goal_description, node.constraints_inherited_ids, ctx
                ),
            )
            if res["status"] == "DENIED":
                escalate_to_hire(node_id, db, budget, kg, registry, logger, state_machine, queue, reason="over_budget")
                continue

            eligible_ids = upsert_constraints(db, res["out"]["constraints"])
            db.nodes.update(node_id, constraints_eligible_ids=eligible_ids)
            state_machine.transition(node_id, "identifying_constraints", "selecting_constraints")
            queue.enqueue(node_id)
            continue

        # ── STEP 3: Select Constraints (Parsimony) ──
        if action == "SELECT_CONSTRAINTS":
            eligible = db.constraints.get_many(node.constraints_eligible_ids)
            res = call_with_budget(
                "select_constraints",
                "claude-code-0",
                lambda: claude_code.estimate_select_constraints(node, eligible),
                lambda: claude_code.select_parsimonious(
                    node.goal_description, eligible, node.constraints_inherited_ids
                ),
            )
            if res["status"] == "DENIED":
                escalate_to_hire(node_id, db, budget, kg, registry, logger, state_machine, queue, reason="over_budget")
                continue

            selected_ids = res["out"]["constraint_ids"]
            db.nodes.update(node_id, constraints_selected_ids=selected_ids)
            state_machine.transition(node_id, "selecting_constraints", "selecting_resolver")
            queue.enqueue(node_id)
            continue

        # ── STEP 4: Select (or compose) Tool Plan ──
        if action == "SELECT_RESOLVER":
            required_caps = required_capabilities(node)
            candidates = registry.query(required_caps, node.goal_modality)
            history = kg.query_resolver_history(node.goal_summary)
            res = call_with_budget(
                "select_tool_plan",
                "claude-code-0",
                lambda: claude_code.estimate_rank_resolvers(candidates, node, history),
                lambda: claude_code.rank_resolvers(candidates, node, history),
            )
            if res["status"] == "DENIED":
                escalate_to_hire(node_id, db, budget, kg, registry, logger, state_machine, queue, reason="over_budget")
                continue

            # Persist the selected tool plan (MVP: usually a single step).
            tool_plan = res["out"]["tool_plan"]  # see §0 "Tool plan artifact"
            plan_artifact_id = persist_artifact(
                db,
                node_id=node_id,
                artifact_type="tool_plan",
                artifact_format="json",
                content=tool_plan,
                lineage={
                    "resolver_id": "claude-code-0",
                    "constraint_ids": node.constraints_selected_ids,
                    "input_artifact_ids": node.input_artifact_ids,
                },
                sigma=None,
            )

            # Denormalize the final-step resolver for indexing (optional).
            final_step_resolver_id = tool_plan["steps"][-1]["resolver_id"]
            db.nodes.update(node_id, {
                "execution_plan_artifact_id": plan_artifact_id,
                "resolver_id": final_step_resolver_id if len(tool_plan["steps"]) == 1 else None,
            })
            state_machine.transition(node_id, "selecting_resolver", "resolving")
            queue.enqueue(node_id)
            continue

        # ── STEP 5: Resolve (TRY loop attempt) ──
        if action == "RESOLVE":
            attempt_index = node.attempt_count + 1
            feedback = feedback_from_last_attempt(db, node.last_attempt_id)

            tool_plan = load_artifact_json(db, node.execution_plan_artifact_id)
            final_step_resolver_id = tool_plan["steps"][-1]["resolver_id"]

            attempt_id = db.resolution_attempts.create({
                "node_id": node_id,
                "attempt_index": attempt_index,
                "resolver_id": final_step_resolver_id,
                "plan_artifact_id": node.execution_plan_artifact_id,
                "input_artifact_ids": node.input_artifact_ids,
                "feedback": feedback,
            })

            # Execute the plan sequentially. Each step is a tool call and is budget-gated.
            current_inputs = node.input_artifact_ids
            total_cost = 0.0
            total_time = 0.0
            output_artifact_id = None
            aborted = False

            for step in tool_plan["steps"]:
                step_resolver = registry.get(step["resolver_id"])
                est = step_resolver.estimate_cost_time(node, node.constraints_selected_ids, current_inputs)
                if budget.pre_check(node_id, est["cost"], est["time_seconds"]) == "DENIED":
                    escalate_to_hire(node_id, db, budget, kg, registry, logger, state_machine, queue, reason="over_budget")
                    db.resolution_attempts.update(attempt_id, {
                        "failure_mode": "failure:over_budget",
                        "verdict": "UNSAT",
                        "cost": total_cost,
                        "time_seconds": total_time,
                    })
                    aborted = True
                    break

                run = step_resolver.resolve(node.constraints_selected_ids, current_inputs, feedback)
                total_cost += run["cost"]
                total_time += run["time_seconds"]

                # Persist step output; later steps consume it as input.
                output_artifact_id = persist_artifact(
                    db,
                    node_id=node_id,
                    artifact_type=run["artifact_type"],
                    artifact_format=run["artifact_format"],
                    content=run["content"],
                    lineage={
                        "resolver_id": step["resolver_id"],
                        "constraint_ids": node.constraints_selected_ids,
                        "input_artifact_ids": current_inputs,
                    },
                    sigma=None,
                )
                current_inputs = [output_artifact_id]

            if aborted:
                continue

            db.resolution_attempts.update(attempt_id, {
                "output_artifact_id": output_artifact_id,
                "cost": total_cost,
                "time_seconds": total_time,
            })
            budget.record(node_id, total_cost, total_time, final_step_resolver_id)

            db.nodes.update(node_id, {
                "output_artifact_id": output_artifact_id,
                "last_attempt_id": attempt_id,
                "attempt_count": attempt_index,
            })
            state_machine.transition(node_id, "resolving", "verifying")
            queue.enqueue(node_id)
            continue

        # ── STEP 6: Verify ──
        if action == "VERIFY":
            attempt = db.resolution_attempts.get(node.last_attempt_id)
            verification = verify_output(
                node=node,
                output_artifact_id=attempt.output_artifact_id,
                constraint_ids=node.constraints_selected_ids,
                registry=registry,
                db=db,
            )  # returns Verification object (see §0)

            db.resolution_attempts.update(attempt.id, {
                "verification": verification,
                "verdict": verification["verdict"],
                "sigma_v": verification["sigma_v"],
                "failure_mode": extract_failure_mode(verification),
            })

            sigma_path = compute_sigma_path(node_id, db)  # product along root→node path
            constraints_resolved_ids = node.constraints_selected_ids if verification["verdict"] == "SAT" else []

            db.nodes.update(node_id, {
                "verification": verification,
                "sigma_v": verification["sigma_v"],
                "sigma_path": sigma_path,
                "constraints_resolved_ids": constraints_resolved_ids,
            })

            record_attempt_in_kg(node, attempt, verification, kg, db)

            if verification["verdict"] == "SAT":
                leaf = classify_leaf(node, attempt.output_artifact_id, verification)
                if leaf["is_leaf"]:
                    db.nodes.update(node_id, {"leaf_kind": leaf["leaf_kind"], "leaf_reason": leaf["leaf_reason"]})
                    register_artifact(node_id, attempt.output_artifact_id, db, kg)
                    state_machine.transition(node_id, "verifying", "resolved")
                    logger.tier1(node_id, f"Resolved (sigma_v={verification['sigma_v']})")
                    check_parent_unblocked(node.parent_id, queue, db)
                else:
                    # Confidence gating (formalization §6; axiom B7)
                    if sigma_path < node.tau_global:
                        state_machine.transition(node_id, "verifying", "gated")
                        backtrack_to_weakest_ancestor(node_id, db, state_machine, queue, logger)
                    else:
                        state_machine.transition(node_id, "verifying", "decomposing")
                        queue.enqueue(node_id)
            else:
                if node.attempt_count < node.max_attempts:
                    state_machine.transition(node_id, "verifying", "resolving")
                    queue.enqueue(node_id)
                else:
                    build_or_hire(node_id, attempt.id, db, budget, kg, registry, logger, state_machine, queue)
            continue

        # ── STEP 8: Decompose ──
        if action == "DECOMPOSE":
            ctx = kg.query_decomposition_history(node.goal_summary)
            res = call_with_budget(
                "decompose",
                "claude-code-0",
                lambda: claude_code.estimate_decompose(node, ctx),
                lambda: claude_code.decompose(
                    node.goal_description,
                    node.constraints_selected_ids,
                    node.output_artifact_id,
                    ctx,
                ),
            )
            if res["status"] == "DENIED":
                escalate_to_hire(node_id, db, budget, kg, registry, logger, state_machine, queue, reason="over_budget")
                continue

            children_spec = res["out"]["children_spec"]  # {children: [...], dependencies: [...]}
            children = []
            for spec in children_spec["children"]:
                child_id = create_node(db, {
                    "root_id": node.root_id,
                    "parent_id": node_id,
                    "goal_description": spec["goal_description"],
                    "goal_summary": spec["goal_summary"],
                    "goal_modality": spec["goal_modality"],
                    "constraints_core_ids": spec.get("constraints_core_ids", []),
                    "constraints_inherited_ids": node.constraints_inherited_ids + node.constraints_resolved_ids,
                    "input_artifact_ids": spec.get("input_artifact_ids", [node.output_artifact_id]),
                    "tau_local": node.tau_local,
                    "tau_global": node.tau_global,
                })
                children.append(child_id)

            for dep in children_spec.get("dependencies", []):
                add_dependency(dep["child_id"], dep["depends_on_id"], db)

            allocate_budgets(node_id, children, budget, kg)
            for child_id in children:
                queue.enqueue(child_id)

            state_machine.transition(node_id, "decomposing", "pending")  # waits for children
            continue

    # ── STEP 12 + 13: Global consistency + Assemble (root-level) ──
    root_id = get_root(db)
    if all_leaves_resolved(root_id, db):
        consistency = check_global_consistency(root_id, db, kg)
        if consistency["verdict"] == "SAT":
            artifact = assemble_artifact(root_id, db, kg)
            logger.tier1(root_id, f"Goal achieved. sigma_system={artifact['sigma_system']}")
            return artifact

        # Monotonic backtracking: add coupling constraints, then reopen affected subtrees.
        apply_coupling_constraints_and_reopen(consistency["conflicts"], db, state_machine, queue, logger)
        return dispatch_loop(queue, db, state_machine, registry, budget, kg, logger)


def build_or_hire(node_id, attempt_id, db, budget, kg, registry, logger, state_machine, queue):
    """Step 10 (BUILD) + Step 11 (HIRE)."""
    node = db.nodes.get(node_id)
    gap = detect_capability_gap(node, db, kg)  # {capability_tag, build_constraints, request_fingerprint, rationale}

    # Lookup-before-build (axiom B4): registry query is mandatory; KG is used for ranking/history.
    existing = registry.query([gap["capability_tag"]], node.goal_modality)
    if existing:
        chosen = existing[0]["id"]
        logger.tier1(node_id, f"Found existing resolver for {gap['capability_tag']}: {chosen}")
        db.nodes.update(node_id, {"resolver_id": chosen, "attempt_count": 0})
        state_machine.transition(node_id, "verifying", "selecting_resolver")
        queue.enqueue(node_id)
        return

    # Dedupe: never ask twice for the same request fingerprint (MVP interpretation).
    if db.capability_requests.exists(node.root_id, gap["capability_tag"], gap["request_fingerprint"], status="declined"):
        db.nodes.update(node_id, {"status": "infeasible"})
        logger.tier1(node_id, f"Infeasible: previously declined capability request {gap['capability_tag']}")
        return

    # Check build budget
    est = claude_code.estimate_build_cost(gap)
    if budget.pre_check(node_id, est["cost"], est["time_seconds"]) == "DENIED":
        escalate_to_hire(node_id, db, budget, kg, registry, logger, state_machine, queue, gap=gap, reason="over_budget")
        return

    # Build as subgoal (recursive SOLVE)
    build_goal_id = create_node(db, {
        "root_id": node.root_id,
        "parent_id": node_id,
        "goal_description": f"Build resolver with capability: {gap['capability_tag']}",
        "goal_summary": f"Build resolver: {gap['capability_tag']}",
        "goal_modality": "text",
        "constraints_core_ids": upsert_constraints(db, gap["build_constraints"]),
        "constraints_inherited_ids": node.constraints_inherited_ids,
        "input_artifact_ids": node.input_artifact_ids,
        "tau_local": node.tau_local,
        "tau_global": node.tau_global,
    })
    budget.allocate(build_goal_id, budget.remaining_cost(node_id) * 0.5, budget.remaining_time(node_id) * 0.5)
    queue.enqueue(build_goal_id)
    add_dependency(node_id, build_goal_id, db)  # original node waits for build result
    state_machine.transition(node_id, "verifying", "pending")


def escalate_to_hire(node_id, db, budget, kg, registry, logger, state_machine, queue, gap=None, reason=None):
    """Step 11: route to human. MVP: human may provide a resolver OR increase budget OR mark infeasible."""
    node = db.nodes.get(node_id)
    gap = gap or detect_capability_gap(node, db, kg)
    history = db.resolution_attempts.list_for_node(node_id)

    logger.tier1(node_id, f"HIRE ({reason}): need human for {gap['capability_tag']}")
    logger.tier1(node_id, f"  Tried: {len(history)} attempts")
    logger.tier1(node_id, f"  Remaining: cost=${budget.remaining_cost(node_id):.2f}, time={budget.remaining_time(node_id):.0f}s")

    db.capability_requests.upsert({
        "root_id": node.root_id,
        "capability_tag": gap["capability_tag"],
        "request_fingerprint": gap["request_fingerprint"],
        "status": "requested",
        "detail": {"reason": reason, "rationale": gap.get("rationale")},
    })

    human_result = await_human_resolution(gap, history, budget.remaining_cost(node_id), budget.remaining_time(node_id))
    if human_result["kind"] == "increase_budget":
        budget.allocate(node_id, human_result["add_cost"], human_result["add_time_seconds"])
        queue.enqueue(node_id)
        return
    if human_result["kind"] == "decline":
        db.capability_requests.update_status(node.root_id, gap["capability_tag"], gap["request_fingerprint"], "declined", detail=human_result)
        db.nodes.update(node_id, {"status": "infeasible"})
        return

    # kind == "resolver"
    human_resolver = human_result["resolver"]
    registry.register(human_resolver)
    kg.ensure_node(f"resolver:{human_resolver['id']}", "resolver", human_resolver["label"])
    kg.ensure_node("actor:human", "actor", "human")
    kg.ensure_node(f"capability:{gap['capability_tag']}", "capability", gap["capability_tag"])
    kg.create_edge(f"resolver:{human_resolver['id']}", "built_by", "actor:human")
    kg.create_edge(f"resolver:{human_resolver['id']}", "provides", f"capability:{gap['capability_tag']}")

    db.capability_requests.update_status(node.root_id, gap["capability_tag"], gap["request_fingerprint"], "delivered", resolver_id=human_resolver["id"])

    db.nodes.update(node_id, {"resolver_id": human_resolver["id"], "attempt_count": 0})
    queue.enqueue(node_id)
```

**Backtracking + invalidation (formalization §6; MVP)**

Confidence gating requires a concrete subtree invalidation rule (otherwise the system will keep decomposing under stale low-confidence ancestors).

**Weakest ancestor selection**:

* Given a gated node `v`, compute the path `root → … → v`.
* Pick `u = argmin σ_u` over nodes on the path with non-null `sigma_v`. Tie-break: choose the *deepest* such node to minimize recomputation.

**Reopen + invalidate**:

* Reopen `u` by transitioning it to `selecting_resolver` (or `selecting_constraints` if the constraint set must change).
* Invalidate the subtree rooted at `u` (excluding `u` itself):
  * For each descendant `w`: clear `output_artifact_id`, `verification`, `constraints_resolved_ids`, `sigma_v`, `sigma_path`; set `status='pending'`.
  * Preserve attempt history (`resolution_attempts`) and logs; do not delete records.
  * Delete and later recompute `node_dependencies` edges for the invalidated subtree (MVP simplification).

This preserves monotonic *constraint propagation* while allowing re-resolution when ancestor confidence improves.

**Global consistency conflicts (formalization §11; MVP)**

`check_global_consistency(root)` returns:

* `verdict: SAT|UNSAT`
* `conflicts[]`: each conflict includes `{nodes_involved[], lca_node_id, coupling_constraints[]}`

On UNSAT, the kernel does **monotonic backtracking**:


1. Persist `coupling_constraints[]` as new constraints (type logic|semantic) in `constraints`.
2. Attach their IDs to `lca_node_id.constraints_eligible_ids` (and/or `constraints_selected_ids` if they are hard interface constraints).
3. Reopen `lca_node_id` (transition to `selecting_constraints`) and invalidate its subtree (as above).
4. Re-enqueue `lca_node_id`.

## 6. Recursive Improvement Mechanisms

How the system gets better with every goal solved. The Knowledge Graph is the flywheel.

### TRY Improvement (within a single goal)

```
Attempt 1: claude-code-0 resolves → UNSAT (constraint C3: latency > 200ms)
  → KG records: attempt:<id> --tried_for--> goal_pattern:<id>
  → KG records: attempt:<id> --failed_because--> failure:synchronous_io
  → KG records: resolver:claude-code-0 --failed_on--> goal_pattern:<id>

Attempt 2: claude-code-0 resolves WITH feedback (violation trace + failure mode from KG)
  → Prompt includes: "Previous attempt failed due to synchronous_io. Known fix: use async."
  → Output uses async → SAT (sigma_v=0.85)
  → KG records: resolver:claude-code-0 --succeeded_on--> goal_pattern:<id> (properties: {sigma_v: 0.85})
```

**Next time a similar goal appears**: KG query at step 4 returns "synchronous_io is a known failure mode; claude-code-0 succeeded after switching to async." Claude Code gets this context upfront → avoids the failure entirely → fewer retries → less cost.

### BUILD Improvement (across goals)

```
Goal A: "build a web scraper" → needs fault_localization capability
  → Registry: no resolver with fault_localization
  → BUILD: create subgoal "build fault localizer"
  → Recursive SOLVE produces a Python AST-based fault localizer
  → Registered: resolver fault-localizer-1, capabilities=[fault_localization]
  → KG: resolver:fault-localizer-1 --built_by--> resolver:claude-code-0
  → KG: resolver:fault-localizer-1 --provides--> capability:fault_localization

Goal B (later): "fix bug in payment service" → needs fault_localization
  → Step 10 lookup-before-build: KG query finds fault-localizer-1
  → Skip BUILD entirely → use existing resolver
  → Cost savings: $X avoided, time savings: Y minutes
```

### HIRE Improvement (human investment compounds)

```
Goal C: "verify thermal properties of PCB design" → needs thermal_analysis capability
  → BUILD attempted → digital resolvers can't do physical thermal analysis
  → HIRE: human builds thermal_analysis_resolver (measurement apparatus + analysis script)
  → Registered: resolver thermal-analyzer-1, type=hybrid, capabilities=[thermal_analysis]
  → KG: resolver:thermal-analyzer-1 --built_by--> actor:human

Goal D (later): "verify thermal properties of enclosure design" → needs thermal_analysis
  → Step 10 lookup-before-build: KG finds thermal-analyzer-1
  → Reuse directly → no human needed
  → The human's one-time investment serves all future thermal goals
```

### System-Level Growth Trajectory

```
t=0:  F = {self-model-0, claude-code-0, codex-tools-0}
      KG = empty
      Every goal → many retries, frequent BUILD, frequent HIRE
      σ_system ~ 0.3-0.5

t=10: F = {boot resolvers + 5 built resolvers + 2 human resolvers}
      KG = ~50 resolution attempts, ~20 goal patterns, ~10 failure modes
      Resolver selection is informed → fewer retries
      BUILD hits cache → fewer builds
      σ_system ~ 0.5-0.7

t=100: F = {boot + 30 built + 8 human resolvers}
       KG = ~500 attempts, ~100 patterns, ~30 failure modes, rich edge structure
       Most goals match existing patterns → fast resolution
       HIRE is rare → most capabilities exist
       σ_system ~ 0.7-0.9

t=1000: F = large resolver library
        KG = comprehensive pattern/failure/success knowledge
        Novel goals still trigger BUILD/HIRE, but routine goals resolve quickly
        Domain-specific verifiers have replaced semantic verification for known domains
        σ_system ~ 0.8-0.95 for known domains
```

### Time-Horizon Meta-Tools (growth targets)

Long-horizon projects require tools that reduce planning/execution horizon and increase self-correction. These are explicit tool categories the system should build when it sees repeated "time-horizon failures" (stalling, low σ_path gating, repeated regressions).

Examples (all are tools; most will be composite):

* **Contract decomposer**: converts a subgoal into an explicit interface/contract artifact (inputs/outputs/acceptance tests) before work begins.
* **Multi-agent executor**: executes independent child nodes in parallel under a shared contract + merges results.
* **Verifier generator**: builds dense verification for a domain (test generators, linters, schema validators) so more constraints become logic-typed (σ=1.0).
* **Self-recovery / resumer**: detects stuck states, restarts safely from persisted state, and performs automatic rollback/redo under budget.
* **Scheduler/router**: learned policy that chooses which tools (and compositions) to use based on historical wins/losses, cost/time, and constraint fit.

### Exploration Policy (build many, select by experience)

To align "build many ideas, persist all, use only what works":

* On repeated failure modes (or high expected value), the system may spawn BUILD subgoals to create alternative tools/strategies (within budget reserve).
* All built tools persist in the library (Registry + KG + artifact store).
* Online selection uses empirical outcomes (success/failure modes, sigma_v, cost, time) as first-class signals; descriptions/capability tags are hints, not ground truth.

## 7. Revised Kernel Axioms

Updated to reflect graph-first knowledge store and per-node budgets.

### Structural Axioms (what must exist at boot)

| #   | Axiom | Why Irreducible |
|-----|-------|-----------------|
| S1  | At least one resolver $f \in \mathcal{F}$ | Can't resolve anything without at least one |
| S2  | Goal interface accepting $G$ in any modality | System needs to ingest problems |
| S3  | Constraint interface typed logic\|semantic | Verification confidence depends on type |
| S4  | Verification function VERIFY$(I, O, C) \to (\text{verdict}, \sigma)$ | Can't confirm outputs satisfy constraints |
| S5  | Knowledge Graph with typed nodes and edges | Without structured persistence of what worked, failed, and why — no learning, no reuse, every goal starts from scratch |
| S6  | Self-model | Must know own capabilities, costs, state to select resolvers and stay grounded |

### Behavioral Axioms (what must always happen)

| #   | Axiom | Why It Can't Be Emergent |
|-----|-------|--------------------------|
| B1  | **Per-node budget metering** — every resolver call is estimated + gated + recorded; parent allocates to children; on `pre_check` denial route to Step 11 (HIRE / human budget decision) | Every task has finite resources. Without per-node tracking, one subtask can silently eat the entire budget. |
| B2  | **Temporal metering** — elapsed time tracked per-node; deadlines enforced as logic-typed constraints | Time is finite and universal. A system that doesn't track its own time cannot make cost/time tradeoffs. |
| B3  | **Dependency ordering** — subgoals form a DAG; blocked nodes cannot execute; topological sort determines queue priority | Without ordering, the system attempts tasks it can't yet do. Correctness requirement, not optimization. |
| B4  | **Lookup-before-build** — mandatory cache check before BUILD (registry capability match + KG history); build only on cache miss | Storage without retrieval = no reuse. The growth promise is meaningless without actual lookup. |
| B5  | **Non-negotiable operation sequence** — identify → select → resolve → verify → decompose/leaf. Enforcer rejects skips. | This is what makes the system *this system* rather than ad-hoc LLM reasoning. |
| B6  | **Monotonic constraint propagation** — inherited constraints are excluded from eligible sets; children treat inherited constraints as assumed truths during their solve. If an ancestor is reopened, descendants are invalidated and recomputed under the new lineage (no oscillation within a lineage). | Without this, infinite loops. Computation never terminates. |
| B7  | **Confidence gating** — $\sigma_{path} < \tau_{global}$ blocks decomposition; backtrack to weakest ancestor | Without gating, the system expands enormous subtrees under low-confidence parents. All wasted if parent turns out wrong. |
| B8  | **Toolization of edge cases / human work** — any HIRE outcome must register a reusable tool/verifier/rule; one-off human patches are disallowed; dedupe by capability request fingerprint | Without compounding, human load scales linearly with goals, breaking "infinite scalability". |

### What's NOT Axiomatic (Emergent)

Everything domain-specific:

* Domain ontologies (thermal, structural, financial, legal)
* Physical resolvers
* Domain-specific decomposers, verifiers, feedback generators
* Sophisticated orchestration strategies
* Cross-domain coupling detection
* The full M0-M8 module stack beyond the kernel minimum

These grow via the TRY → BUILD → HIRE loop. The axioms define the rails; the growth engine fills in the content.

## 8. Summary: What Gets Built

| Component | Type | Lines of Code (estimate) | Dependencies |
|-----------|------|--------------------------|--------------|
| SQLite schema + migrations | Infra | \~100                    | sqlite3      |
| Node State Machine | Core | \~150                    | —            |
| Dispatch Loop | Core | \~300                    | State Machine, Budget, KG, Logger |
| Budget Enforcer | Core | \~150                    | SQLite       |
| Dependency Resolver | Core | \~80                     | SQLite (topological sort) |
| Knowledge Graph CRUD + Queries | Core | \~250                    | SQLite       |
| Resolution Logger | Core | \~100                    | SQLite, KG   |
| Tiered Logger | Core | \~80                     | SQLite       |
| Registry CRUD + Query | Core | \~120                    | SQLite       |
| Lookup-Before-Build Gate | Core | \~50                     | KG, Registry |
| Claude Code adapter | Integration | \~150                    | Claude API   |
| Codex Tools adapter | Integration | \~100                    | subprocess   |
| Human Interface (CLI) | Integration | \~100                    | stdin/stdout |
| Context Injection (KG → prompt) | Integration | \~150                    | KG, templates |
| **Total** |      | **\~1,900**              |              |

The kernel is \~1,900 lines of code. Everything else — every domain resolver, every ontology, every specialized verifier — gets built through the system itself.


# Karan's v1 Implementation Notes

## Scope

Build the minimum system required to run the 3-stage reasoning loop (Constraint Extraction, Task Decomposition, Approach Survey) on non-trivial goals, with enough observability to verify trace quality before adding execution capabilities.

Hard runtime cap: 10 minutes wallclock per goal when we are just testing its planning abilities (we can vary this, but just avoid it running indefinitely).


---

## What to Test First

The first iteration validates that the system can:


1. Extract both explicit and implicit constraints from an underspecified goal, including removal consequences and open questions
2. Decompose a goal into a task DAG with cost/time estimates, dependency edges, parallelization windows, and a critical path. We can relax forced decomposition to see if it emerges on its own, given a broader set of verifiers and the search infrastructure in a second test.
3. Compute constraint feasibility (SAT / UNSAT / TIGHT) across the full task DAG via budget rollup
4. Detect constraint walls — tasks or task clusters whose cost/time estimates push a constraint into UNSAT
5. Trigger an approach survey for high-uncertainty or constraint-violating tasks, producing multiple alternative approaches with cost/time/confidence estimates and known-method vs judgment-required splits
6. Select a repair path and produce a revised budget that flips violated constraints back to SAT or TIGHT

This corresponds to the SOLVE loop Steps 1-9 in the specification, with decomposition (Step 8) and approach survey (Steps 9-10) included. BUILD (Step 10), HIRE (Step 11), global consistency (Step 12), and artifact assembly (Step 13) are deferred.


---

## Mapping to Specification Sections

| Spec Section | Included in First Iteration | Notes |
|--------------|-----------------------------|-------|
| SQLite Schema (Section 3) | Partial                     | 5 of 11 tables: `nodes`, `constraints`, `resolution_attempts`, `node_dependencies`, `logs`. Remaining tables (`kg_nodes`, `kg_edges`, `budget_ledger`, `capability_requests`, `resolvers`, `artifacts`) deferred. |
| Node State Machine (Section 4) | Yes                         | States: `pending`, `identifying_constraints`, `selecting_constraints`, `selecting_resolver`, `resolving`, `verifying`, `decomposing`, `resolved`, `failed`. States `gated` and `infeasible` deferred. |
| Dispatch Loop (Section 5) | Partial                     | Steps 1-9 only. No BUILD, HIRE, global consistency check, or final assembly. |
| Budget Enforcer (Section 6) | Partial                     | Wallclock hard cap (10 min). Per-node cost/time estimation and feasibility rollup (SAT/UNSAT/TIGHT). Full allocation/reallocation/reserve mechanism deferred. |
| Dependency Resolver (Section 7) | Yes                         | Topological sort, parallel wave identification, critical path computation. |
| Knowledge Graph (Section 8) | No                          | No value for initial verification runs. Resolution logger provides sufficient tracing. Promote to KG after validating the reasoning loop. |
| Resolution Logger (Section 9) | Yes                         | Full structured JSON per attempt: step, inputs, outputs, rationale, verdict, cost, time. |
| Tiered Logger (Section 10) | Yes                         | All 3 tiers from the start. Tier 1: human-readable flow. Tier 2: structured decision JSON. Tier 3: raw I/O (prompts/responses). |
| Registry (Section 11) | Minimal                     | Hardcode 2 boot resolvers (`claude-code-0`, `codex-tools-0`). No dynamic registration or capability matching. |
| Lookup-Before-Build Gate (Section 12) | No                          | No BUILD step in first iteration. |
| Claude Code Adapter (Section 13) | Yes                         | 6 prompt types: constraint extraction, task decomposition, constraint feasibility check, approach survey, approach selection/repair, verification. |
| Codex Tools Adapter (Section 14) | No                          | No execution in first iteration. |
| Human Interface (Section 15) | Minimal                     | Goal input only. No HIRE escalation. |
| Context Injection (Section 16) | No                          | No KG to inject from. |
| Confidence Model (Section 17) | Partial                     | Per-constraint `sigma_i` (logic=1.0 via Z3, semantic via LLM). Per-node `sigma_v = min(sigma_i)`. Path and system-level propagation deferred until subtree depth > 1. |


---

## Verification Examples

Each example must have constraint tension that forces decomposition, violation detection, and restructuring. Outputs are structured JSON, verifiable via assertions. These should be quite difficult, but not impossible. The point is to have the system thoroughly introspect and adapt to the constraints it is given before it attempts a solution.

### Example 1: Autonomous SWE Agent

**Goal:** "Build an autonomous SWE agent achieving 80% on SWE-bench Lite, within 24h, for &lt;$500 compute."

**Expected behavior:**

* Extracts 5 constraints (3 explicit: score, time, cost; 2 implicit: autonomy, no data leakage)
* Decomposes into \~9 tasks across research/build/evaluation types with dependency DAG
* Budget rollup: mid-estimate total exceeds $500 cap, c3 flagged UNSAT
* Approach survey triggered for high-cost evaluation tasks
* Repair path found via scope narrowing and substitution (deterministic tooling replaces LLM calls)
* Revised budget brings c3 to SAT

**Digital verification:**

* Implicit constraints present with removal consequences
* Dependency edges form a valid DAG (no cycles, all references resolve)
* Cost rollup arithmetic correct (sum of task mid-estimates matches reported total)
* Constraint feasibility status consistent with rollup vs threshold comparison
* At least one surveyed approach has lower cost than the original for each UNSAT-triggering task
* Revised total &lt; budget constraint threshold

**Reference trace:** Existing [GPT-5.2 decomposition outputs](https://graftonsciences.getoutline.com/doc/layer-0-seed-synthesis-WCdzCjhhbu) serves as a baseline for expected decomposition quality.

### Example 2: Quantitative Trading Strategy

**Goal:** "Build and backtest a momentum-based equities trading strategy achieving >12% annualized return with &lt;15% max drawdown, using only free data sources (Yahoo Finance, FRED), deployable to Alpaca paper trading, within 16h dev time, for &lt;$100 total compute."

**Key terms:**

* *Annualized return (>12%):* The yearly percentage gain, normalized. If $100 invested becomes $112+ after one year, the annualized return is 12%. Even if the strategy runs for 6 months, the return is scaled to a full-year equivalent. 12% is a meaningful bar because the S&P 500 historical average is \~10%, so the strategy must beat passive index investing.
* *Max drawdown (&lt;15%):* The largest peak-to-trough drop during the backtest period. If the portfolio goes from $100 to $85, that is a 15% drawdown. It measures worst-case pain — how much would be lost before the strategy recovers. This constraint creates tension with the return target: aggressive strategies that hit 12%+ often have 20-30% drawdowns, so the system must find something that performs well without excessive risk.
* *Momentum-based:* Buy assets that have been rising, sell those declining, on the assumption that trends persist. The system needs to determine lookback windows, rebalancing frequency, and position sizing — all of which affect both return and drawdown.

**Expected** **behavior:**

* Extracts explicit constraints: return target, drawdown limit, data source restriction, deployment target, time cap, compute cap
* Extracts implicit constraints: no lookahead bias, out-of-sample validation required, survivorship bias handling, transaction cost modeling
* Decomposes into tasks: data acquisition/cleaning, strategy design, backtesting framework, parameter optimization, paper trading integration, compliance/validation
* Budget rollup: if LLM-in-the-loop strategy iteration across parameter space, compute exceeds $100
* Approach survey: deterministic grid search or walk-forward optimization vs LLM-guided exploration
* Repair path: substitute deterministic backtesting for LLM-assisted parameter search

**Digital verification:**

* Lookahead bias constraint present (critical for backtesting validity)
* Data tasks depend on nothing; strategy tasks depend on data; backtesting depends on both; optimization depends on backtesting
* Compute cost of LLM-heavy approach exceeds $100; at least one alternative does not
* Critical path arithmetic consistent with dependency edges and time estimates

### Example 3: Document Classification Deployment

**Goal:** "Deploy a document classification model achieving >88% F1 on a 50-class subset of RVL-CDIP (document images: invoices, contracts, forms, etc.) to a REST endpoint handling 10,000 requests/week with p99 latency &lt;500ms, hosted for &lt;$50/month, for <$50 total compute including training, within 8h dev time."

**Expected behavior:**

* Extracts explicit constraints: F1 score, class count, throughput (10k req/day), latency, hosting cost, training compute, dev time
* Extracts implicit constraints: model must generalize (not memorize training set), endpoint must handle burst traffic (not just average throughput), model must fit within hosting tier memory limits, document images are large (scanned pages, not thumbnails) so inference cost per request is non-trivial
* Decomposes into tasks: model selection/training, optimization (quantization/pruning/distillation), API framework, deployment, load testing
* Budget wall: large vision model (ViT-L, etc.) hits F1 target but blows hosting cost and may exceed $50 training compute; small model fits hosting but misses 88% F1 on 50 classes; fine-tuning a mid-size model from scratch risks exceeding the compute cap
* Approach survey: pretrained document model (DiT, LayoutLM) + quantization + CPU deployment vs training from scratch + GPU hosting vs smaller model with aggressive augmentation
* Repair path: use pretrained document-specialized weights (skip most training cost), quantize to INT8, deploy on CPU with ONNX Runtime

**Digital verification:**

* Hosting cost constraint checked separately from training compute constraint
* At least one approach satisfies both $50 compute and $50/month hosting simultaneously
* Training task cost estimate reflects model size and hardware choice
* Latency constraint at 10,000 req/week with burst patterns creates tension with CPU-only deployment (system should acknowledge this as TIGHT, not ignore it)

  \

### Assertion Framework

For all three examples, the following checks apply to the structured JSON output:

```
1. Constraint completeness
   - At least N explicit constraints extracted (N = number in goal statement)
   - At least 1 implicit constraint with removal consequence

2. Decomposition validity
   - All dependency references resolve to existing task IDs
   - No cycles in dependency graph
   - At least one task has no dependencies (entry point)
   - At least one task depends on all others transitively (exit point)

3. Budget arithmetic
   - Sum of task mid-cost estimates equals reported total
   - Cumulative waterfall values are monotonically increasing
   - Remaining budget values are monotonically decreasing
   - Constraint feasibility status matches threshold comparison

4. Approach survey triggers
   - Surveyed tasks have either confidence < 0.3 or contribute to an UNSAT constraint
   - Each survey produces >= 2 distinct approaches
   - At least one approach per survey has lower cost than original

5. Repair effectiveness
   - Revised total satisfies the previously violated constraint
   - Revised task estimates are individually plausible (no negative costs, no zero-time builds)

6. Critical path
   - Reported critical path is the longest path through the dependency DAG by time
   - Critical path duration <= time constraint (or flagged as TIGHT/UNSAT)
```


---

## Reusable Components from Existing Codebase (`graftonlab`)

### Direct Reuse

**LLM Client** (`src/agents/llm.py`)

* `LLMClient` with `complete_json(prompt, system_prompt, schema)` for structured output via Pydantic models
* Token logging and error reporting on empty responses
* `MockLLMClient` for testing without API calls
* Provider protocol (`src/cad/intent_decomposition/llm/base.py`) already includes `ANTHROPIC` enum value
* **Modification needed:** Swap OpenAI SDK calls to Anthropic SDK; update default model

**Z3 Constraint Solver** (`src/verification/semantic/z3_solver.py`)

* `solve_constraints()` returning `SolveResult` with `SAT/UNSAT/UNKNOWN` status and unsat core
* Per-constraint tracking via `assert_and_track()` — maps to per-constraint `sigma_i = 1.0` for logic constraints
* Scoped solving via `solve_constraints_scoped()` — applicable to per-node constraint feasibility
* 30-second timeout preconfigured
* **Modification needed:** Remap `Constraint` dataclass from engineering parameters (min/max/tolerance) to expression-based constraints (cost &lt;= threshold, time &lt;= cap). Z3 plumbing unchanged.

**Constraint Extraction** (`src/verification/semantic/constraint_extractor.py`)

* Extraction pipeline producing typed constraints with provenance
* `ExtractionResult` with `coverage: float` and `warnings` — maps to constraint completeness checking
* Tolerance parsing and unit canonicalization patterns
* **Modification needed:** Replace engineering domain (Pint units, geometric tolerances) with budget/time/performance domains

**Uncertainty Propagation** (`src/uncertainty/propagation.py`)

* `compute_confidence()` with weakest-link aggregation through a graph
* Matches the kernel spec's `sigma_v = min(sigma_i)` model
* **Modification needed:** Adapt from hypergraph traversal to task DAG traversal; add path-product computation for `sigma_path`

### Pattern Reuse (adapt structure, replace domain logic)

**Agent Orchestration** (`src/agents/orchestrator.py`, `src/agents/base.py`)

* `Trigger` / `TriggerType` system for event-driven routing — maps to node state transitions
* `HypergraphMutation` for atomic graph changes — maps to node/constraint creation
* `AgentResult.next_triggers` for step sequencing — maps to dispatch loop's `next_action()`
* `route_trigger()` for dispatching to handlers — maps to dispatch loop routing
* **Modification needed:** Replace CAD-specific trigger types and agents with kernel stages (constraint extraction, decomposition, approach survey, verification)

**Observability Traces** (`src/cad/intent_decomposition/observability/`)

* `IterationTrace`, `DecompositionTrace` structured trace types
* Trace collector and JSON storage
* Post-hoc analysis via `TraceAnalyzer`
* **Modification needed:** Generalize trace types from CAD operations to kernel steps; add Tier 1/2/3 classification

**Feedback Loop** (`src/cad/intent_decomposition/execution/feedback_loop.py`)

* `FeedbackLoopManager`: execute, check, analyze error, generate fix, re-execute
* `ErrorAnalyzer` with error categorization
* Maps to SOLVE loop Steps 5-9 (resolve, verify, UNSAT, feedback, retry)
* **Modification needed:** Replace CAD execution errors with constraint violations; replace code generation with approach restructuring

**Budget Tracking** (`src/hypergraph/models.py`)

* `Budget` node with `total/consumed/remaining` per resource type
* `CONSUMES` edge type for tracking resource usage
* **Modification needed:** Add per-node allocation, cost/time estimates (low/mid/high ranges), feasibility rollup computation, waterfall tracking, cost wall detection

**Tool Gateway** (`src/tools/gateway/`)

* Subprocess isolation for executing untrusted code
* Relevant for phase 2 (sandboxed execution), not first iteration
* **Retain for later use**

### Not Reusable

| Existing Component | Reason |
|--------------------|--------|
| Hypergraph JSON store (`src/hypergraph/store.py`) | Kernel requires relational queries (budget rollup across DAG, topological sort, constraint feasibility joins). JSON store cannot support these efficiently. SQLite required. |
| CAD-specific agents (GRS refinement, contract extraction, mechanical verification) | Domain-specific to engineering design. Logic is not transferable. |
| DSPy optimization pipeline (`src/cad/intent_decomposition/dspy_optimization/`) | Prompt optimization is a future concern. First iteration uses direct prompting. |
| Intent cache / mem0 integration (`src/memory/intent_cache.py`) | Caching and similarity search are Knowledge Graph concerns, deferred to after reasoning loop validation. |
| CadQuery execution and API retrieval | Domain-specific. |


---

## Build Order


1. **SQLite schema** (5 tables) + **Tiered logger** — storage and observability foundation
2. **Node state machine** + **Dependency resolver** — state discipline and DAG ordering
3. **Claude Code adapter** (6 prompt types), reusing `LLMClient` — the reasoning engine
4. **Z3 verification glue**, reusing `z3_solver.py` — deterministic constraint checking for logic-typed constraints
5. **Budget enforcer** (cost/time rollup, feasibility status), adapting `Budget` node — constraint wall detection
6. **Dispatch loop** (Stages 1-3), adapting orchestrator trigger/routing pattern — wire everything together
7. **Resolution logger**, adapting observability traces — full attempt history
8. **Run 3 examples**, check traces, run assertion framework
9. **Decide next:** subtree decomposition depth > 1, or sandboxed execution

Estimated new code: \~1,300 LOC, reduced from \~1,900 in the full specification due to reuse of LLM adapter, Z3 solver, confidence propagation, and observability patterns.