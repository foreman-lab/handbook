# Foreman Architecture

**Status:** Locked — Phase 1 baseline (2026-04-17; reviewed by Copilot twice; amended by [ADR 0001](adr/0001-architectural-style-declaration.md) to declare architectural style in §1.1)
**Last updated:** 2026-04-17
**Related:** [`docs/foundations.md`](foundations.md) (principles, security, decisions), [`docs/roadmap.md`](roadmap.md) (phases, tripwires)

This document describes **how foreman is structured across all four tiers** (Solo, Daemon, Team, Enterprise). Principles live in `foundations.md`; product plan lives in `roadmap.md`; this document is the system design that both constrain and that both reference.

---

## 1. System Overview

### 1.1 Architectural Style

Foreman is built on a **hybrid architectural style**. Seven patterns apply across all four tiers. Each pattern earns its place by solving a concrete concern required by the principles (P-1..P-10) or foundational decisions (D-1..D-20); no pattern is adopted as architectural dogma.

| Pattern                                            | How it applies to foreman                                                                                                                                                                                                       | Anchors                        |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------ |
| **Hexagonal (Ports & Adapters)** — narrow scope    | Zero outbound ports at Phase 1. Seams inject as function parameters (clock) and handler internals (token stub). Ports promote only on real variation (Phase 2a candidate: `TokenVendingPort`). Rule: D-20; Phase 1 application: §2.2. | D-17, D-20, §2.2, ADR 0007     |
| **Layered architecture** — strict 3 layers         | `Transport / Handler / Flow` boundaries; import-boundary lint enforces the dependency direction. Each axis of change maps to one directory.                                                                                     | D-17, §2                       |
| **Command bus**                                    | Agent signals are validated commands. The handler layer dispatches them to the flow engine via LangGraph `Command` objects. Agent never sees the bus internals.                                                                 | §3, §11.7                      |
| **Event sourcing**                                 | The `journal` channel is append-only and the authoritative record of what happened. Derived state (retry count, active block, resume target) is computed from journal + graph position, never stored as redundant fields (I-8). | §1.6 (invariants), §5.2, §11.2 |
| **CQRS** (Command/Query Responsibility Separation) | Read (`status`) and write (`step`, `resume`, `token_request`, etc.) are separate signal types with distinct response shapes. Reads never mutate; writes have side effects.                                                      | §3.1, §3.4                     |
| **Finite State Machine**                           | Four LangGraph nodes (`init` → `plan` → `work` → `evaluate`) with conditional edges encode the business protocol. Transitions are checkpointed; replay from any checkpoint is deterministic (P-5).                              | §1.4, §4, D-16                 |
| **Dependency Injection via Composition Root**      | `packages/solo/src/app.ts` (consumed by `cli/bin.ts`) wires the checkpointer, graph, handlers, and MCP server. Post-2026-04-18 domain-blind pivot: no outbound ports at Phase 1 (ADR 0007 + §2.2). Each tier introduces its own composition root (Phase 2a daemon, Phase 3 team) without modifying the core. | §11.7                          |

**Why hybrid, not a single named style?** No single architectural style covers all of foreman's concerns cleanly:

- Hexagonal alone doesn't prescribe a business protocol (we need FSM)
- Layered alone doesn't give us the agent-facing command contract (we need Command Bus)
- Command Bus alone doesn't give us the audit trail (we need Event Sourcing)
- Event Sourcing alone conflates reads and writes (we need CQRS)
- CQRS alone doesn't solve the variation-across-tiers problem (we need Hexagonal + DI)

Each pattern is a tool, chosen because it solves a concrete problem — not because it's fashionable. The **D-20 discipline** (no ceremony ports) generalizes: patterns are adopted where they solve concrete problems, rejected when they add ceremony.

**Anti-patterns explicitly rejected:**

| Anti-pattern                                   | Why rejected                                                                                                                                          |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Microservices**                              | Phase 1 is a single process; tiers extend additively (P-8). Foreman's surface is an MCP server, not a mesh.                                           |
| **Service locator** (hidden global singletons) | Breaks testability; breaks P-5 reproducibility; breaks the composition root pattern.                                                                  |
| **Shared mutable state across handlers**       | Handlers are stateless between signals. All state lives in LangGraph-owned checkpoints. Concurrent safety (§11.5) relies on this.                     |
| **Inheritance over composition**               | No abstract class hierarchies. Composition via interfaces and injection. Domain types are plain data; behavior lives in pure functions and node code. |
| **Open-core feature gating**                   | Rejected at the business-model level (P-10). All features ship open source; monetization is hosted-service moat.                                      |

**When to revisit this declaration:** amending the architectural style (adding a pattern, dropping one, changing the hybrid to a single style) requires an ADR — the drift-prevention rules in foundations.md apply. ADR 0001 is the first; subsequent changes reference it.

### 1.2 What foreman is

Foreman is a **harness** — a process orchestrator for AI coding agents. It owns the state machine, journal, checkpoint/replay, operator approval gates, and signal protocol. It does **not** call LLMs, interpret domain content, or decide how work should be done (P-1).

Three actors, strict boundaries (P-2):

- **Harness** (foreman) — state-machine authority; owns channels + journal, validates signal sequences, dispatches transitions. Sequence-validity gating (e.g., journaled approval required before irreversible `ToolCall` per spec §8.3) is foreman's responsibility; **identity and authentication are wrapper concerns** (see ADR 0003 for the wrapper-owns-auth doctrine)
- **Agent** (AI — Claude Code, Copilot CLI, Codex, Cursor, Aider) — content executor; produces plans, work, evaluations
- **Operator** (human) — judge of irreversible decisions; sets goals, resolves blocks, declares failure

Everything flows through these three boundaries. Crossings are typed (Zod), not conventional.

### 1.3 The execution unit: a Node

A **Node** is a unit of work under orchestration, identified by `nodeId` and bound to an immutable `Brief`. Nodes are either:

