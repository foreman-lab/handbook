---
status: Draft v1 — supersedes ADR 0002 §4.4 subgraph model; pending operator sign-off
date: 2026-04-18
deciders: Operator (xuzhijie), after composite/Send spike + Codex + Copilot debate rounds 5–6
supersedes: none
superseded_by: none
amends:
  - docs/adr/0002-langgraph-1x-adoption-and-deviations.md §4.4 Scenario 3 (subgraph + `checkpoint_ns = childNodeId` query shape)
  - docs/phase-1-solo/architecture.md §4.4 Scenario 3 (composite parent with 2 children)
related:
  - docs/adr/0003-state-machine-scope-and-mcp-surface.md (state-machine-as-storage doctrine; this ADR is a natural consequence)
---

# ADR 0004: Composite nodes as independent threads (supersedes subgraph model)

## Status and scope

Draft pending operator sign-off. ADR 0002 (approved 2026-04-18) committed to running a **composite/Send spike** at M3 planning to verify whether LangGraph subgraph children are queryable at a semantic `checkpoint_ns = childNodeId` handle. ADR 0002:121 said explicitly: _"If not, a second ADR amends §4.4."_

The spike ran (artifact at `packages/solo/scripts/composite-send-spike.ts`). The hoped query shape **does not work** under LangGraph 1.2.9. This ADR is that second ADR.

## Context

### What ADR 0002 §4.4 Scenario 3 assumed

Architecture.md §4.4 Scenario 3, as currently written:

```
initialize (kind=composite) ─▶ init node (parent, checkpoint_ns=parentId)
plan (children: [c1, c2])   ─▶ plan node returns Send([ initialize(c1), initialize(c2) ])

LangGraph runs subgraphs in parallel:
  ├─ Child c1: full init → plan → work → evaluate → pass  (checkpoint_ns=c1)
  └─ Child c2: full init → plan → work → evaluate → pass  (checkpoint_ns=c2)

Parent evaluate node:
  • graph.getState({thread_id:projectId, checkpoint_ns:"c1"})  → pass
  • graph.getState({thread_id:projectId, checkpoint_ns:"c2"})  → pass
```

The assumption: dispatching children via LangGraph `Send` produces checkpoints at a stable, operator-chosen `checkpoint_ns` value (e.g., `"c1"`). ADR 0002 flagged this as "TO-VERIFY" and required a spike.

### What the spike proved

Run on 2026-04-18 against LangGraph 1.2.9 + `@langchain/langgraph-checkpoint-sqlite` 1.0.1:

- `getState({thread_id, checkpoint_ns: 'c1'})` returns **empty** — the hoped query shape does not work.
- LangGraph auto-assigns subgraph checkpoint namespaces as `<nodeName>:<task-uuid>`, e.g., `child_run:e85adf49-02c5-5570-b173-b81bb09a57f6`. These are **ephemeral** (task-scoped, not node-identity-scoped).
- Children DO save state correctly — retrievable via the auto-assigned ns, but not via any stable semantic handle.
- This means Scenario 3's parent-evaluate query pattern cannot be implemented as written.

Options for recovering: (X) scan `checkpointer.list` post-invoke and map auto-ns to childId by reading state values (fragile); (Y) drop subgraphs entirely — each child becomes its own top-level thread (clean); (Z) hybrid — Send dispatches independent thread invocations (adds indirection without benefit).

### The architectural reframe that settles the choice

The operator observed during debate (verbatim):

> "because for normal kernel job maybe is only one static graph, however foreman is interactive state machine, and it's lazy planning so it can resolve more complicated jobs"

Unpacking:

- **Normal LangGraph usage** is _static_: one `graph.invoke` runs a known fanout topology to completion. `Send` is designed for this case — dispatch N parallel paths within a single invocation.
- **Foreman is interactive**: each `step` signal drives exactly one transition (D-M2-16). The graph is invoked many times across a session. Composite children's full lifecycles (`init → plan → work → eval`) are _not_ contained within the parent's `plan`-signal invocation — they span many later signals.
- **Foreman uses lazy planning**: a composite parent does not know its children at init time. Children are discovered when the agent's response to the `plan` prompt arrives and returns a `children: Brief[]` list. Children may themselves be composite, cascading the pattern recursively.

LangGraph `Send` is designed for in-invocation fanout where children complete before the dispatching invoke returns. That is a **structural mismatch** with foreman's signal-driven, cross-invocation, lazily-discovered composite lifecycle. Subgraph dispatch was the wrong primitive, not just a query-shape bug.

## Decision

**Composite children are modeled as independent top-level threads, not as LangGraph subgraphs.**

Each foreman node (terminal or composite) gets its own thread in the checkpointer:

