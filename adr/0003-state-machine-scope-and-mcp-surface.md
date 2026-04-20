---
status: Approved v2 — rounds 2/3/4 merged; C3 spike PASS (protocol level); host-rendering manual check pending
approved_by: user (kidtsui@outlook.com)
approved_date: 2026-04-18
date: 2026-04-18
deciders: Operator (xuzhijie), after rounds 1–4 debate with Codex + Copilot
supersedes: none
superseded_by: none
amends:
  - docs/phase-1-solo/spec.md §6 (MCP tool surface — 7 tools → 2 tools)
  - docs/phase-1-solo/spec.md §8.1 (signal catalog — stays 7 types; reclassified as internal vocabulary; `retry` added, `status` stays read-only)
  - docs/phase-1-solo/spec.md §8.3 (error codes — phrased as sequence-validity, not identity-validity)
  - docs/phase-1-solo/spec.md §8.4 (per-signal schemas — add `retry` row; resume action routing is state-sensitive)
  - docs/phase-1-solo/spec.md §7 (CLI — `foreman step --signal` + drops `foreman resume` as separate command)
  - docs/phase-1-solo/architecture.md §4.3 (interrupt/resume flow — resume arrives via `step(type:'resume')`; exhausted uses `step(type:'retry')` as a control signal)
  - docs/phase-1-solo/architecture.md §5.2 (mcp/ layer — registers 2 tools, not 7)
  - docs/foundations.md D-21 (clarification: MCP tool names are transport affordances; semantic boundary is signal + channel + journal)
companion:
  - Downstream phase sections of spec.md / roadmap.md (if any imply foreman performs auth/identity enforcement)
---

# ADR 0003: Foreman state-machine scope + 2-tool MCP surface + wrapper-owns-auth doctrine

## Status and scope

Draft. The change is driven by a round-2 design debate (2026-04-18) in which the operator reframed foreman as a pure state machine `(S, Σ, δ)`. Under that framing, several decisions baked into the Phase 1 spec (7-tool MCP surface, operator-vs-agent partitioned by tool name, and implicit assumptions about foreman performing authentication at Phase 2a+) are drift relative to the operator's intent.

This ADR captures **protocol-shape decisions** (MCP tool surface, actor handling, auth boundary, sequence-validity enforcement). Engine-specific decisions (LangGraph 1.x, checkpointer namespace) remain in ADR 0002. Architectural style (7-pattern hybrid) remains in ADR 0001.

## Context

### What was in the spec before this ADR

`docs/phase-1-solo/spec.md` §6 locked a **7-tool MCP surface**:

| Tool name                | Wraps signal    | Direction                | Read/write      |
| ------------------------ | --------------- | ------------------------ | --------------- |
| `foreman__initialize`    | `initialize`    | Agent → Foreman          | Write           |
| `foreman__plan`          | `plan`          | Agent → Foreman          | Write           |
| `foreman__work`          | `work`          | Agent → Foreman          | Write           |
| `foreman__eval`          | `eval`          | Agent → Foreman          | Write           |
| `foreman__resume`        | `resume`        | Operator → Foreman       | Write           |
| `foreman__token_request` | `token_request` | Agent → Foreman          | Write           |
| `foreman__status`        | `status`        | Agent/Operator → Foreman | **Read** (CQRS) |

The spec asserted (§6:396): _"Foreman exposes **7 MCP tools**, namespace-prefixed `foreman__*`. Each tool wraps exactly one signal type."_ A "Direction" column distinguished agent-actor tools from operator-actor tools.

### The drift

Under a state-machine framing `(S, Σ, δ)`:

- **S** = graph state = channels (`nodeId`, `kind`, `brief`, `plan`, `work`, `evaluation`, `journal`) as maintained by LangGraph `Annotation.Root`.
- **Σ** = signal alphabet = valid inputs (`initialize`, `plan`, `work`, `eval`, `resume`, `token_request`).
- **δ** = transition function = `(current_state, signal) → next_state`, realised by `graph.invoke` + the per-node reducers + the dispatch-on-signal topology (D-M2-16).