- **Terminal** — does the work itself (agent plans, works, evaluates)
- **Composite** — coordinates children (agent plans, delegates via `Send`, evaluates aggregate)

Kind is declared at initialization and never changes (D-19 terminology; I-2 invariant).

### 1.4 The 4-node LangGraph flow

```
                  ┌────┐
     START ──▶ │init│      validate brief, emit init event
                  └─┬──┘
                    ▼
                  ┌────┐      agent proposes plan
                  │plan│      ─ composite: return Send(subgraph, children[])
                  └─┬──┘      ─ terminal: advance to work
                    ▼
                  ┌────┐
                  │work│      agent submits work report
                  └─┬──┘
                    ▼
              ┌──────────┐     preflight scan → agent eval → outcome
              │ evaluate │
              └────┬─────┘
                   │
      ┌────────────┼──────────────┬─────────────┐
      ▼            ▼              ▼             ▼
    pass         retry         block         exhausted
      │            │              │             │
     END       → plan      interrupt()        END
                               │
                        ┌──────┴────────┐
                        ▼               ▼
                    resume:          give_up:
                   origin node        END
```

4 nodes. 4 outbound edges from `evaluate` (pass, retry, block, exhausted). Interrupts pause at node boundaries, not mid-execution.

### 1.5 State channels

Seven channels carry all state (§5 details):

| Channel      | Purpose                                               |
| ------------ | ----------------------------------------------------- |
| `nodeId`     | Immutable node identity                               |
| `kind`       | `"terminal"` or `"composite"`, set at init, immutable |
| `brief`      | The contract handed to this node (includes `playbook`) |
| `plan`       | Latest proposal from agent                            |
| `work`       | Latest work report from agent                         |
| `evaluation` | Latest evaluation from agent                          |
| `journal`    | Append-only event log                                 |

All other state is **derived** from these — retry count, active block, resume target, playbook, children states. No redundant channels.

### 1.6 Key invariants

| #   | Invariant                                           | Enforcement                                                                                                                            |
| --- | --------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| I-1 | Brief is immutable after init                       | Types: `Readonly<Brief>`; no post-init edge writes                                                                                     |
| I-2 | Kind cannot change after init                       | Signal schema: `kind` absent from non-init signals                                                                                     |
| I-3 | Completion requires evaluation                      | No edge from `work` → END; only `evaluate` emits `pass`                                                                                |
| I-4 | Composite completion requires all children complete | Parent `evaluate` polls children via subgraph state API                                                                                |
| I-5 | Security blocks terminal unless operator overrides  | `interrupt()` resumes only via explicit `Command(resume=override)`                                                                     |
| I-6 | Preflight gates evaluation                          | `evaluate` runs preflight scan before agent evaluation                                                                                 |
| I-7 | Evaluator cannot directly assert outcome            | Eval signal carries `findings` + `meetsCriteria`; foreman derives outcome via truth table (§4.2). Agent cannot set `outcome` directly. |
| I-8 | Resume target derived, not stored                   | Only `journal` + graph position determine resume                                                                                       |
| I-9 | Blocked state never resumes to evaluate             | Resume edge targets `init`/`plan`/`work` only                                                                                          |

---

## 2. Component Layers

D-17 (Handler/Transport Separation) defines three strict layers. Each axis of change maps to one directory:

```
┌─ Transport ─────────────────────────────────────────┐
│  packages/solo/src/mcp/    packages/solo/src/cli/   │  ← wire adapters
│  Zod-validate input → translate to signal           │
│  Imports: @modelcontextprotocol/sdk (mcp only)      │
│           commander (cli only)                      │
│           @foreman-lab/core (for signal types)          │
└──────────────────────┬──────────────────────────────┘
                       ▼ (signal: validated domain object)
┌─ Handler ───────────────────────────────────────────┐
│  packages/solo/src/handlers/                        │  ← domain logic
│  signal → graph.invoke(Command) or interrupt resume │
│  Imports: ./graph/, ./types/                        │
│           NO MCP or CLI imports                     │
└──────────────────────┬──────────────────────────────┘
                       ▼ (Command: LangGraph primitive)
┌─ Flow ──────────────────────────────────────────────┐
│  packages/solo/src/graph/                           │  ← LangGraph nodes
│  init / plan / work / evaluate + state channels     │
│  Imports: @langchain/langgraph (exclusive)          │
│           NO handler, transport, or domain imports  │
└─────────────────────────────────────────────────────┘

(At Phase 2a per D-9, `handlers/`, `graph/`, `types/`, `errors/` are extracted to `packages/core/src/...`; the boundary rules do not change. Post-2026-04-18 domain-blind pivot: no `ports/` or `adapters/` directory at Phase 1 — see ADR 0007. If Phase 2a promotes `TokenVendingPort`, those directories materialize inside `packages/core/` at that time.)
```

**Responsibilities per layer:**

- **Transport** — wire-format validation (Zod), serialization (JSON), protocol plumbing (MCP stdio framing, CLI argv parsing). Never invokes the graph directly.
- **Handler** — domain logic: "given this validated signal, what should happen?". Calls `graph.invoke()` with a `Command`. Implements business rules that aren't node-internal (e.g., routing a `TokenRequest` to vending hook).
- **Flow** — the LangGraph graph: nodes, edges, state channels, reducers. Consumes `Command` objects; emits state transitions and interrupts.

**Why this boundary matters:**

- MCP SDK changes → only Transport changes
- LangGraph changes → only Flow changes
- Domain rule changes → only Handler changes
- Three axes of change, three directories. One PR touches one directory in the common case.

**Enforcement:** `eslint-plugin-boundaries` or `dependency-cruiser` configured in Phase 1. CI fails the build on unauthorized cross-layer imports.

### 2.1 Monorepo layout (D-9 materialized)

Phase 1 ships a **single package**: `@foreman-lab/solo`. Core extraction (to `packages/core/`) happens at Phase 2a per D-9, when the daemon becomes the second consumer — not before.