- `thread_id = ${projectId}:${nodeId}` (ADR 0002 composite thread_id shape, unchanged)
- `checkpoint_ns = ''` (always empty; non-empty was an ADR 0002 open question that this ADR closes by saying: we don't use it)

Composite structure lives in parent state, not in graph topology:

- Parent's `plan` channel carries `{kind: 'composite', children: Brief[]}`
- Parent's `plan.children` is an array of child-node briefs (each carrying its own `nodeId`, `kind`, `skill`, `goal`)
- The LangGraph graph definition does not know about parent/child relationships. Composite is a _state shape_, not a _graph topology_ concept.

### Mental model: each node is a ticket

A layman framing that captures the invariant:

Think of each foreman node as a **Jira ticket**. Each ticket has its own ID, its own activity log (journal), its own state transitions (init → plan → work → eval), and its own outcome (pass / capped / etc.). You never have "half a ticket" — a ticket is one atomic trackable unit of work.

A `kind: composite` ticket is an **epic**. When you plan the epic, you list sub-tickets instead of outlining how to do the work yourself. Each sub-ticket is its own ticket with its own ID and history. Some sub-tickets may themselves be epics, cascading recursively. The epic closes when all its sub-tickets close.

This ADR codifies that per-ticket isolation: **each ticket gets its own file.** Sub-tickets are not jumbled into the epic's history.

### Atomic-action invariant

Each node's full lifecycle is one semantically atomic unit of work:

- **Signal-level atomicity (strong):** one `step(...)` call = one `graph.invoke` = one transition = one journal event = one SQLite checkpoint row. Crash mid-signal leaves no partial state (checkpointer rolls back).
- **Node-level atomicity (semantic):** the full `init → plan → work → eval → outcome` sequence on one thread is a coherent saga with a complete journal. Not DB-atomic (spans many signals), but one thread's journal is a self-contained audit story.

Per-node-thread storage (this ADR's model) preserves this invariant. Subgraph-based composition (the rejected model) would tangle children's histories into the parent's thread, breaking per-unit audit completeness.

### Concrete mechanics

Lazy planning cascade for a composite tree `root → { c1 (terminal), c2 (composite) → { gk1, gk2 (both terminal) } }`:

1. **Init root**: `step(type: 'init', nodeId: 'root', payload: {kind: 'composite', brief: {...}})`.
   - Foreman creates thread `project:root`. Set-once reducers lock `nodeId`, `kind`, `brief`.
2. **Plan root**: `step(type: 'plan', nodeId: 'root', payload: {kind: 'composite', children: [briefC1, briefC2]})`.
   - Foreman stores children list in root's `plan` channel (replace reducer).
   - **This is the lazy-expansion moment for root.** Before this, children didn't exist.
3. **Wrapper reads root's plan**, sees `children: [...]`, sends `step(type: 'init', ...)` for each child.
   - `step(type: 'init', nodeId: 'c1', payload: {kind: 'terminal', brief: briefC1})` → thread `project:c1` created
   - `step(type: 'init', nodeId: 'c2', payload: {kind: 'composite', brief: briefC2})` → thread `project:c2` created
4. **c1 runs normally on its thread**: `plan → work → eval → pass`. (Plain terminal flow.)
5. **c2 cascades the same pattern**: `plan` signal returns `children: [briefGK1, briefGK2]`; wrapper spawns threads for gk1 and gk2.
6. **Leaf terminals run**: gk1, gk2 each go `plan → work → eval → pass` on their own threads.
7. **c2 consolidates** (Option B, v0.7 amendment): once gk1 and gk2 both reach terminal pass, the agent reads their outputs via `foreman__status` and synthesizes them into a consolidated `WorkReport` using `briefC2.skill.methodology.work` as the consolidation prompt (D-18 skill injection). The agent then sends `step(type: 'work', nodeId: 'c2', payload: {report, toolCalls: []})`.
   - **Handler readiness pre-check** (`handlers/step.ts`): sees `state.kind === 'composite'`, reads children via `graph.getState(checkpointConfig(projectId, childNodeId))` for each of `state.plan.children`; if any child lacks evaluation, rejects with `COMPOSITE_CHILD_NOT_READY`.
   - With all children evaluated, handler invokes the graph with the consolidated `WorkReport`. c2's `work` + `evaluate` nodes run **kind-agnostic** (same code as terminal): write `work`, then assess.
   - All pass → c2 passes.
8. **root consolidates**: once c1 and c2 both pass, the agent synthesizes a root-level consolidation and sends `step(type: 'work', nodeId: 'root')`. Same handler pre-check + kind-agnostic node execution as step 7.

### Composite parent work phase = consolidation (Option B, 2026-04-18)

Composite parents **do** have a `work` phase — the agent's work report IS the consolidation of children's outputs. Composite work is not "do the work"; it's "synthesize the work that already happened on the children's threads." The skill's `methodology.work` text provides the consolidation prompt (a composite skill has a different prompt than a terminal skill); foreman serves the prompt via D-18 skill injection but does not interpret it.

Readiness gate lives in `handlers/step.ts` (not in the graph): the handler reads each child thread via `graph.getState(checkpointConfig(projectId, childNodeId))` before permitting the parent's work signal. If any child has no evaluation, the handler returns `COMPOSITE_CHILD_NOT_READY`. Once all children are evaluated, the handler invokes the graph normally — the `work` and `evaluate` nodes are kind-agnostic.

**Hexagonal rationale for handler-level aggregation** (not a graph node, not a new port):

- **Sequence-validity, not transition.** The composite readiness check rejects BEFORE any channel write or journal append. Per foundations D-21 (amended in this round), read-only cross-thread access for sequence-validity is explicitly permitted.
- **Reuses existing abstraction.** `graph.getState(checkpointConfig(...))` is already used for same-thread reads at line 103 of step.ts; reading a sibling thread is the same API with a differently-keyed config. D-20's "narrow port discipline — ports grow only when real variation exists" is honored (no multi-store variation at Phase 1).
- **FSM simplicity preserved.** Adding a `n_evaluate_composite` sibling node (a rejected alternative) would have meant a 5th node + parent-only edge — architectural weight for no runtime benefit. The 4-node graph from ADR 0001 §1.1 stays intact.

**Superseded:** an earlier draft of this ADR (pre-Option B) said "composite parents do not have a work phase; `step(type: 'work', nodeId: composite)` is rejected." That was wrong. Composite work is the consolidation phase; the readiness check rejects only when children are not yet evaluated.

### Tree depth

Unbounded. Every level is just more threads at the flat `projectId:nodeId` namespace. The `nodeId` allowlist regex (`/^[A-Za-z0-9._-]{1,256}$/`) rejects path separators (`/`, `\`, `:`), so hierarchical paths in the ID itself are disallowed. Tree structure lives in each parent's `plan.children` state field, not in the thread_id shape.

### Aggregation is wrapper-driven

Foreman does not auto-fire a parent's evaluate when its last child reaches terminal state. Per ADR 0003's state-machine-storage doctrine, foreman records events; the wrapper orchestrates transitions. The wrapper reads child states (via `getState` on each child thread), determines readiness, and explicitly sends `step(type: 'eval', nodeId: parent)` when ready.

Sequence-validity may optionally reject premature parent evaluate — e.g., if any child is still non-terminal — as a safety check. This is a state-precondition validation, not workflow orchestration.

## Consequences

### Immediate (doc amendments, before M3 code)

- **Update `docs/adr/0002-*.md`** — §4.4 Scenario 3 paragraph gains a pointer: _"SUPERSEDED by ADR 0004; see that ADR for the per-thread composite model."_ Keep the prior text for historical context.
- **Update `docs/phase-1-solo/architecture.md` §4.4 Scenario 3** — rewrite the flow diagram to show per-thread child spawning + wrapper-driven aggregation. Remove `Send` + `checkpoint_ns = childId` references.
- **Update M3 development proposal (forthcoming)** — composite handler section reflects this model; no subgraph/Send machinery to build.

### M3 code consequences (Option B, v0.7)

- **`packages/solo/src/graph/nodes/evaluate.ts`** — **kind-agnostic** (no composite branch). Assesses the `WorkReport` in state; same code path for terminal and composite. Adds `interrupt()` wiring for block outcomes (D-M3-8); kind-awareness lives in the handler, not the node.
- **`packages/solo/src/handlers/step.ts`** (new at M3) —
  - On `plan` signal with `kind: 'composite'`, store children in the plan channel (replace reducer); do NOT spawn children (wrapper's responsibility).
  - On `work` signal when parent is composite: readiness pre-check reads each `plan.children[i].nodeId` via `graph.getState(checkpointConfig(projectId, childNodeId))`; rejects `COMPOSITE_CHILD_NOT_READY` if any child has `evaluation === null`. Otherwise invokes graph with the agent's consolidated `WorkReport`.
  - On `resume` signal: `new Command({resume: payload})` only (per LangGraph 1.x docs — invoke-input Commands accept only `resume`). Fail-routing lives inside evaluate node as node-return `Command({update, goto: END})`.
- **`packages/solo/src/graph/build.ts`** — no `Send` additions. No subgraph node. Dispatch-on-signal topology unchanged. `{ends: [END]}` on `n_evaluate.addNode` declares END as a valid dynamic goto target for the fail-routing Command.
- **Sequence-validity checks** — composite `work` gated on children-readiness per above. Composite `eval` has no extra guards at M3 (same as terminal — the work-phase pre-check is the readiness gate; eval just assesses whatever consolidation landed).

### Phase 2a+ implications

- **Per-thread mutex.** Each node has its own thread → own SQLite write lock → no cross-child contention. Multi-agent concurrency on different children scales cleanly.
- **Tenant namespacing.** The three-segment composite `${tenantId}:${projectId}:${nodeId}` from ADR 0002 Consequences applies identically; every node (terminal or composite) is a thread under the tenant's namespace.
- **No special workflow-engine semantics.** The wrapper (daemon, team service) owns orchestration. Foreman remains pure storage per ADR 0003.

### Composite parent retry — non-cascading (added M3 planning round 9)

When a composite parent reaches `capped` state, the operator's `step(type='retry', nodeId=parent)` re-enters the parent at its `plan` node with retry-count reset on the parent's thread **only**. Retry does **not** cascade to children:

- Children's threads retain their terminal outcomes (pass / failed / capped). Their state is preserved as historical context.
- The parent's new plan signal may return a different `children: Brief[]` list — operator/agent decides whether to reuse prior child nodeIds (incorporating their journals as context) or spawn fresh child nodeIds (clean slate).
- If the operator wants a specific child retried, they send `step(type='retry', nodeId=<child>)` directly on that child's thread.

This preserves the atomic-action invariant: each thread's retry is scoped to that thread. Cross-thread cascades would tangle histories and break the "one thread = one node = one atomic unit" rule.

### Grandchild ownership (added M3 planning round 9)

When a composite child's `plan` signal returns grandchildren, the **wrapper** issues `step(type='initialize', nodeId=<grandchild>)` for each. Neither the parent's handler nor foreman itself spawns grandchild threads. This follows the same rule as root-level composite init: foreman records state; the wrapper orchestrates. The cascade is recursive but always wrapper-driven at every level.

## Alternatives Considered

### Alt 1 (rejected): Keep subgraphs with `checkpoint_ns = childNodeId`

ADR 0002's original hope. Empirically refuted by the spike — LangGraph 1.2.9 auto-assigns subgraph ns as `<nodeName>:<task-uuid>`, not at operator-chosen handles. No query shape for stable child retrieval exists.

### Alt 2 (rejected): Subgraphs + scan/map discovery

Post-invoke iterate `checkpointer.list`, filter ns matching the subgraph pattern, read child state, extract `childId` from values, build an in-memory map keyed by childId. Complexity: operator cannot stably address a specific child; requires scan on every access; children's histories remain in the parent's thread (audit-tangling). Fragile.

### Alt 3 (rejected): Hybrid — Send dispatches independent invocations

Parent's `plan` node uses conditional edges + `Send` to fan out, but each Send target triggers an independent `graph.invoke` on a fresh thread (rather than a subgraph). Children still get their own threads. Adds an indirection layer (Send-as-launcher) without clear benefit over Alt 4's direct per-node-thread model.

### Alt 4 (adopted): Per-node independent threads

Each node is its own thread. Wrapper orchestrates initialization and aggregation. Composite is a state-shape pattern, not a topology concept. Clean audit, clean mutex, clean parallelism, forward-compatible with Phase 2a+ multi-tenant + multi-user.

## References

- ADR 0002 §4.4 Scenario 3 ("TO-VERIFY" query shape; now superseded by this ADR)
- ADR 0002 line 121 ("composite/Send spike is required at M3 planning")
- ADR 0003 §1 (state-machine storage doctrine; wrapper owns orchestration; this ADR is the composite-specific consequence)
- Spike artifact: `packages/solo/scripts/composite-send-spike.ts`
- Spike verdict: see Appendix A of this ADR

## Appendix A — Composite/Send spike results (2026-04-18)

Method: in-process LangGraph test. Parent `StateGraph` with `addConditionalEdges` returning `Send` instances fans out to a child subgraph. SqliteSaver backed by tmp SQLite file. Parent invoked with `children: ['c1', 'c2']`.

Results:

- **`getState(thread_id, checkpoint_ns: 'c1')`** → returns empty `values: {}`. ADR 0002 Scenario 3's hoped query shape fails.
- **Distinct `checkpoint_ns` on the parent's thread**: `["", "child_run:e85adf49-02c5-5570-b173-b81bb09a57f6", "child_run:fa1ae719-5cd5-5be1-8137-394c0858065c"]`. Two subgraph namespaces auto-assigned (one per Send dispatch); the format is `<nodeName>:<task-uuid>`.
- **Querying auto-ns works**: `getState(thread_id, checkpoint_ns: 'child_run:...')` returns `{childId: 'c2', result: 'done-by-c2'}`. Children DO persist state; just not at semantic handles.
- **Verdict**: ADR 0002's query shape invalidated. Children must be modeled differently. This ADR adopts Alt 4 (per-node threads).

Artifact kept at `packages/solo/scripts/composite-send-spike.ts` for reproducibility and future regression-check.