A pure state machine **does not discriminate inputs by who sent them**. It only validates `(state, signal) → transition_valid`. Who sent the signal is recorded as an extrinsic metadata field on the journal event — `types/journal.ts:12` already defines `JournalActorSchema = z.enum(['agent', 'operator', 'harness'])`, confirming that actor is a property of the recorded event, not a partition of the transition function.

Three consequences of taking the framing seriously:

1. **`resume` is not a distinct operation** — it is a transition variant `interrupted → {plan|work|evaluate}` with payload `{action, note?}`. The fact that a human typically sends it is journal metadata, not a signal-type distinction that should be encoded at the MCP transport layer.
2. **Partitioning MCP tools by actor** (one tool for "Agent → Foreman" writes, a separate tool for "Operator → Foreman" writes) re-smuggles request-response framing into a state-store abstraction. The partition belongs in the wrapper around foreman (which may understand agent-vs-operator identity), not in foreman itself.
3. **Foreman does not authenticate, authorize, or distinguish identity at any phase.** That is the wrapper's concern — the MCP stdio client at Phase 1, the daemon at Phase 2a, the team service at Phase 2b, the enterprise backend at Phase 3. Foreman records the `actor` field as given by the caller and journals it.

### Why this matters now (timing)

- M2 committed (`b8a2c88`) with a 3-tool subset of the 7-tool surface (`foreman__initialize`, `foreman__plan`, `foreman__status`). M3 was previously scoped to add the remaining 4 tools.
- Before M3 starts, the spec should be amended so that any skill YAML, agent instruction, test fixture, or downstream doc authored during M3 dogfooding references the _correct_ surface. Renaming tool names after skills are written is a breaking change.
- The package is `version: 0.0.0` and unpublished; live consumer risk is zero; documentation drift is real and active.

## Decision

**Adopt a 2-tool MCP surface, grounded in a state-machine scope, with a wrapper-owns-auth doctrine.** Specifically:

### 1. Foreman's scope (the doctrine)

Foreman at every phase is **state-machine storage** — nothing more.

| Phase                 | Wrapper                                  | Foreman's role                                         |
| --------------------- | ---------------------------------------- | ------------------------------------------------------ |
| Phase 1 (Solo)        | Local MCP stdio client, single user      | Store states; accept validated signals; journal events |
| Phase 2a (Daemon)     | HTTP/IPC daemon with its own auth        | Same                                                   |
| Phase 2b (Team)       | Team service with SSO / RBAC             | Same                                                   |
| Phase 3+ (Enterprise) | Enterprise backend with tenant isolation | Same                                                   |

**Responsibilities owned by the wrapper, never by foreman:** authentication, authorization, operator-vs-agent discrimination, tenant identity, approval-authority verification, session management, rate limiting.

**Responsibilities owned by foreman, at every phase:** state storage (channels + journal), signal validation (shape + sequence), transition dispatch, audit-trail append, checkpoint persistence.

Multi-tenancy at Phase 2a is achieved by prefixing `thread_id` (per ADR 0002 Consequences: `${tenantId}:${projectId}:${nodeId}` three-segment composite), not by foreman learning to understand tenant identity.

### 2. MCP tool surface = 2 tools

Replace the 7-tool table with:

| Tool name         | Kind  | Purpose                                                                                                                                                |
| ----------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `foreman__status` | Read  | Project current state to a read model. No side effects.                                                                                                |
| `foreman__step`   | Write | Polymorphic advance: one of `{initialize, plan, work, eval, resume, retry, token_request}` payload variants. Exactly one semantic transition per call. |

`foreman__step` input is a Zod discriminated union keyed on the `type` field:

```typescript
const StepSignal = z.discriminatedUnion('type', [
  z.object({ type: z.literal('initialize'),    nodeId: …, payload: InitializePayload,   actor: z.enum(['agent','operator','harness']).optional() }),
  z.object({ type: z.literal('plan'),          nodeId: …, payload: PlanPayload,         actor: … }),
  z.object({ type: z.literal('work'),          nodeId: …, payload: WorkPayload,         actor: … }),
  z.object({ type: z.literal('evaluate'),      nodeId: …, payload: EvaluatePayload,     actor: … }),
  z.object({ type: z.literal('resume'),        nodeId: …, payload: ResumePayload,       actor: … }), // block-state intervention; spec §8.4 action = approve|override|fail
  z.object({ type: z.literal('retry'),         nodeId: …, payload: RetryPayload,        actor: … }), // exhausted-state control signal; α-only (target=plan)
  z.object({ type: z.literal('token_request'), nodeId: …, payload: TokenRequestPayload, actor: … }),
]);
```

`actor` is caller-asserted. At Phase 1 it is trusted. At Phase 2a+ the wrapper stamps it before forwarding — the state machine records what it is given. No `actor` field is hard-constrained per variant (e.g., no `actor: z.literal('operator')` on resume) because actor enforcement is a wrapper concern, not foreman's.

### 2.5 Block and exhausted — different intervention mechanisms

The two stuck states use two different mechanisms:

| Stuck state                                                               | Mechanism                                                                                               | Operator signal                                                     | Routing                                                                                                                                                                                                                                                                                                                                                                                                           |
| ------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `block` (rules 1–2 of deriveOutcome: blockDetected OR hasCriticalFinding) | LangGraph `interrupt()` inside `n_evaluate` throws `GraphInterrupt`; node body re-runs on resume (v0.7) | `step(type='resume', payload={action, note?/justification/reason})` | Handler sends `new Command({resume: payload})` to `graph.invoke` (v0.7 fix — **invoke-input Commands accept only `resume`** per LangGraph 1.x docs; `goto`/`update`/`graph` are node-return-only). Evaluate node owns routing: `approve`/`override` return a plain state-update + static `n_evaluate → END` edge fires; `fail` returns `new Command({update: {journal}, goto: END})` (node-return Command shape). |
| `exhausted` (rule 5: retryCount ≥ maxRetries)                             | Graph runs to END; state is **terminal**                                                                | `step(type='retry', payload={})`                                    | Control signal: append retry_reset journal marker, clear `evaluation` channel to `null`. Agent then sends `step(type='plan', ...)` which dispatches normally to `n_plan`                                                                                                                                                                                                                                          |

Resume IS a LangGraph runtime primitive (it unpauses a paused graph). Per the official LangGraph 1.x docs (`docs.langchain.com/oss/javascript/langgraph/interrupts`): when resumed, the node body **re-runs from the top** — any code before `interrupt()` runs twice; `interrupt()` returns the resume payload on the second invocation. Foreman's evaluate node keeps side-effect writes (journal append, state return) AFTER `interrupt()` returns so they execute exactly once per resume.

Retry is NOT a LangGraph primitive — it's a state-preparation handler that resets counters and clears stale eval state, after which the agent re-enters via a normal `plan` signal. This distinction follows naturally from the state-machine view: block is a pause point foreman owns; exhausted is just a terminal state.

`RetryPayload` is α-only (no target discriminator) and **empty** at M3 (v0.3 YAGNI):

```typescript
const RetryPayload = z.object({}).strict();
```

Forward-compat: a future ADR may add `reason?: string` (audit note) and/or `target: 'plan' | 'work'` (rework-without-replan) if dogfooding surfaces demand. Adding optional fields does not break existing callers.

`retry-count.ts` semantics: count `evaluate` journal events **after the latest retry_reset marker** for this nodeId. If no marker exists, count all `evaluate` events (Phase 1 rule unchanged).

### 3. Internal signal vocabulary stays