```
foreman/
├── packages/
│   └── solo/                       # @foreman-lab/solo — everything at Phase 1
│       └── src/
│           ├── graph/              # LangGraph nodes, edges, channels
│           ├── handlers/           # signal → Command translation (use cases)
│           ├── types/              # Zod schemas, TS types
│           ├── errors/             # BaseError taxonomy
│           ├── mcp/                # MCP stdio inbound adapter
│           ├── cli/                # CLI inbound adapter
│           └── app.ts              # composition root (wires checkpointer + graph + handlers + mcp-server)
│                                   # (bin entrypoint is cli/bin.ts)
├── examples/                       # reserved for M5 — playbooks ship as docs + YAML fixtures
├── docs/                           # this doc set
├── pnpm-workspace.yaml
├── turbo.json
└── package.json
```

Layer boundaries (D-17) are enforced by **import-boundary lint within the package**, not by package splits. `graph/` imports only `@langchain/langgraph`; `handlers/` imports `graph/` + `types/`; `mcp/` and `cli/` import `handlers/` + `types/`. No cross-layer backwards imports. Per D-20 + ADR 0007 there is no `ports/` or `adapters/` directory at Phase 1; if Phase 2a promotes `TokenVendingPort`, they materialize inside `packages/core/` then.

Future tiers add packages and extract shared code:

- **Phase 2a (daemon):** extract `packages/core/` with `graph/`, `handlers/`, `types/`, `errors/`. If `TokenVendingPort` promotes to a real port at Phase 2a, `ports/` + `adapters/` directories materialize inside `packages/core/` at that time — not before. `packages/solo/` keeps `mcp/`, `cli/`, `app.ts`. `packages/daemon/` adds HTTP transport. Both depend on `@foreman-lab/core`.
- **Phase 2b (dashboard):** `packages/dashboard/` (UI only) depends on `packages/daemon/`.
- **Phase 3 (team):** `packages/team/` adds `PostgresSaver` + auth.
- **Phase 4 (enterprise):** `packages/enterprise/` adds SSO + multi-tenant.

Phase 2a core extraction is a mechanical refactor (~4 hours: codemod, test updates, publish `@foreman-lab/core` as a new package) that pays for itself by letting the public API be discovered before it's published (D-9 rationale).

### 2.2 Outbound Ports (zero at Phase 1 post-pivot)

Foreman defines **zero** explicit outbound ports at Phase 1 after the 2026-04-18 domain-blind pivot. The rule, non-port seams table, and promotion triggers live in `docs/foundations.md` D-20 (SSOT); see also `docs/adr/0007-domain-blind-core.md`.

**Pre-pivot retirement (tier-specific audit, recorded here because it landed as Phase 1 code):** M4 planning introduced `PromptAssemblerPort` with a default `SkillMethodologyAssembler` adapter that rendered prompts inside graph nodes by interpolating `brief.skill.methodology[phase]` against a flat vars bag. The code landed at M4 Batches A1/A2/B/C2 (`packages/solo/src/ports/`, `packages/solo/src/adapters/`, `types/prompt-view.ts`, `view-mapping.ts`, `nextPrompt` channel + `StepSuccessSchema` extension) and was fully reverted at the pivot. The M4 proposal file (`docs/phase-1-solo/m4-development-proposal.md`) carries a supersession banner recording this. Post-pivot, the step envelope carries raw `brief.playbook.phases[phase]` through the existing `brief` channel; external agents render locally.

---

## 3. Signal Protocol

A **signal** is the primary message type from agent to foreman. The signal protocol is the stable public API — agents interact with it via MCP or CLI. LangGraph's internals (`Command`, `Send`, `interrupt`) are implementation details; agents never see them (D-16, D-17).

### 3.1 Signal catalog

| Signal          | Direction                | Purpose                                                                                                                                                                                                                                   |
| --------------- | ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `initialize`    | Agent → Foreman          | Create a new node with a Brief (includes playbook); sets `nodeId`, `kind`                                                                                                                                                                 |
| `plan`          | Agent → Foreman          | Submit plan for current node; advances flow `plan` → `work` (or emits `Send` for composite)                                                                                                                                               |
| `work`          | Agent → Foreman          | Submit work report; advances flow `work` → `evaluate`                                                                                                                                                                                     |
| `evaluate`      | Agent → Foreman          | Submit evaluation; outcome is derived, not asserted                                                                                                                                                                                       |
| `resume`        | Operator → Foreman       | Resolve an active block (approve / override / fail)                                                                                                                                                                                       |
| `token_request` | Agent → Foreman          | Request scoped credentials (D-18 hook — Solo: pass-through local env; hosted: vends from KMS). **M3/M4: handler rejects with `SCHEMA_VALIDATION_FAILED`. M4 Batch C1 (unstub) was dropped to Phase 2a per operator + GAN 2026-04-18 — foreman as infrastructure does not call external APIs; pass-through vending provides zero security value vs the agent reading env directly.** `TokenVendingPort` materializes at Phase 2a when a real Vault/HSM backend provides scoping + revocation + audit. |
| `status`        | Agent/Operator → Foreman | Read-only: inspect current state (CQRS-style)                                                                                                                                                                                             |

### 3.2 Wire format

Every signal is:

```typescript
{
  type: "initialize" | "plan" | "work" | "evaluate" | "resume" | "token_request" | "status",
  nodeId: string,          // target node (for terminal), or root nodeId (for initialize)
  payload: /* signal-specific, see §3.3 */
}
```

All fields validated by Zod at the transport boundary (S-4). Unknown fields rejected. Unknown types rejected.

### 3.3 Per-signal schemas (representative — full definitions in `packages/solo/src/types/signal.ts`)

```typescript
// initialize — full brief, includes embedded playbook (D-18)
const InitializeSignal = z.object({
  type: z.literal('initialize'),
  nodeId: z.string(),
  payload: z.object({
    kind: z.enum(['terminal', 'composite']),
    brief: BriefSchema, // see §7
  }),
});

// plan — terminal: proposal; composite: children specs
const PlanSignal = z.object({
  type: z.literal('plan'),
  nodeId: z.string(),
  payload: z.discriminatedUnion('kind', [
    z.object({ kind: z.literal('terminal'), proposal: z.string() }),
    z.object({ kind: z.literal('composite'), children: z.array(BriefSchema) }),
  ]),
});

// work — agent reports what was done
const WorkSignal = z.object({
  type: z.literal('work'),
  nodeId: z.string(),
  payload: z.object({
    report: z.string(),
    toolCalls: z.array(ToolCallSchema), // each with reversibility (S-3)
  }),
});

// evaluate — assessment; outcome is DERIVED from this (not asserted)
const EvaluateSignal = z.object({
  type: z.literal('evaluate'),
  nodeId: z.string(),
  payload: z.object({
    findings: z.array(FindingSchema), // each with severity
    meetsCriteria: z.boolean(),
  }),
  // NO outcome field — derived by foreman
});

// resume — operator resolves a block
const ResumeSignal = z.object({
  type: z.literal('resume'),
  nodeId: z.string(),
  payload: z.discriminatedUnion('action', [
    z.object({ action: z.literal('approve'), note: z.string().optional() }),
    z.object({ action: z.literal('override'), justification: z.string() }),
    z.object({ action: z.literal('fail'), reason: z.string() }),
  ]),
});

// token_request — D-18 hook
const TokenRequestSignal = z.object({
  type: z.literal('token_request'),
  nodeId: z.string(),
  payload: z.object({
    scope: z.string(), // e.g., "github:repo:read"
    ttl: z.number().int().positive(), // seconds
  }),
});
```

### 3.4 Response format

Foreman replies with a discriminated wire envelope (D-4 wire boundary):

```typescript
type WireResponse<T> =
  | { ok: true; data: T }
  | {
      ok: false;
      error: { code: ErrorCode; message: string; details?: unknown };
    };
```

Internal code throws `BaseError` subclasses (D-4); a single catch at the transport boundary converts to this envelope.

### 3.5 Versioning

Signal schemas are versioned (policy deferred per foundations.md to pre-Phase-1 release). Breaking changes to any signal's payload schema require a major version bump. Adding a new optional field is non-breaking. Adding a new signal type is non-breaking (agents opt in).

---

## 4. Flow Specification

### 4.1 Node responsibilities

| Node       | Reads                                         | Writes                                                                          | Purpose                                     |
| ---------- | --------------------------------------------- | ------------------------------------------------------------------------------- | ------------------------------------------- |
| `initialize` | `InitializeSignal.payload`                    | `nodeId`, `kind`, `brief`; appends `initialize` event to journal                | Validate brief; establish identity          |
| `plan`     | `brief`, `PlanSignal.payload`                     | `plan`; for composite, emits `Send(subgraph, children[])`; appends `plan` event | Agent proposes approach                     |
| `work`     | `plan`, `WorkSignal.payload`                      | `work`; appends `work` event                                                    | Agent submits report                        |
| `evaluate` | `brief`, `plan`, `work`, `EvaluateSignal.payload` | `evaluation`; appends `evaluate` event; emits outcome                           | Preflight + agent eval + outcome derivation |

### 4.2 Conditional edges out of `evaluate`

```
evaluate → pass    if outcome = pass        → END
         → retry   if outcome = retry       → plan (unified retry — no kind branch)
         → block   if outcome = block       → interrupt(...) (metadata = block category)
         → exhausted if outcome = exhausted  → END (retries exhausted or unrecoverable)
```

**Outcome is derived by foreman, not asserted by the agent.** The derivation is a pure function:

```typescript
function deriveOutcome(input: {
  meetsCriteria: boolean;
  findings: Finding[]; // each with severity
  retryCount: number;
  maxRetries: number; // default 3
  blockDetected: boolean;
  blockCategory?: BlockCategory;
}): { outcome: Outcome; blockCategory?: BlockCategory };
```

**Priority-ordered rules (first match wins):**

| #   | Condition                                 | Outcome                         | Rationale                                                               |
| --- | ----------------------------------------- | ------------------------------- | ----------------------------------------------------------------------- |
| 1   | `blockDetected === true`                  | `block` (per `blockCategory`)   | Preflight / security / cascade / voluntary signals override everything  |
| 2   | Any finding has `severity === "critical"` | `block` (category: `integrity`) | Critical finding overrides agent self-assessment — harness vetoes (P-1) |
| 3   | `meetsCriteria === true`                  | `pass`                          | Agent says pass, no critical finding, no block — accept                 |
| 4   | `retryCount < maxRetries`                 | `retry`                         | Not passing, retries still available                                    |
| 5   | else                                      | `exhausted`                     | Retries exhausted, not passing, not blocked                             |

**Edge case explained:** `meetsCriteria === true` **AND** any critical finding = **block**, not pass. The agent's self-assessment does not override evidence. This is P-1 in action — the harness decides outcome based on findings, not the agent's claim.

The truth table is a pure TypeScript function in `packages/solo/src/graph/outcome.ts`, unit-tested exhaustively against the full `(meetsCriteria, severity, retryCount, blockDetected)` input space. The `evaluate` node calls it; conditional edges branch on its return value.

### 4.3 Interrupt semantics

Interrupts pause the graph **at node boundaries** (between edges), not mid-node. Triggered inside `evaluate` when outcome is `block`. Emits to the agent/operator:

```typescript
{
  nodeId: string,
  block: {
    category: "security" | "integrity" | "preflight" | "authenticity" | "configuration" | "cascade" | "voluntary",
    originNode: "init" | "plan" | "work",  // never "evaluate" — I-9
    reason: string
  }
}
```

Operator sends `ResumeSignal` to unblock. The conditional edge from `interrupt()` back into the graph routes to `originNode` (derived from the block event in journal — I-8).