`packages/solo/src/types/signal.ts` carries **8 discriminated-union variants**: `InitializeSignal`, `PlanSignal`, `WorkSignal`, `EvalSignal`, `ResumeSignal`, `RetrySignal` (new at M3), `TokenRequestSignal`, `StatusSignal`. Spec §8.1 reclassifies these as **internal vocabulary** — the payload-shape library — not a transport contract. The dispatch-on-signal graph topology (D-M2-16) remains unchanged; it gains a dispatch entry for `retry` (which routes to the handler's state-preparation logic, not to a graph node).

The refactor: split the top-level `SignalSchema` into `StepSignalSchema` (the 7 write variants: initialize, plan, work, eval, resume, retry, token_request) and `StatusSignalSchema` (the single read variant), reflecting the 2-tool MCP surface.

### 4. Sequence-validity, not identity-validity

Foreman's own enforcement checks **signal-sequence validity**, not caller-identity.

Examples:

- `OPERATOR_APPROVAL_REQUIRED` is thrown when an irreversible `ToolCall` is submitted without a prior journaled `resume` event carrying `action: 'approve'`. Foreman checks that the event exists in the journal; it does **not** check that the entity who journaled the resume is truly an operator. That is the wrapper's responsibility.
- `NODE_NOT_FOUND` is thrown when a `step` of type `plan|work|eval|resume|retry` targets a `nodeId` with no prior `initialize`. This is a state-validity check, actor-independent.
- `SCHEMA_VALIDATION_FAILED` is thrown when a duplicate `initialize` arrives for a `nodeId` already initialized. The set-once reducer on the `nodeId|kind|brief` channels enforces this at the state layer.
- **Resume is valid only from `block`**: a `step(type='resume')` targeting a node that is not currently paused at an `interrupt()` is rejected with `SCHEMA_VALIDATION_FAILED` (reason: "resume applies only to block state").
- **Retry is valid only from `exhausted`**: a `step(type='retry')` targeting a node whose last derived outcome is not `exhausted` is rejected with `SCHEMA_VALIDATION_FAILED` (reason: "retry applies only to exhausted state").

### 5. CLI shape adjustment

Phase 1 CLI gains `foreman step --signal <json>` (already proposed in spec §7, line 420). Under this ADR, `foreman resume --action <...>` drops as a separate CLI subcommand — it becomes `foreman step --signal '{"type":"resume","nodeId":"n1","payload":{"action":"approve"}}'`. Ergonomic CLI wrappers (`foreman plan --proposal ...`) may still be added as **UX sugar** on top of the single `step` subcommand, but they are not separate MCP tools.

The M3 **shipped** CLI surface is `init` + `status` + `step` = 3 operator subcommands (+ a hidden `mcp` run-mode used by MCP stdio clients). `inspect` + `validate-playbook` are planned in spec §7 but land at **M5** post-2026-04-18 domain-blind pivot (ADR 0007) — originally M4-alongside-playbook-assembly, slipped when the PromptAssemblerPort work was reverted and the example-playbook + agent-setup deliverables relocated to M5. The 5-subcommand count in prior drafts of this ADR was aspirational, corrected here per Codex Round 10 post-WORK GAN. `foreman resume --action <...>` is dropped as originally stated; it becomes `foreman step --signal '{"type":"resume",...}'`.

## Rationale

### Why 2 tools, not 3/4/5/7

| Tool count            | Who argued                                                            | Why rejected                                                                                                                                                       |
| --------------------- | --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 7 (original spec)     | Phase 1 spec baseline                                                 | Encodes request-response framing and actor partition in the transport contract. Spec §8.1's actor column is the drift vector.                                      |
| 5                     | Copilot round 2 (initialize + step + resume + token_request + status) | Keeps `resume` separate on security grounds (prevent agent self-approval via forged `actor`).                                                                      |
| 4                     | Codex round 1 (initialize + status + step + resume)                   | Same security argument as Copilot's 5-tool.                                                                                                                        |
| 3                     | Copilot round 1 (initialize + status + step)                          | Keeps initialize separate for pre-check asymmetry (`∅ → initialized` vs. `initialized → planned`).                                                                 |
| **2 (status + step)** | **Codex round 2 + operator correction**                               | **Chosen.** Consistent with state-machine framing. Pre-check asymmetry is handler logic, not a transport boundary. Actor/auth is a wrapper concern, not foreman's. |

### Why `initialize` collapses

The `∅ → initialized` pre-check is expressed inside the single `step` handler as a state-guard branch:

```typescript
if (signal.type === 'initialize' && state?.values.nodeId) {
  return error('SCHEMA_VALIDATION_FAILED', 'node already initialized');
}
if (signal.type !== 'initialize' && !state?.values.nodeId) {
  return error('NODE_NOT_FOUND', 'node not initialized');
}
```

One handler, two pre-check branches. Matches the state-machine semantics without introducing a transport-layer partition.

### Why `resume` collapses

The security argument against collapsing `resume` relied on the assumption that foreman, at Phase 2a or beyond, should enforce operator-vs-agent distinctions at the tool boundary. Under the scope doctrine adopted here, foreman never enforces identity at any phase. Enforcement lives in the wrapper. Therefore the tool partition provides no security benefit.

At Phase 1 (single-user dogfood) the self-approval case is not a threat — there is only one human. At Phase 2a+ the daemon/team/enterprise wrapper stamps `actor` from its own auth context before forwarding to foreman; an agent session cannot forge `actor: 'operator'` because the wrapper overrides the payload value.

### Why `retry` is its own signal type (not a `resume` action)

Round 3 of the debate considered adding a `retry` _action_ within `resume`'s action union (e.g., `resume(action: 'retry')` at capped). Round 4 rejected this in favor of a distinct signal type. Reasons:

- **One responsibility per signal type.** `resume` answers "how to resolve this block?" — three valid answers (approve/override/fail). `retry` answers "restart this capped node?" — one operation with no sub-actions. Mixing them into a single signal with 4+ actions would overload the semantic.
- **Different sequence-validity contexts.** `resume` is valid only from `block`; `retry` is valid only from `capped`. A single `resume` signal conflating these would need per-action validity checks (action='approve' valid only from block; action='retry' valid only from capped), which is less clean than per-signal-type validity.
- **Different mechanisms.** `resume` uses LangGraph's `Command({resume, goto})` to unpause an `interrupt()`. `retry` is a state-preparation control signal (resets journal counter + clears evaluation channel), followed by a normal `plan` signal. Binding these under one signal type would confuse the mechanics.
- **Naming clarity.** `retry` describes what the operator wants cleanly. `resume.action='retry'` would inherit the resume namespace while semantically belonging elsewhere.

Retry ships **α-only** at M3 (no target discrimination). A future ADR may add `target: 'plan' | 'work'` if dogfooding surfaces a frequent need for _rework-without-replan_. The current empty-payload shape is forward-compatible. See Alternatives Considered for the α+β debate.

### Why `token_request` collapses

If `token_request` appends an audit event to the journal (Phase 2a behavior once `TokenVendingPort` lands with a real Vault/HSM adapter), it is a semantic transition under foundations D-21. Semantic transitions belong in `step`. Until the vendor port is wired, the variant stays in the `step` discriminated union and the handler rejects with `SCHEMA_VALIDATION_FAILED` carrying `details: { reason: 'token_request deferred to Phase 2a' }` — no dedicated `NOT_IMPLEMENTED` error code is added (v0.3 YAGNI; see M3 proposal D-M3-14). The transport surface does not change. **Amended 2026-04-18:** M4 Batch C1 (pass-through env vending) dropped per operator + GAN reality-check — foreman as infrastructure does not call external APIs; pass-through provides zero security value vs the agent reading env directly. Promotion trigger shifts from M4 to Phase 2a (Vault/HSM provides real scoping + revocation + audit). See ADR 0007 (domain-blind core).

### Why `status` stays separate

`status` is the only read operation. CQRS separation is a pattern foreman already adopted (architecture §1.1 pattern list). A read operation with no side effects belongs on its own tool for discoverability; collapsing it with `step` would require agents to encode "no-op writes" — wrong shape.

## Consequences

### Immediate (before any M3 code)

- **Spec §6 amendment** — replace 7-row tool table with 2-row table. Drop the Direction column. Add a pointer to ADR 0003 for rationale.
- **Spec §7 amendment** — drop `foreman resume --action` as a separate subcommand; cover via `foreman step --signal '{"type":"resume",...}'`. CLI M3 shipped surface = 3 operator subcommands (`init` + `status` + `step`) + 1 hidden `mcp` run-mode. `inspect` + `validate-playbook` remain on the spec §7 roadmap and land at **M5** post-2026-04-18 domain-blind pivot (ADR 0007) — originally M4-alongside-playbook-assembly, slipped with the PromptAssembler revert.
- **Spec §8.1 amendment** — retain the 7-element signal type enum as internal vocabulary; add a prefatory note that MCP tool granularity differs from signal-type granularity.
- **Spec §8.3 amendment** — adjust the "When thrown" column for `OPERATOR_APPROVAL_REQUIRED` to phrase the check as sequence-validity (prior journaled resume event) rather than identity-validity (operator caller identity).
- **Architecture §4.3 amendment** — rewrite the resume example so resume arrives via `step(type:'resume')`, still routed to `originNode`.
- **Architecture §5.2 amendment** — the `mcp/` layer registers 2 tools, not 7; handler dispatch happens inside foreman.
- **Foundations D-21 clarification** — explicit note that MCP tool names are transport affordances; semantic boundary is signal envelope + channel + journal event.

### M3 code (deferred; follows the amended spec)

- `packages/solo/src/handlers/plan.ts` and `handlers/initialize.ts` collapse into `handlers/step.ts`; `handlers/status.ts` stays.
- `packages/solo/src/mcp/adapter.ts` registers 2 tools (`foreman__status`, `foreman__step`).
- `packages/solo/src/types/signal.ts` splits `SignalSchema` into `StepSignalSchema` (7 write variants including `retry`) and `StatusSignalSchema` (read variant).
- `handlers/step.ts` handles `retry` specially: (1) reject if current node's last derived outcome is not `exhausted`; (2) append a journal event with `type: 'retry_reset'`, envelope `{actor, ts}`, and `data: { nodeId }` (no `reason` at M3 per v0.3 YAGNI; see §2.5 RetryPayload); (3) clear `evaluation` channel to `null` (explicit write supported by `channels.ts` replace reducer); (4) return success — does not invoke graph. Agent then sends a subsequent `plan` signal to continue.
- `handlers/step.ts` handles `resume` by constructing `new Command({resume: payload})` and calling `graph.invoke(command, config)`. Fail-routing for resume is NOT carried in this invoke-input Command (v0.7 fix — invoke-input Commands accept only `resume`); `fail` routing lives inside `graph/nodes/evaluate.ts` as a node-return `new Command({update: {journal}, goto: END})`. Build-time: `addNode('n_evaluate', ..., {ends: [END]})` registers END as a valid dynamic target.
- `handlers/step.ts` handles `work` on composite parents with a readiness pre-check (Option B, v0.7): when `state.kind === 'composite'`, read each child thread via `graph.getState(checkpointConfig(projectId, childNodeId))` (reusing the existing helper so LangGraph config shape stays in one authoritative location); if any child's `evaluation` is null, reject with `COMPOSITE_CHILD_NOT_READY`. Otherwise invoke the graph with the agent's consolidated `WorkReport` (the agent synthesizes children's outputs using `brief.playbook.phases.work` per D-18 playbook injection). Read-only cross-thread access for sequence-validity is explicitly permitted by D-21 — see foundations.md clarification note.
- `packages/solo/src/graph/nodes/evaluate.ts` is **kind-agnostic** (Option B, v0.7): it assesses whatever `WorkReport` the agent submitted, whether terminal or composite consolidation. Composite aggregation does NOT live in evaluate; it lives in step.ts's work pre-check above.
- `packages/solo/src/graph/retry-count.ts` gains retry_reset marker awareness: count `evaluate` events after the latest retry_reset marker for this nodeId; fall back to total-count if no marker exists.
- CLI `packages/solo/src/cli/` gains a `step` subcommand; `resume` subcommand is not added.
- Integration tests for the 2-tool surface replace the enumerated per-signal tests. Retry scenario test: initialize → plan → work → evaluate (fail) × 3 → exhausted → retry → plan → work → evaluate → pass.

### Phase 2a and beyond

- The daemon/team/enterprise wrapper owns auth and stamps `actor` before forwarding.
- Multi-tenancy via thread_id prefix (ADR 0002 Consequences): `${tenantId}:${projectId}:${nodeId}`.
- No foreman-layer changes required to support multi-user; the state machine stays single-shape.

## Alternatives Considered

### Alt 1: Retain 7-tool surface (status quo)

**Rejected.** Encodes actor partition at the transport layer. Skill YAMLs and agent instructions would reference `foreman__plan`, `foreman__work`, `foreman__resume` individually; renaming them later becomes a breaking change across the skill ecosystem.

### Alt 2: 5-tool surface (Copilot round 2)

**Rejected.** The security rationale (prevent agent self-approval) presumes foreman enforces actor identity. Under the scope doctrine adopted here, it doesn't — the wrapper does. Partitioning tools by actor provides no enforcement and adds surface area.

### Alt 3: 4-tool surface (Codex round 1; init + status + step + resume)

**Rejected.** Same reason as Alt 2 — `resume` partitioning is security-motivated but foreman never enforces identity. The initialize asymmetry was also cited as a reason to keep init separate; this ADR locates that asymmetry inside the step handler as a state-guard branch.

### Alt 4 (chosen): 2 tools (status + step) + wrapper-owns-auth

**Adopted.** Clean state-machine framing; internal signal vocabulary preserved; authentication boundary moved to the wrapper where it belongs at every phase.

### Alt 5: `retry` as an action inside `resume`'s union (round-3 proposal, α+β hybrid)

**Rejected.** Round 3 considered folding capped-intervention into `resume` by adding a 4th action value:

```ts
resume: {action: 'approve'|'override'|'fail'|'retry', ...}
```

Rejected because it conflates two different semantic axes — resume's actions are _resolution decisions_ for block (continue with approval / abandon); "retry" would be a _re-entry directive_ for capped (restart here). One signal type carrying both made sequence-validity checks per-action rather than per-type, and mixed LangGraph's `Command({resume})` primitive (block) with handler state-preparation (capped). Round 4 resolved by splitting `retry` to its own signal type.

### Alt 6: `retry` with `target: 'plan' | 'work'` discriminator (round-4 α+β proposal)

**Rejected for M3; kept as a forward-compat possibility for a future ADR.** Round 4 considered `RetryPayload = z.discriminatedUnion('target', [plan, work])` so operators could choose to keep the existing plan and only re-work (target=work) or fully replan (target=plan). Both Codex and Copilot converged on α-only (target=plan) because: (a) the common operator case is replan-then-work; (b) composite parents have no meaningful "work" phase, so target=work needs per-composite rejection logic; (c) the one-extra-round-trip cost of α-only is acceptable. Future ADR can add `target` if dogfooding surfaces frequent rework-only demand.

## Verification (before M3 code starts)

### C3 spike: MCP-host rendering of polymorphic `step`

A polymorphic `step` tool exposes a discriminated-union `inputSchema`. Some MCP hosts render each tool as a distinct palette entry with a simple form; a discriminated union might render poorly or not at all depending on the host.

**Spike protocol:**

1. Write a throwaway MCP server with one tool whose `inputSchema` is a JSON-Schema `oneOf` keyed on a `type` discriminator.
2. Connect from Claude Desktop, then from at least one other MCP host (Cursor, Zed, etc.).
3. Attempt to invoke the tool with each discriminated variant. Confirm the host surfaces the discrimination.
4. If any major host cannot render the polymorphic tool usably: this ADR falls back to a 4-tool shape (init + status + step-for-{plan,work,eval} + resume), noted as **Amendment A** in a later revision of this ADR.

The spike artifact lives at `packages/solo/scripts/c3-spike.ts` (throwaway, kept for auditability and future re-runs). Results are recorded in Appendix A below.

## Appendix A — C3 spike results (2026-04-18)

### Protocol-level verification: **PASS**

Script: `packages/solo/scripts/c3-spike.ts`. Run via `pnpm --filter @foreman-lab/solo exec tsx scripts/c3-spike.ts`.

Method: single-process in-memory MCP client+server pair via `InMemoryTransport.createLinkedPair()`. Server registers 2 tools (`foreman__status`, `foreman__step`); `foreman__step.inputSchema` is a JSON-Schema object with a `oneOf` array of 7 variants discriminated on `type` (initialize, plan, work, eval, resume, retry, token_request).

Verdict on each check:

- `tools/list` returns 2 tools — **PASS** (count=2, both names correct)
- `foreman__step.inputSchema.oneOf` is array of 7 — **PASS** (schema survives client retrieval intact)
- `tools/call foreman__step` with each of the 7 variant types — **PASS** (all 7 round-trip cleanly; server receives correct `type` discriminator)

**Protocol-level conclusion:** The MCP SDK + JSON-Schema `oneOf` shape works end-to-end. 2-tool polymorphic surface is protocol-valid. No Amendment A needed on this ground.

### Host-rendering verification: **pending manual check**

The in-memory spike does not verify how specific MCP host apps (Claude Desktop, Cursor, Zed, etc.) render a polymorphic tool in their UI palette. Hosts may collapse the `oneOf` into an unusable form, hide variant discrimination, or require manual JSON entry. This is a UX-quality question, not a protocol-validity question.

**To run the manual check (operator-executable):**

1. Convert the spike from `InMemoryTransport` to `StdioServerTransport` (approx. 10-line change).
2. Configure Claude Desktop: add to `claude_desktop_config.json`:
   ```json
   {
     "mcpServers": {
       "foreman-c3-spike": {
         "command": "node",
         "args": ["<absolute-path-to>/dist/c3-spike.js"]
       }
     }
   }
   ```
3. Restart Claude Desktop. Confirm `foreman__step` appears in the tool palette with schema detail sufficient to select a variant (`type`) and fill its payload.
4. Invoke the tool with each variant; confirm the UX is usable (ideal: variant picker auto-adjusts fields; acceptable: manual JSON entry with variant hint visible).
5. Repeat for Cursor (configure `.cursor/mcp.json`), and any other target host.
6. Record findings as an Appendix B addition to this ADR.

**Gate for M3 code:** protocol-level PASS is sufficient to begin M3 implementation. Host-rendering may surface UX refinements (e.g., add CLI-sugar subcommands for common step variants, or supply specialized hints in the `description` string of each `oneOf` variant) but these are additive and do not invalidate the 2-tool surface.

**Fallback (Amendment A):** preserved for future use. If host-rendering turns out to be broadly unusable and cannot be mitigated via description hints, a later ADR revision falls back to 4 tools: `foreman__status`, `foreman__step` (plan/work/eval only), `foreman__resume`, `foreman__retry`. The handler layer (unified `step.ts`) remains unchanged; only `mcp/adapter.ts` tool registration differs.

## References

- ADR 0001 — Architectural style declaration (7-pattern hybrid including CQRS + FSM)
- ADR 0002 — LangGraph 1.x adoption + composite thread_id
- Foundations D-21 — harness-identity + protocol-integrity invariant (semantic transitions via validated signals, channels, journaled events)
- Spec §6 / §8.1 / §8.3 — MCP surface + signal catalog + error codes (amended by this ADR)
- Architecture §4.3 / §5.2 — interrupt/resume flow + layer boundaries (amended)
- M2 development proposal v8 §3.2, D-M2-9 — the 3-tool M2 shipped surface (now superseded at M3)