**Why no `"evaluate"` in `originNode`:** Blocks detected during the `evaluate` node always resume to `work` — the agent redoes the work to address the finding. Evaluate is deterministic given the same work input (P-5), so re-entering evaluate without changing work produces the same block. When an operator decides a flagged finding should not block, they use `ResumeSignal.action: "override"` → outcome flips to `pass` directly; the graph does not re-run `evaluate`.

### 4.4 Composite coordination

Composite plan returns `Send` to a subgraph compiled from the same graph definition. Subgraph state nests under LangGraph's `checkpoint_ns` within the parent's `thread_id`. **All nodes in one project share one `thread_id`** (the project ID derived from the `.foreman/` location); **each node gets its own `checkpoint_ns`** equal to its `nodeId`.

```typescript
// in parent's evaluate node — pull each child's final state
const childStates = await Promise.all(
  children.map((c) =>
    graph.getState({
      configurable: {
        thread_id: projectId,
        checkpoint_ns: c.nodeId,
      },
    }),
  ),
);
const allPass = childStates.every(
  (s) => s.values.evaluation?.outcome === 'pass',
);
// ... parent's own evaluation based on aggregate
```

**Why `checkpoint_ns` nesting (not separate `thread_id`s):** One `thread_id` per project gives automatic subgraph interrupt propagation, shared time-travel across the whole project, and coherent checkpoint history. Separate `thread_id`s would be simpler to reason about but lose these features — features Phase 2+ will rely on for dashboard timeline views and operator inspection.

No push signals, no cascade scans. Parent pulls via subgraph state API.

### 4.5 Reducible vs irreducible state

`journal` uses an append reducer (`(a, b) => [...a, ...b]`). All other channels use replace semantics (latest write wins). This matches LangGraph idioms and makes replay from a checkpoint deterministic (P-5).

---

## 5. State Channel Specification

Seven channels. Full Zod schemas in `packages/solo/src/types/state.ts`. Representative sketches:

```typescript
const FlowState = Annotation.Root({
  nodeId: Annotation<string>(),
  kind: Annotation<'terminal' | 'composite'>(),
  brief: Annotation<Brief>(), // includes playbook (D-18)
  plan: Annotation<Plan | null>(),
  work: Annotation<WorkReport | null>(),
  evaluation: Annotation<Evaluation | null>(),
  journal: Annotation<JournalEvent[]>({
    reducer: (a, b) => [...a, ...b],
    default: () => [],
  }),
});
```

### 5.1 Per-channel semantics

| Channel      | Type                        | Reducer | Written by | Notes                                              |
| ------------ | --------------------------- | ------- | ---------- | -------------------------------------------------- |
| `nodeId`     | `string`                    | replace | `initialize` | Never rewritten after set                          |
| `kind`       | `"terminal" \| "composite"` | replace | `initialize` | I-2 enforced                                       |
| `brief`      | `Brief`                     | replace | `initialize` | `Readonly<Brief>` in code; I-1 enforced            |
| `plan`       | `Plan \| null`              | replace | `plan`       | Null until first plan signal                       |
| `work`       | `WorkReport \| null`        | replace | `work`       | Null until first work signal                       |
| `evaluation` | `Evaluation \| null`        | replace | `evaluate`   | Null until first eval; preflight findings included |
| `journal`    | `JournalEvent[]`            | append  | all nodes    | P-5 audit trail                                    |

### 5.2 Derived state (not channels)

| Derived value  | Source                                          | Used by                                     |
| -------------- | ----------------------------------------------- | ------------------------------------------- |
| `retryCount`   | `journal.filter(e => e.type === "evaluate").length` | outcome derivation                          |
| `activeBlock`  | Last `block` event in `journal` (if unresolved)     | operator UI                                 |
| `resumeTarget` | Block category + journal scan (I-8)                 | interrupt edge routing                      |
| `playbook`     | `brief.playbook`                                    | prompt assembly; never a standalone channel |
| `children`     | Subgraph state API (composite only)                 | parent `evaluate`                           |

### 5.3 `projectId` field (Gate 6 hedge)

`projectId: string` lives inside `brief` (not a top-level channel). Used to scope multi-project aggregation when Phase 2+ hosted service needs it. Phase 1 Solo sets it to a stable identifier derived from `.foreman/` parent directory path; hosted service uses a user-assigned project ID.

---

## 6. Storage Model

### 6.1 Checkpointer contract (D-16)

Foreman uses LangGraph's `BaseCheckpointSaver` interface. All state persistence — channels, events, replay positions — is delegated to the configured saver:

| Tier                 | Saver                                      | Package                                    |
| -------------------- | ------------------------------------------ | ------------------------------------------ |
| Phase 1 (Solo)       | `SqliteSaver`                              | `@langchain/langgraph-checkpoint-sqlite`   |
| Phase 3 (Team)       | `PostgresSaver`                            | `@langchain/langgraph-checkpoint-postgres` |
| Phase 4 (Enterprise) | `PostgresSaver` + tenant-scoped key prefix | Same, wrapped                              |

Foreman **never** reads or writes raw DB rows (S-5). Agents don't see the saver at all — they see only the signal protocol.

**Schema evolution:** LangGraph state channels are additive by convention — adding a new channel with a default value is a backward-compatible schema change. Existing checkpoints reload with the new channel at its default. Renaming or removing a channel is a breaking change that requires a migration step (dump → transform → reload) and a signal-protocol version bump (D-7). No automatic migration across phases per P-8.

### 6.2 `.foreman/` directory layout (Phase 1 Solo)

```
<project-root>/
└── .foreman/                          # project root marker (like .git/)
    ├── checkpoints.db                 # SqliteSaver file (all channels + journal)
    ├── config.json                    # project-scoped config (D-8 precedence)
    └── logs/                          # reserved for Phase 2a daemon logs
```

`.foreman/` is the **project root marker**. Foreman's CLI walks up from CWD until it finds `.foreman/` (mirrors `.git/` discovery). One `.foreman/` = one SQLite file = N threads (one per `nodeId`).

### 6.3 Journal channel vs external log

The `journal` state channel inside a checkpoint IS the journal. Not a separate file, not a separate table. Export via LangGraph's state-history API or `sqlite3 .dump`. JSONL export is deferred to Phase 2 if consumers need it (P-6 satisfied by SQLite being a documented, portable format).

---

## 7. Playbook Contract (D-18 materialized)

### 7.1 Brief schema

```typescript
const Brief = z.object({
  nodeId: z.string(),
  kind: z.enum(['terminal', 'composite']),
  goal: z.string(),
  projectId: z.string(),
  playbook: PlaybookSchema,
  children: z.array(z.string()).optional(), // composite only: child nodeId refs
});
```

### 7.2 Playbook schema

```typescript
const Playbook = z.object({
  name: z.string(),
  phases: z.object({
    plan: z.string(), // prompt segment for plan node
    work: z.string(), // prompt segment for work node
    evaluate: z.string(), // prompt segment for evaluate node
  }),
  scope: z
    .object({
      readOnly: z.boolean(),
      paths: z.array(z.string()).optional(), // allowed read/write paths
    })
    .optional(),
});
```

No tool names, no agent-specific fields. The playbook is phase content, not agent wiring.

### 7.3 Injection flow

```
┌─────────┐                                     ┌──────────┐
│  Agent  │                                     │ Foreman  │
└────┬────┘                                     └─────┬────┘
     │                                                │
     │ 1. Reads playbook YAML / TS / DB from local FS │
     │    (foreman does NOT read playbook files)      │
     │                                                │
     │ 2. Constructs Brief { ..., playbook: {...} }   │
     │                                                │
     │ 3. initialize signal ──────────────────────────▶│
     │                                                │ 4. Zod validate
     │                                                │ 5. Store in `brief` channel
     │                                                │ 6. Emit initialize journal event
     │◀───────────────────── initialized response ────┤
     │                                                │
     │ ... subsequent plan/work/evaluate signals ...  │
     │                                                │
     │ foreman stores brief.playbook verbatim;        │
     │ agent reads phases via foreman__status         │
     │ and renders each phase's prompt locally        │
     │ (domain-blind per ADR 0007)                    │
```

Key properties:

- **foreman's filesystem never touches playbook files** — wrappers parse YAML/TS into a `Playbook` object and embed it inline in `brief.playbook` at init. Preserves P-1 (pure infrastructure), P-4 (agent-primary), and avoids agent-specific config leakage.
- **foreman does not render prompts** — the agent reads `brief.playbook.phases[plan|work|evaluate]` verbatim via `foreman__status` (guarded by `brief-exposure.test.ts`) and substitutes placeholders locally. ADR 0007 locks this domain-blind posture.

---

## 8. Error Model (D-4 materialized)

### 8.1 Error class hierarchy

```
BaseError              (base — generic/uncategorized)
├── ValidationError       (schema failure, malformed input)
├── SecurityError         (unauthorized, missing approval, credential issue)
└── StorageError          (DB failure, corruption, constraint violation)
```

Each carries:

- `code: ErrorCode` (SCREAMING_SNAKE)
- `message: string`
- `details?: unknown` (structured payload for diagnosis)

### 8.2 Internal vs wire

- **Internal code:** `throw new ValidationError("BRIEF_KIND_MISSING", "...")`. Functions type their return as pure success; failure is an exception. No Result types (D-4).
- **Transport boundary:** single catch converts to `WireResponse<T>` wire format. Internal code stays clean; protocol remains explicit.

### 8.3 Error code catalog (representative — full list in `docs/error-codes.md`)

| Code                          | Class      | Cause                                       | Fix                                                            |
| ----------------------------- | ---------- | ------------------------------------------- | -------------------------------------------------------------- |
| `SCHEMA_VALIDATION_FAILED`    | Validation | Signal did not match Zod schema             | Check signal shape against `packages/solo/src/types/signal.ts` |
| `BRIEF_KIND_MISSING`          | Validation | `initialize` signal omitted `kind`          | Set `kind: "terminal"` or `"composite"` in payload             |
| `NODE_NOT_FOUND`              | Validation | Signal targeted an unknown `nodeId`         | Verify node was initialized                                    |
| `OPERATOR_APPROVAL_REQUIRED`  | Security   | Irreversible ToolCall without operator gate | Mark reversibility; wait for resume                            |
| `PREFLIGHT_SECRET_DETECTED`   | Security   | Work report contains credential pattern     | Remove secret; request TokenRequest instead                    |
| `CHECKPOINTER_WRITE_FAILED`   | Storage    | SQLite/Postgres write error                 | Check disk/DB availability; foreman auto-retries once          |
| `JOURNAL_CORRUPTION_DETECTED` | Storage    | Journal read cannot deserialize             | Inspect via `foreman inspect`; restore from prior checkpoint   |

`docs/error-codes.md` (Phase 1 deliverable) enumerates every code with cause + fix + example response body. P-6 data portability requires this.

---

## 9. Security Model

S-1..S-5 (foundations.md) materialized in concrete mechanisms:

### S-1: No ambient credentials

**Mechanism:** `TokenRequest` signal (Phase 1 stub; real vending deferred to Phase 2a).

- Agent never reads env vars for credentials itself when calling foreman-mediated tools
- Agent sends `TokenRequest { scope, ttl }` → Phase 1/M3+M4 foreman rejects with `SCHEMA_VALIDATION_FAILED` + `details.reason: 'token_request deferred to Phase 2a'` (M4 Batch C1 drop per ADR 0003 amendment + ADR 0007)
- Phase 2a hosted tier will introduce `TokenVendingPort` with a Vault/HSM adapter; that is the first real outbound port foreman ships (matches D-20 narrow-port discipline — real variation exists at Phase 2a)
- Pre-commit scan + runtime assertion guard against accidental env-var reads elsewhere
- Signal variant stays in `StepSignalSchema` for forward-compat; when Phase 2a promotes the port, `TOKEN_VENDING_FAILED` materializes as the active error code (reserved in `ERROR_CODE_VALUES` since M4)

### S-2: Operator approval mandatory for irreversible actions

**Mechanism:** `interrupt()` at node boundary + `ResumeSignal.action = "approve" | "override" | "fail"`.

- Work report's ToolCalls carry `reversibility: "reversible" | "irreversible" | "unknown"` (S-3)
- If any ToolCall is non-reversible and lacks prior approval → `evaluate` emits `block` with category `security`
- Interrupt pauses flow; operator must explicitly resume with `approve` or `override`
- No bypass flag in CLI or MCP surface

### S-3: Action classification

**Mechanism:** Zod-validated `reversibility` field on every `ToolCall`.

- Schema: `z.enum(["reversible", "irreversible", "unknown"]).default("unknown")`
- Unknown defaults to irreversible (safe)
- Agents are expected to classify; foreman does not infer

### S-4: MCP transport input validation

**Mechanism:** Zod parse at the transport boundary, before any handler invocation.

- Every inbound MCP message parsed through the corresponding signal schema
- Parse failure → `ValidationError` → wire envelope with `ok: false`
- Unknown fields stripped (`.strict()` for top-level; `.passthrough()` only where explicitly allowed)

### S-5: Checkpointer-only state access

**Mechanism:** No direct DB client in handler/transport layers. Type boundary enforces it.

- `packages/solo/src/graph/` imports `@langchain/langgraph` exclusively
- Handlers call `graph.invoke()` / `graph.getState()` — never SQLite or Postgres directly
- Import-boundary lint prevents accidental cross-imports

---

## 10. Tier Variations

All tiers share: 4-node flow, signal protocol, three-actor model, handler/transport separation, playbook injection, error model, security model. What varies:

| Concern            | Phase 1 Solo                 | Phase 2a Daemon                                                                           | Phase 2b Dashboard | Phase 3 Team           | Phase 4 Enterprise                  |
| ------------------ | ---------------------------- | ----------------------------------------------------------------------------------------- | ------------------ | ---------------------- | ----------------------------------- |
| Checkpointer       | `SqliteSaver`                | `SqliteSaver`                                                                             | (same)             | `PostgresSaver`        | `PostgresSaver` + tenant key prefix |
| Transport          | MCP stdio + CLI              | + HTTP                                                                                    | + WebSocket/SSE    | + auth headers         | + SSO (SAML/OIDC)                   |
| Auth               | OS file perms                | Localhost bind                                                                            | (same)             | User accounts + RBAC   | SSO + SCIM                          |
| Storage location   | `.foreman/checkpoints.db`    | Daemon-owned file                                                                         | (same)             | Team Postgres instance | Per-tenant schema                   |
| Logging            | stdio (no separate log)      | Structured (library TBD — locked at Phase 2a start per foundations.md deferred decisions) | (same)             | (same)                 | + SIEM export                       |
| Observability      | CLI (`foreman inspect`)      | HTTP status endpoint                                                                      | Web UI             | Team dashboard         | Audit export                        |
| Playbook source    | Agent injects per-invocation | (same)                                                                                    | (same)             | Team playbook library  | Private registry mirror             |
| Vending hook (S-1) | Pass-through env             | (same) — local dev                                                                        | (same)             | Team vault             | HSM/KMS integration                 |

Every tier's code is **additive** (P-8 clean breaks): Phase 2 adds packages; it does not modify `packages/core/`.

---

## 11. Cross-cutting Concerns

### 11.1 Reproducibility (P-5)

- No internal randomness in foreman code (verified by principle test)
- No wall-clock decisions (`Date.now()` only used for journal timestamps, never as control flow)
- LangGraph checkpointer is deterministic given same inputs + state
- Resume: `graph.invoke(new Command({ resume: {} }), config)` on same `thread_id` continues from the last checkpoint
- Replay (rewinding to a prior checkpoint) uses `graph.updateState()` + `invoke()` — a different operation from resume

### 11.2 Observability

- Journal channel is the **single source of truth** for what happened
- `foreman inspect` dumps current state (channels + last N journal events) as JSON
- `foreman status` summarizes: current node, active block if any, round count, last outcome
- Phase 2a daemon adds structured logs via `pino` — complementary to journal, not replacing

### 11.3 Configuration precedence

See D-8 in [`foundations.md`](foundations.md). One source of truth; this doc does not duplicate.

### 11.4 Versioning (deferred per foundations.md)

Signal protocol versioning policy will be locked before the first `npm publish` (see foundations.md deferred decisions). Until then: additive changes only; no breaking changes to any signal schema.

### 11.5 Concurrency model (Phase 1 Solo)

SQLite is single-writer. Foreman's handler layer maintains a **per-nodeId async mutex**: concurrent signals targeting the same `nodeId` serialize at the handler before `graph.invoke()`. Second request waits for first to complete.

```typescript
// packages/solo/src/handlers/mutex.ts — sketch
const locks = new Map<string, Promise<void>>();

export async function withNodeLock<T>(
  nodeId: string,
  fn: () => Promise<T>,
): Promise<T> {
  const prev = locks.get(nodeId) ?? Promise.resolve();
  let release!: () => void;
  const next = new Promise<void>((r) => {
    release = r;
  });
  locks.set(
    nodeId,
    prev.then(() => next),
  );
  await prev;
  try {
    return await fn();
  } finally {
    release();
    if (locks.get(nodeId) === next) locks.delete(nodeId);
  }
}
```

**Scope:** per-nodeId, not per-project. Concurrent signals to **different** nodes within the same project run in parallel at the handler layer.

**SQLite caveat:** SQLite remains single-writer at the DB level. When two concurrent handler invocations reach different `checkpoint_ns` values under one `thread_id`, their writes serialize inside SQLite and may briefly contend. The checkpointer is opened with WAL mode (`PRAGMA journal_mode = WAL`); on `SQLITE_BUSY` the checkpointer retries transparently. Phase 1 Solo does not need additional DB-level locking.

**Phase 2a daemon:** extends to distributed locks across machines; PostgresSaver at Phase 3 removes the single-writer constraint entirely.

### 11.6 Timeout / cancellation (Phase 1 Solo)

Each node has a **24-hour TTL** stored in its `init` journal event as `ttlExpiresAt: ISO 8601 timestamp`. No heartbeat signal, no background timer.

On any `status` read, foreman compares `ttlExpiresAt` against current time. Expired nodes surface as `stale: true` in the response. **`stale` is advisory, not enforcing:** a stale node continues to accept signals normally — an agent returning after a weekend offline can send `plan`/`work`/`eval` without foreman rejecting. The flag simply tells the operator "this node has been idle for a long time; consider whether it's still relevant." Operator can explicitly close stale nodes via:

- `ResumeSignal.action: "fail"` → node transitions to END with `exhausted` outcome + `reason: "ttl_expired"`
- Re-initializing a new node if the work has moved on

**Rationale:** Agents that vanish (killed process, dropped network, closed terminal) otherwise leave invisible zombie state. Passive TTL makes `foreman status` honest about idle work without the complexity of heartbeats or active timers. Keeping `stale` advisory preserves the weekend-offline use case that a strict expiration would break.

**Phase 2a daemon:** extends to active background scanner for timely cleanup.

### 11.7 Composition root

The composition root is `packages/solo/src/app.ts` (consumed by the CLI bin entrypoint at `packages/solo/src/cli/bin.ts`). It:

1. Resolves config per D-8 precedence (CLI flags → env → project config → defaults)
2. Instantiates the checkpointer (Phase 1: `SqliteSaver.fromConnString(.foreman/checkpoints.db)`)
3. Builds the graph with channels + reducers + compiled nodes (accepts checkpointer + clock + retry cap; no outbound ports per §2.2 post-pivot)
4. Wires handlers with the compiled graph + projectId + clock
5. Starts the transport adapter (MCP stdio via `createMcpServer`, or CLI subcommand via `createCliProgram`) with handler bindings

```typescript
// packages/solo/src/app.ts — post-pivot sketch (see real file for current source)
export function buildApp(options: BuildAppOptions = {}): App {
  const cwd = options.cwd ?? process.cwd();
  const env = options.env ?? process.env;
  const now = options.now ?? (() => new Date());

  const openRuntime = (foremanDirOverride?: string): CliRuntime => {
    const resolved = resolveConfig({ cwd, env, cliForemanDir: foremanDirOverride });
    const handle = openSqliteCheckpointer(options.checkpointPath ?? resolved.checkpointPath);
    const graph = buildGraph({ checkpointer: handle.checkpointer, now });
    const handlers = createHandlers({ graph, projectId: resolved.config.projectId ?? 'default' });
    const mcpServer = createMcpServer({ handlers });
    return { mcpServer, handlers, close: async () => { await mcpServer.close(); handle.close(); } };
  };

  const cliProgram = createCliProgram({ cwd, env, openRuntime, ... });
  return { cliProgram, shutdown: async () => { /* close runtime if opened */ } };
}
```

**Pre-pivot note:** M4 Batch C2 wired `SkillMethodologyAssembler` (PromptAssemblerPort adapter) at this root. The 2026-04-18 domain-blind pivot reverted that — agents render prompts externally, so no assembler instantiation lives here. If Phase 2a introduces `TokenVendingPort`, its adapter instantiation (e.g., `VaultTokenVendor`) lands here alongside the existing wiring.

**Phase 2a daemon** introduces its own composition root at `packages/daemon/src/app.ts` that reuses `@foreman-lab/core` and swaps transport to HTTP + vending hook to a Vault-backed `TokenVendingPort` adapter.

---

## Appendix A: Cross-reference index

| Topic                              | Primary doc                                   | Secondary                                                                                     |
| ---------------------------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------- |
| Principles (P-1..P-10)             | foundations.md §1                             | this doc §1, §9                                                                               |
| Security requirements (S-1..S-5)   | foundations.md §2                             | this doc §9                                                                                   |
| Foundational decisions (D-1..D-20) | foundations.md §3                             | this doc — D-8 §11.3; D-9 §2.1; D-16 §1, §4, §6; D-17 §2; D-18 §7; D-19 throughout; D-20 §2.2 |
| Phase scope                        | roadmap.md                                    | this doc §10                                                                                  |
| Tripwires                          | roadmap.md § Tripwires                        | this doc (none — roadmap is authoritative)                                                    |
| Hosted-service moat                | roadmap.md § Phase 2                          | this doc §10 (tier variations)                                                                |
| Ship criteria for Phase 1          | `docs/phase-1-solo/spec.md` (exists — locked) | this doc (none)                                                                               |
| Error code catalog                 | `docs/error-codes.md` (Phase 1 deliverable)   | this doc §8.3                                                                                 |

## Appendix B: Glossary (D-19 locked vocabulary)

Minimal restatement — full table with "synonyms to avoid" in foundations.md D-19.

- **Node** — unit of work under orchestration
- **Brief** — immutable contract for a node
- **Playbook** — phase-content contract embedded in brief
- **Plan** — agent's proposal
- **Work** — agent's report
- **Evaluation** — agent's assessment
- **Outcome** — derived truth-table result (pass/retry/block/exhausted)
- **Block** — operator-input-required pause
- **Journal** — append-only event log
- **kind** — `"terminal"` or `"composite"`
- **Checkpoint** — LangGraph state snapshot
- **Operator** — human judge
- **Agent** — AI caller
- **Harness** — foreman itself

---

**End of document.** This is the locked Phase 1 baseline. Revisions require ADR per foundations.md Drift Prevention rules.
