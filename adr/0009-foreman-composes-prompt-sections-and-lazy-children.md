---
status: Draft v0.1 (2026-04-20) — authored after mid-M5 re-examination of ADR 0007; restoring proven design from earlier foreman codebase commit 341bf34 (Sprint 6, 2026-04-15); pending Codex/Copilot GAN rounds before lock
date: 2026-04-20
deciders: Operator (xuzhijie), pending GAN concurrence
supersedes_in_part:
  - docs/adr/0007-domain-blind-core.md § "foreman does not render prompts at runtime" clause (this ADR distinguishes *composition* from *interpretation*; composition is restored to foreman; interpretation stays out)
amends:
  - docs/adr/0003-state-machine-scope-and-mcp-surface.md (status response envelope grows a `prompt` field; init phase becomes a real FSM state with a non-null composable prompt)
  - docs/adr/0004-composite-independent-threads.md (cross-thread read now ALSO serves parent→child plan propagation during child prompt composition, not just parent's work-phase consolidation)
  - docs/foundations.md §D-20 (prompt composition is a pure function with no outbound ports — D-20 principle preserved; new seams are function parameters, not ports)
related:
  - docs/phase-1-solo/architecture.md §3 (signal protocol), §4 (graph/handlers/prompt tier rules)
  - docs/phase-1-solo/m5-development-proposal.md (Batch D scenarios must cover composed-prompt flow + lazy-child loading)
prior_art:
  - earlier foreman codebase at git 341bf34289d42e88bd8b202dd6816a5c625caa9b (Sprint 6; 2026-04-15) — `src/prompt/` module with `compilePrompt()`, 6-section envelope, `classifyTrajectory()`, `assembleEvalRules()`, `buildEvidenceGraph()`. This ADR restores that design into the current redevelopment with renamed terminology (Playbook / Phases / Scope). Note: `trimmer.ts` from that codebase is NOT restored — see D9.
  - https://www.anthropic.com/engineering/harness-design-long-running-apps — 6-layer prompt priority pattern (Operator layer, envelope, orientation, methodology, derivation, footer)
---

# ADR 0009: Foreman composes prompt sections and supports lazy composite children

## Status and scope

**Draft.** Captures the mid-M5 design re-examination. Operator-confirmed on 2026-04-20 after reviewing the earlier foreman codebase (341bf34 Sprint 6) and comparing to current post-pivot design. ADR 0007 retired a flawed prompt-renderer (`PromptAssemblerPort` with variable interpolation inside graph nodes and a pluggable port abstraction over a single adapter). ADR 0009 restores the EARLIER — and better — design: foreman composes structured prompt *sections* from state + playbook + contract shape; agents map sections to their runtime's LLM API surface. Composition ≠ interpretation. Domain-blindness is preserved.

**Scope:**
1. The shape and ownership of the `foreman__status` response (introduces a structured `prompt` field).
2. The lifecycle of the `init` FSM phase (now a real state with a composable prompt; no separate bootstrap).
3. The composite `plan` signal contract (proposal + `children[]` metadata).
4. How a composite parent's plan propagates down to each child's phase prompts (cross-thread read).
5. How a lazy-initialized child discovers its own `init` prompt before its playbook is loaded.
6. Output-gate eval rules as part of the composed sections at evaluate-phase.

**Out of scope:**
- LangGraph engine choice (ADR 0002 stands).
- 2-tool MCP surface (ADR 0003 stands as to the tool count; only the response envelope shape is amended).
- Composite thread isolation (ADR 0004 stands as the per-child thread model; this ADR adds a parent→child read direction).
- Playbook YAML format (ADR 0005 stands; the file format doesn't change, only how its `phases` content flows into composed sections).

## Context

### The pivot overcorrected

ADR 0007 retired `PromptAssemblerPort` for good reasons (interpolation ambiguity, port abstraction over a single adapter, "rendering-as-domain-knowledge" leak). It then over-corrected by banning **all** foreman-side prompt work and shifting assembly fully to the agent. The operator noted 2026-04-20: *"even the harness plugin predecessor did more than we do now."*

### The proven design was already in foreman's own history

Commit 341bf34 (Sprint 6 of the pre-redevelopment codebase, 2026-04-15) shipped a mature prompt module:

```
src/prompt/
  compiler.ts     -- compilePrompt(input) → 6 sections
  seed.ts         -- resolveSeedSkill() for init-phase default
  trimmer.ts      -- context-hygiene trim pass on trimmable sections
  evidence.ts     -- buildEvidenceGraph() for composite-evaluate
  eval-rules.ts   -- assembleEvalRules() — output-gate rules
  trajectory.ts   -- classifyTrajectory() from priorEvals
```

That codebase's architecture doc §8 "Prompt Assembly" declared the prompt foreman's **primary product** and returned *sections*, not a rendered string — leaving LLM-API mapping to the agent wrapper. §14.2 placed `prompt/` as a sibling to `core/` (FSM / scheduling / outcome) with both depending only on `types/`. 351 tests green.

The six sections (locked §8.1 at 341bf34):

| # | Section | Source |
|---|---------|--------|
| 1 | envelope | foreman invariants (fixed protocol preamble) |
| 2 | operator | `config.operatorInstructions` |
| 3 | orientation | state + playbook.identity + node position |
| 4 | methodology | `playbook.phases[phase]` + playbook.instructions |
| 5 | derivation | prior proposals, reports, evals, trajectory, sibling summaries |
| 6 | footer | signal schema (contract) + tool policy |

(The 341bf34 table annotated some sections as "trimmable" to indicate where a future trimmer would act; per D9 below, foreman no longer ships a trimmer. All sections are produced in full; rendering-time truncation is the agent runtime's responsibility.)

### Alignment with Anthropic research

The 6-section design maps onto Anthropic's "harness design for long-running apps" article: a fixed **envelope** (protocol) + layered context (orientation / operator / footer for invariant framing; phases / derivation for per-iteration working memory). Foreman stores all of it; the agent decides how to render and — if its runtime has context-window pressure — how to truncate at render time. See D9 for why foreman itself does not trim.

### What needs to change post-pivot

The 341bf34 design used the old "skill" vocabulary and lacked an explicit treatment of composite children beyond a flat `children[]` on the parent. It also predated ADR 0004 (independent-threads). The restoration needs four adjustments:

1. Rename `methodology` as a content source to **`playbook.phases`** (already done across the codebase via the C1–C3 sweeps).
2. Make the `init` FSM phase a real state that also emits composed sections (no separate bootstrap phase).
3. Declare how a composite parent's plan proposal reaches each child's composed sections under ADR 0004's independent-threads model.
4. Declare how a child whose playbook hasn't been loaded yet receives an `init` prompt telling the agent what shape of playbook to submit.

## Target flows

Before enumerating the decisions (D1..D9), this section fixes the end-to-end behaviour we're designing toward. D1..D9 below each contribute a specific mechanism; these diagrams show how they compose at runtime. Any gap discussion after this ADR lands should reference these flows first.

### Flow A — Terminal node full lifecycle

Single terminal node from `foreman init` through to `pass`. The agent drives `loop(status → step)`; foreman composes the next prompt on each status call.

```mermaid
sequenceDiagram
  autonumber
  actor hm as Human
  actor ag as Agent (runtime)
  participant fr as Foreman
  participant db as SqliteSaver

  hm->>ag: "help me with X"
  ag->>ag: load generic foreman command/skill
  ag->>fr: foreman init (CLI)
  fr->>db: create .foreman/checkpoints.db
  fr-->>ag: ok

  rect rgba(255,235,200,0.3)
  Note over ag,fr: Phase 'init' — no playbook yet. Foreman still composes a prompt<br/>(built-in directive describing the expected initialize signal shape).

  ag->>fr: foreman__status  (no --node → defaults to root)
  fr->>fr: composePrompt(phase='init', playbook=null,<br/>state={}, contract=InitializeSignalSchema)
  fr-->>ag: { nodeId:'root', phase:'init', childNodeIds:[],<br/>prompt:<init sections — discover goal, pick playbook,<br/>submit initialize with brief.playbook> }

  ag->>hm: (optional) clarify goal
  hm-->>ag: clarified
  ag->>ag: load agent-side skill (embeds terminal playbook)
  ag->>fr: foreman__step { type:'initialize', nodeId:'root',<br/>payload:{ brief:{ goal, playbook:<terminal>, parentNodeId:null }}}
  fr->>db: persist brief; transition init → plan
  fr-->>ag: ok
  end

  rect rgba(200,230,255,0.25)
  Note over ag,db: Per-phase loop — each status returns the composed sections<br/>for the current phase; agent returns contract-shaped artifacts.

  ag->>fr: foreman__status --node root
  fr->>fr: composePrompt(phase='plan', playbook.phases.plan,<br/>state={brief}, contract=PlanSignalSchema)
  fr-->>ag: { nodeId:'root', phase:'plan', childNodeIds:[], prompt:<plan sections> }
  ag->>ag: execute → produce proposal
  ag->>fr: foreman__step { type:'plan', kind:'terminal',<br/>nodeId:'root', payload:{ proposal }}
  fr-->>ag: ok (→ 'work')

  ag->>fr: foreman__status --node root
  fr-->>ag: { nodeId:'root', phase:'work', childNodeIds:[], prompt:<work sections><br/>(derivation carries the proposal from prior round) }
  ag->>fr: foreman__step { type:'work', payload:{ report, toolCalls }}
  fr-->>ag: ok (→ 'evaluate')

  ag->>fr: foreman__status --node root
  fr->>fr: composePrompt(phase='evaluate', playbook.phases.evaluate,<br/>state={brief, plan, work}, contract=EvaluateSignalSchema);<br/>inject eval-rules into derivation (D6)
  fr-->>ag: { nodeId:'root', phase:'evaluate', childNodeIds:[], prompt:<evaluate sections><br/>(derivation carries proposal + work + eval-rules list) }
  ag->>fr: foreman__step { type:'evaluate', payload:{ findings, meetsCriteria }}
  fr->>fr: deriveOutcome (I-7 truth table) → 'pass'
  fr-->>ag: ok
  end

  ag->>fr: foreman__status --node root
  fr-->>ag: { nodeId:'root', phase:'pass', childNodeIds:[], prompt:null }
  Note over ag: loop exits on terminal phase
```

### Flow B — Composite parent with lazy-loaded children

Composite parent decomposes into two children declared as metadata; each child lazy-loads its own playbook via `status → init-prompt → initialize`. Parent's plan proposal auto-propagates into every child's prompt via cross-thread read (D5).

```mermaid
sequenceDiagram
  autonumber
  actor ag as Agent
  participant fr as Foreman
  participant pt as parent thread<br/>(thread_id: projectId:root)
  participant c1 as child-1 thread<br/>(thread_id: projectId:c1)
  participant c2 as child-2 thread<br/>(thread_id: projectId:c2)
  participant db as SqliteSaver

  Note over ag,fr: Parent already initialized with a COMPOSITE playbook<br/>(same bootstrap as Flow A; initialize signal carried kind:composite).

  rect rgba(255,240,230,0.3)
  Note over ag,pt: Parent plan phase — produce proposal + declare children metadata

  ag->>fr: foreman__status --node root
  fr->>pt: getState
  fr->>fr: composePrompt(phase='plan', parentPlaybook.phases.plan,<br/>state={brief},<br/>contract=CompositePlanSignalSchema)
  fr-->>ag: { nodeId:'root', phase:'plan', childNodeIds:[], prompt:<plan sections —<br/>"produce architecture doc + decomposition instructions<br/>+ children[] metadata"> }

  ag->>fr: foreman__step { type:'plan', kind:'composite', nodeId:'root',<br/>payload:{<br/>  proposal:{ architecture, decompositionInstructions, ... },<br/>  children:[ {nodeId:'c1', name, responsibility},<br/>             {nodeId:'c2', name, responsibility} ]<br/>}}
  fr->>pt: persist plan (proposal + children metadata);<br/>transition → 'work'
  pt->>db: checkpoint
  fr-->>ag: ok
  end

  ag->>fr: foreman__status --node root
  fr-->>ag: { nodeId:'root', phase:'work', childNodeIds:['c1','c2'],<br/>prompt:<work sections — "drive each child to completion, then consolidate"> }

  Note over ag,fr: Agent owns scheduling across children (may parallelize;<br/>diagram shows sequential for clarity).

  rect rgba(220,240,220,0.3)
  Note over ag,c1: Child c1 lazy-load — status returns init-prompt<br/>enriched with parent-declared responsibility

  ag->>fr: foreman__status --node c1
  fr->>c1: getState (empty — thread not yet initialized)
  fr->>pt: getState (cross-thread) → find 'c1' entry in parent.plan.children[]
  fr->>fr: composePrompt(phase='init', playbook=null,<br/>state={},<br/>parentState={parent.plan.proposal, responsibility:<c1>},<br/>contract=InitializeSignalSchema)
  fr-->>ag: { nodeId:'c1', phase:'init', childNodeIds:[], prompt:<init sections —<br/>orientation names parent + responsibility;<br/>derivation carries parent.plan.proposal inherited;<br/>phases section is built-in init directive> }

  ag->>ag: load agent-side skill matching responsibility → extract playbook
  ag->>fr: foreman__step { type:'initialize', nodeId:'c1',<br/>payload:{ brief:{ goal, playbook:<chosen>, parentNodeId:'root' }}}
  fr->>c1: persist brief; transition init → plan
  fr-->>ag: ok
  end

  Note over ag,c1: c1 now drives its own loop(status→step). Every c1 status call<br/>composes with cross-thread read of parent.plan — derivation always<br/>carries the parent's architecture + decomposition as inherited context.<br/>If c1's playbook is composite, it decomposes further (recursive).

  Note over ag,c2: Same lazy-load pattern for c2.

  rect rgba(230,220,240,0.3)
  Note over ag,pt: Parent work phase — consolidate after all children terminal (pass)

  ag->>fr: foreman__status --node root
  fr->>pt: getState (parent)
  fr->>c1: getState (cross-thread — child state for consolidation)
  fr->>c2: getState (cross-thread)
  fr->>fr: pre-check: every child has terminal phase 'pass' (or terminal otherwise);<br/>assemble children summaries into derivation;<br/>composePrompt(phase='work', parentPlaybook.phases.work,<br/>state={brief, plan}, childSummaries={c1, c2},<br/>contract=WorkSignalSchema)
  fr-->>ag: { nodeId:'root', phase:'work', childNodeIds:['c1','c2'], prompt:<work sections —<br/>derivation carries parent.plan.proposal + c1 state + c2 state> }
  ag->>fr: foreman__step { type:'work', nodeId:'root', payload:{ consolidationReport }}
  fr-->>ag: ok (→ 'evaluate')
  end

  Note over ag,pt: Parent evaluate proceeds like terminal evaluate<br/>but with composite quality rules added to eval-rules (D6).
```

### Flow C (fragment) — Parent re-plan invalidates children (D8)

Shown as a small delta, not a full diagram:

```
(parent was in 'evaluate'; outcome derives to 'retry')
parent transitions evaluate → plan (existing retry semantics)
agent foreman__status --node root  →  phase='plan' (fresh plan prompt)
agent foreman__step { type:'plan', kind:'composite', payload:{ new proposal + new children }}

foreman, on persisting new plan:
  for each nodeId in OLD parent.plan.children[]:
    mark that child's thread with outcome 'invalidated' (terminal)
  new children from NEW parent.plan.children[] spawn on fresh threads via lazy-load

agent foreman__status --node <old-c1>  →  phase='invalidated', prompt=null
agent foreman__status --node <new-c1>  →  phase='init', prompt=<init sections> (fresh)
```

Grandchildren (children of an invalidated child) are NOT auto-cascade-invalidated. Their composition reads parent=invalidated and the composed `derivation` carries a STOP directive; the agent decides whether to kill them or let them finish. See D8 for rationale.

---

## Decision

### D1. `foreman__status` returns a minimal, navigation-complete response

The response surface is factored to the four fields an agent actually needs to drive the loop, nothing more. Structured artifacts (brief, plan, work, evaluation) live in foreman's internal channels and are surfaced via `foreman__inspect` for debugging — NOT duplicated in the status envelope. Their content is already woven into `prompt.derivation` as prose for LLM consumption.

```ts
type StatusResponse = {
  nodeId: string;  // echo — confirms which node this response is about

  phase:
    | 'init' | 'plan' | 'work' | 'evaluate'            // active lifecycle
    | 'pass' | 'block' | 'exhausted' | 'invalidated';  // terminal outcomes

  childNodeIds: string[];  // empty unless composite + post-plan;
                           // non-empty: IDs the agent can call status on next

  prompt: PromptSections | null;  // null iff phase is terminal
};

type PromptSections = {
  envelope: string;       // foreman protocol preamble
  operator: string;       // operator-injected cross-cutting instructions
  orientation: string;    // node position + objective + persona
  phases: string;         // playbook.phases[currentPhase] + playbook.instructions
  derivation: string;     // prior artifacts + trajectory + sibling context
  footer: string;         // signal schema contract + tool-call policy
};
```

**Field justifications:**

| Field | Why it's in the response |
|---|---|
| `nodeId` | Agent omitting `--node` gets root by default; echoing back the resolved nodeId removes ambiguity and lets the agent loop on the same id without state |
| `phase` | Agent uses it to detect terminal (exit loop) and to pick the next signal type (phase name IS the signal type for active phases) |
| `childNodeIds` | Composite parent's children are foreman-knowledge; agent navigates to each child by ID. Only IDs, no structured metadata (metadata lives in prompt.derivation + inspect) |
| `prompt` | The composed next-step instruction — foreman's primary product |

**Fields explicitly NOT in the response** (available via `foreman__inspect`, composed into `prompt.derivation`, or held in the brief that foreman already validated):

| Not-a-field | Where it lives instead |
|---|---|
| `brief` | Submitted via `initialize`; persisted in channels; content surfaces in `prompt.orientation` + `prompt.phases` |
| `plan` / `work` / `evaluation` | Persisted in channels; content surfaces in `prompt.derivation` as prose; structured access via `foreman__inspect` |
| `parentNodeId` | Part of `brief`; parent context auto-propagates into `prompt.derivation` via cross-thread read (D5) |
| `terminal: boolean` / `outcome` | Collapsed into `phase` — terminal phases ARE the outcome values |
| Full child summaries | In `plan.children[]` (inspect); composed into prompt.derivation for agent consumption |

**Root nodeId convention (R1):** the root of a project's node tree has the constant nodeId `'root'`. `foreman__status` called with no `--node` defaults to `'root'`. This is a fixed convention — no operator choice at `foreman init` time; no dynamic root naming. Multi-root support is a Phase 2a+ concern.

**Agent loop shape:**

```
node = 'root'                                  # or a child id passed from a parent's status
loop:
  r = foreman__status --node node
  if r.phase in {pass, block, exhausted, invalidated}:
    break
  execute(r.prompt)                            # LLM call, produces artifacts
  foreman__step --signal {type: r.phase, nodeId: r.nodeId, payload: artifacts}
  # for composites, agent may then enter sub-loops on each r.childNodeIds[i]
```

**Section naming note:** the previous codebase named section 4 `methodology`. The C2 rename ships `playbook.phases`, so the section was renamed `phases` for consistency. The operator has flagged this collides with the FSM `phase` discriminator (same word, two meanings); a follow-up rename to `guidance` (both the playbook field and this section) is pending operator pick.

**Why sections and not a rendered string:** the agent wrapper maps sections to its runtime's LLM API shape (Claude Code may use `system` for envelope/operator/orientation and `user` for phases/derivation/footer; Codex CLI may wrap differently; Copilot CLI likewise). Foreman does not know about LLM APIs (P-1 preserved); the 6-section contract is the neutral hand-off.

### D2. `init` is a first-class FSM phase with a composable prompt

Current code: `init.ts` node validates the brief submitted via `initialize` signal. Phase transitions init→plan on validation success. Status before init returns empty state.

**New behavior:** the FSM sits in `init` phase until an `initialize` signal arrives with `brief.playbook`. In that phase, `foreman__status` composes sections with:

- `envelope`, `operator`, `footer` — same fixed content
- `orientation` — node identity + goal (if supplied) + "awaiting playbook-bearing initialize signal"
- `phases` — foreman-built-in init directive: *"Determine the goal from operator/human context. Select a playbook that matches the goal's nature. Submit `foreman__step {type:'initialize', payload:{brief:{goal, playbook}}}`. Expected playbook shape: `{name, phases:{plan, work, evaluate}, scope?}`."*
- `derivation` — parent proposal if this is a child being lazy-loaded; empty otherwise
- `footer` — `InitializeSignalSchema` rendered as the contract

No separate "bootstrap" concept. The built-in init-phase directive IS the prompt. Foreman doesn't ship a default bootstrap playbook — the directive is a fixed string inside `src/prompt/compose-init.ts` describing the shape of the initialize signal. This is foreman's only "content" — the shape of its own input contract — and is defensible because it's the protocol, not the work.

### D3. Composite plan signal = terminal plan + `children[]` metadata

```ts
PlanSignalSchema = z.discriminatedUnion('kind', [
  // terminal
  z.object({
    type: z.literal('plan'),
    kind: z.literal('terminal'),
    nodeId: z.string().min(1),
    payload: z.object({ proposal: PlanProposalSchema }).strict(),
  }).strict(),

  // composite
  z.object({
    type: z.literal('plan'),
    kind: z.literal('composite'),
    nodeId: z.string().min(1),
    payload: z.object({
      proposal: PlanProposalSchema,   // architecture doc, decomposition instructions, etc.
      children: z.array(z.object({
        nodeId: z.string().min(1),
        name: z.string().min(1),
        responsibility: z.string().min(1),
        dependsOn: z.array(z.string()).optional(),
      }).strict()).min(1),
    }).strict(),
  }).strict(),
]);
```

Composite parent plan STILL produces its own `proposal` content (architecture doc, job decomposition instructions, whatever the parent's playbook calls for). It ADDITIONALLY declares `children[]` with per-child `{nodeId, name, responsibility, dependsOn?}` — **metadata only**. No child playbook at parent-plan time. Children lazy-load.

### D4. Children lazy-load playbooks via `status → init-sections → initialize`

When the agent calls `foreman__status --node <childId>` on an uninitialized child thread:
1. Foreman reads child thread state (empty → phase='init', brief=null).
2. Foreman reads parent thread state cross-thread (ADR 0004 Option B mechanics) and locates the child's entry in `parent.plan.children[]` by matching `nodeId`.
3. If the child is registered under a parent, foreman composes init-phase sections with:
   - `orientation` — child's `name`, `responsibility`, `parentNodeId`
   - `derivation` — parent's `plan.proposal` (architecture + decomposition) inherited as context
   - `phases` — built-in init directive (same as D2) but enriched: *"This is a child of parent `<id>`. Your responsibility: `<parent-declared>`. Load a playbook matching this responsibility. Submit initialize with `brief.parentNodeId=<parent>` to confirm lineage."*
4. Agent loads the matching agent-side skill (which embeds a playbook) and submits `foreman__step {type:'initialize', nodeId, payload:{brief:{goal, playbook, parentNodeId}}}`.
5. Foreman persists the child brief; transitions init→plan on that child thread.

Kind is discovered at child-init time — the child's `brief.kind` (terminal | composite) may differ from the parent's kind. A composite child decomposes further on its own plan phase; recursion is natural.

### D5. Parent plan propagates to children automatically via cross-thread read (P1)

During ANY child phase's section composition, foreman cross-thread-reads the parent's state and injects:

- `parent.plan.proposal` into the child's **derivation** section (top-most entry, labeled "INHERITED FROM PARENT")
- `parent.plan.children[otherChildId]` summaries (sibling context) into child's **derivation** when relevant phases are reached (evaluate especially)

The agent does not shuttle parent context. Foreman owns the parent↔child link via two signals:
- **Discovery:** parent's `plan.children[]` enumerates child nodeIds. Agent calling status on a listed child triggers lineage inference.
- **Confirmation:** child's `initialize.brief.parentNodeId` confirms the link. (C3 hybrid — both mechanisms present; confirmation wins if they disagree.)

### D6. Output gate: eval-rules compose into the footer/derivation at evaluate phase

Restores `assembleEvalRules(node, config)` from 341bf34. At evaluate phase, the `derivation` section includes an EVALUATION RULES list assembled from:
1. Always-on security invariants (no secrets, no injection patterns, scope-respecting).
2. Always-on authenticity invariants (verify claimed work exists; no rubber-stamping; proposal-drift check).
3. Composite-only quality rules (no interface mismatches across children; no completeness gaps in decomposition).
4. Playbook's own `phases.evaluate` text (appended as discipline-specific rules).
5. Node criteria (from brief).

The agent reads these as a checklist; the evaluate signal returns findings + `meetsCriteria`. Foreman doesn't interpret findings — it only checks the Zod shape and derives the outcome per the I-7 truth table. The eval-rules list itself is mechanical concatenation (rule #N templates + playbook/criteria text appended).

### D7. Code layout — prompt module is a sibling to graph/handlers

```
packages/solo/src/
  prompt/
    index.ts           -- barrel exports
    compose.ts         -- composePrompt({state, playbook, phase, contract, parentState?}) → PromptSections
    compose-init.ts    -- buildInitSections() with built-in directive
    evidence.ts        -- composite-evaluate evidence graph
    eval-rules.ts      -- assembleEvalRules()
    trajectory.ts      -- classifyTrajectory() from priorEvals
  graph/               -- unchanged
  handlers/
    status.ts          -- calls composePrompt() + cross-thread read when child
    step.ts            -- unchanged from C1/C2 shape
  types/               -- Playbook, Phases, Scope, Brief, signal types (already shipped)
```

(No `trimmer.ts`. See D9.)

Dependency rules (§14.2 of 341bf34 restored):
- `prompt/` imports **only** `types/` (and zod for schema-to-text). No `graph/`, no `handlers/`.
- `handlers/status.ts` is the only consumer of `prompt/`. It also calls `graph.getState(parentConfig)` for the cross-thread parent read.
- `graph/` still imports no `prompt/` (prompts compose AFTER the state is read, not inside nodes).

This preserves D-20 (no outbound ports; `prompt/` is a collection of pure functions consumed at the handler layer) and the layered-architecture discipline.

### D8. Parent re-plan auto-invalidates existing children

Composite parent re-plan happens when the parent's evaluate phase yields `retry` or the operator issues a `retry` signal on a capped parent. Re-planning clears the parent's `plan` / `work` / `evaluation` channels (per existing replace-reducer semantics) and transitions the parent back to `plan` phase.

**On the new plan signal arriving:** foreman iterates every child currently declared in the OLD `parent.plan.children[]` and marks each child's thread with `outcome = 'invalidated'` (new outcome value — see below). The child thread becomes terminal; no further signals are accepted on it. The child's journal is preserved verbatim (P-5 durability); cross-thread reads from future parents do NOT resurrect it (status on an invalidated child returns `{outcome:'invalidated', terminal:true, prompt:null}`).

Fresh children from the NEW `parent.plan.children[]` (even if a nodeId happens to coincide with a previously-invalidated one) spawn new threads following the normal lazy-load flow (D4) — foreman does not attempt to reuse the old thread's state.

**Why auto-invalidate (not selective):**
- Keeps the parent's plan proposal as the single source of truth for child responsibilities. If the parent's decomposition changed, even a child whose `nodeId` and `name` are unchanged may have a different `responsibility` — blending old progress with a new responsibility corrupts the child's derivation.
- Avoids the combinatorial matching problem (diff old-vs-new children by nodeId? by name? by responsibility? by dependsOn?). Auto-invalidate is atomic and unambiguous.
- Journal preserves old child state so operator can audit what was discarded and why.

**New outcome value:** `Outcome = 'pass' | 'retry' | 'block' | 'exhausted' | 'invalidated'` (added to `graph/outcome.ts` enum and derivation truth table). The `invalidated` branch is NOT a result of the child's own evaluate phase — it's set by the parent-re-plan handler. `deriveOutcome(input)` treats it as terminal.

**Cascade:** if an invalidated child was itself composite, its children (grandchildren of the re-planning parent) are NOT separately invalidated by foreman. Instead, they remain active on their threads, and status on them will read their own parent (the invalidated child) as invalidated; the grandchildren's own composition logic can detect the parent-invalidated state and decide to self-invalidate or continue. The default implementation: when composing a child's prompt sections, foreman reads the immediate parent's state; if parent's outcome is `invalidated`, the child's composed `derivation` section carries a STOP directive ("your parent has been invalidated; submit no further signals") and the `phases` section is empty. The child does not auto-mark its own outcome — the operator / agent decides whether to kill it or let it finish anyway. (This avoids runaway cascade invalidation sweeping the whole subtree; invalidation is opt-in per level.)

### D9. No trimming — foreman stores, agent renders

The 341bf34 codebase included `src/prompt/trimmer.ts` for context-hygiene truncation of the `methodology` and `derivation` sections under context-window pressure. ADR 0009 **does not restore the trimmer**.

**Reasoning:**
- Trimming is a rendering-time concern, not a storage-time concern. Foreman's job is to preserve the full composition; what the agent does when feeding it into an LLM context is the agent's problem.
- Storage is cheap. SqliteSaver + the `journal` append-reducer hold the full history. Dropping bytes at composition time loses audit trail and makes replay non-deterministic.
- Different agent runtimes have different context budgets. Claude Code with 1M-context has different trimming thresholds than a constrained runtime. Foreman choosing one policy for all runtimes would be wrong for most.
- Trimming introduces a side concern (token counting, section-priority ordering, what-to-drop-first heuristics) that doesn't belong in the domain-blind core.

**Consequence:** `derivation` grows monotonically as a node iterates (each retry adds eval history, trajectory recalculates, sibling context expands). On long-running composites the derivation section may be very large. The agent wrapper is expected to handle this — either by rendering only the latest N evals, or by summarizing prior rounds on its own side, or by relying on its runtime's automatic context management. Foreman emits everything; the agent decides.

**Future reconsideration:** if operator experience shows a specific pattern of agent-side truncation is universally applied (e.g., "every agent drops evals older than the last 3"), foreman may expose a *query parameter* on `foreman__status` like `--derivation-depth=3` that lets the agent tell foreman what to include. This remains a render-time hint, NOT a storage-time trim — state stays complete. Out of scope for this ADR; document when/if the pattern emerges.

### D10. Security gate — built-in system prompt template

Foreman does NOT know how to perform security checks. Security enforcement is **agent-performed**; foreman's contribution is a built-in system prompt template that tells the agent what to verify. Foreman ships no regex library, no secret scanner, no injection detector, no structural `securityChecks[]` verdict schema. The agent reads the composed prompt at evaluate phase, performs the checks, and reports results as ordinary **`Evaluation.findings[]`** (the shipped schema) — optionally tagged with a `ruleId` when the playbook declared one.

The sole structural security gate foreman enforces directly is the **scope allowlist** (already shipped at M5 Batch C): `Playbook.scope.paths` + `readOnly` → `PREFLIGHT_BLOCKED`. Scope is foreman's job because paths are directly observable without content interpretation.

**Built-in system prompt template (shipped with foreman, injected into `prompt.derivation` at evaluate phase):**

```
SECURITY REVIEW (verify each; report any failure as a finding with severity):
- No hardcoded secrets, API keys, tokens, or credentials in produced artifacts
- No command, SQL, XSS, or path-traversal patterns in generated code
- All file writes confined to declared scope (paths + readOnly)
- Authentication/authorization boundaries preserved (no privilege escalation in generated code)
- Irreversible tool calls (rm -rf, force push, destructive DB ops) justified in the proposal

Additional rules from this playbook's scope.security.rules:
  <playbook rules enumerated here>
```

**Playbook-level rule enumeration (optional, structured for rendering):**

```ts
Playbook.scope.security = {
  rules: Array<{
    id: string;               // e.g. 'sec-004'; agent may tag findings with this
    description: string;      // rendered verbatim into the prompt
    severity: 'info' | 'warning' | 'blocking';
  }>
};
// Optional; absence means "use only foreman's built-in template"
```

This schema is purely a **rendering input** — foreman lists each rule in the composed prompt and never consumes the agent's response structurally. The agent's findings may reference `ruleId` for operator traceability, but foreman does not validate or gate on that reference.

**Outcome consequences flow through existing `Finding.severity`:**
- Agent returns findings with severity `'info' | 'warning' | 'blocking'` (see D13 rename)
- Any `'blocking'` finding (security-related or otherwise) → `deriveOutcome` returns `'block'` per existing truth table
- No new outcome fields, no new schema
- If agent fails to perform the security review at all, that's detected by the absence of expected `findings[]` entries; foreman can't enforce this structurally but the playbook-declared rules appear unchecked in the agent's response prose

**Explicit rejections (see Alt-6 + Alt-7):**
- No foreman-side regex / pattern library
- No `securityChecks[]` verdict field in `Evaluation` schema
- No foreman interpretation of what "secret" or "injection" means
- No LLM call for semantic security review

### D11. Authenticity gate — built-in system prompt template

Authenticity follows the same delegation pattern as D10. Foreman does not compute authenticity dimensions; it ships a built-in prompt template naming the four dimensions to check, and the agent returns findings when any dimension fails. **No `authenticityGate{}` schema field, no AND-of-booleans in foreman.**

**Built-in system prompt template (shipped with foreman, injected into `prompt.derivation` at evaluate phase):**

```
AUTHENTICITY REVIEW (judge each dimension; if any fails, report a finding with
severity:'blocking' and category:'authenticity'):

- internal_consistency: do the artifacts share consistent conventions
  (structure, terminology, style) throughout the round?
- intentionality: is there evidence of project-specific decisions rather
  than unmodified defaults / generic boilerplate?
- craft: are the technical fundamentals correct for each artifact type
  (valid code, well-formed docs, proper test structure)?
- fitness_for_purpose: can the deliverables be used by the target audience
  without extra explanation?
```

**Agent behavior (contract by convention, not by schema):**
- If any dimension is judged to fail, the agent returns a finding with `severity: 'blocking'` and a category/description referencing the failing dimension
- If all four pass, the agent returns no authenticity-specific finding (absence = pass)
- Dimension failures surface as ordinary blocking findings; existing `deriveOutcome` converts them to `'block'` → operator resolves

**Why no structural `authenticityGate{}` field:**
- Making foreman read booleans means foreman would "know" what authenticity IS; even just AND'ing four booleans crosses the P-1 line
- Prose findings with severity give operator full traceability without foreman reasoning about content
- Matches foreman's "trust the agent's verdict; provide the template that frames the question" discipline

**No prevention checklist.** Agent-side pre-implementation authenticity discipline (if any) lives in the agent's skill, not in foreman's output. This is consistent with both the harness plugin and `~/Desktop/ai/harness` codebases: authenticity is a detection path, not a prevention path.

**No forced-completion escape.** `~/Desktop/ai/harness` allows terminal-round authenticity bypass via `isForcedCompletion`; foreman does not. A blocking finding remains blocking regardless of round number. Phase 2a may revisit if operator experience shows this is too strict.

### D12. Parser-fail = hard fail — malformed signals never silent-pass

Current shipped behavior: Zod schema validation in `handlers/step.ts` rejects malformed signals with `SCHEMA_VALIDATION_FAILED`. D12 elevates this to an explicit ADR-level rule because the harness codebase treats it as a primary safety primitive (`~/Desktop/ai/harness` `packages/orchestrator/src/evaluator-protocol.ts:71` — unparseable evaluator output fails the round).

**Rule:**
- Any `foreman__step` signal that fails Zod validation is rejected with `SCHEMA_VALIDATION_FAILED`; no state transition occurs; no outcome progresses.
- Specifically for the evaluate signal: if the agent returns a JSON blob that doesn't validate against `EvaluateSignalSchema` (including the new `securityChecks[]` + `authenticityGate{}` fields from D10/D11), the round is NOT considered evaluated. The node stays in the `evaluate` phase awaiting a correct signal.
- Retry counting does NOT increment on parser-fail — parser-fail is "you didn't answer," not "you answered wrong." Distinct from D11 authenticity-retry.
- Repeated parser-fails on the same phase (above a threshold, e.g. 3) trigger a distinct outcome `'malformed'` (new value) or fold into `'block'` requiring operator intervention.

**Why ADR-level:** without this, a subtly-malformed evaluation that partially validates could pass silently, masking gate failures. Zod's strict-mode + required-field discipline already protects against this; D12 names the protection so future schema evolution preserves it.

### D13. Severity tri-state — align finding and rule severity to `info | warning | blocking`

Today's shipped `Finding.severity` enum is `'info' | 'warning' | 'critical'` (`types/finding.ts`). The harness codebase uses `'info' | 'warning' | 'blocking'` (`packages/orchestrator/src/types.ts:30`) and the D10 security-rule severity vocabulary matches. Consistency matters — `critical` reads as "subjective critical bug" while `blocking` reads as "this is what gates the round." Rename to `'blocking'` across the codebase.

**Change:**
- `Finding.severity`: `'info' | 'warning' | 'critical'` → `'info' | 'warning' | 'blocking'`
- `SecurityRule.severity`: `'info' | 'warning' | 'blocking'` (new — matches)
- Any `hasCriticalFinding` internal check renames to `hasBlockingFinding` (or similar)
- Outcome truth table uses `'blocking'` not `'critical'` when evaluating "is there a finding that should surface as block"

Only `'blocking'` short-circuits outcome derivation. `'warning'` and `'info'` are recorded, visible to the agent in derivation, do not gate.

**Scope:** code rename (one more C-class sweep in the refactor series, call it C4) + doc rename. Small surface area.

## Consequences

### Code changes required

| Area | Change |
|---|---|
| `src/prompt/` (new) | Port 6 files from 341bf34, rename `methodology→phases` per C2 |
| `src/types/wire-response.ts` | `StatusSuccessSchema.data` grows `prompt: PromptSectionsSchema \| null` |
| `src/handlers/status.ts` | Call `composePrompt({state, playbook, phase, contract, parentState?})`; return sections under `prompt` field |
| `src/types/signal.ts` | `PlanSignalSchema` becomes `discriminatedUnion('kind', [terminal, composite])` |
| `src/graph/nodes/plan.ts` | Handle composite plan payload persisting `children[]` metadata into state |
| `src/types/brief.ts` | Add `parentNodeId: z.string().nullable()` (optional, defaults null) |
| `src/handlers/step.ts` | On initialize with parentNodeId, validate the link against parent.plan.children[] |
| `tests/contract/brief-exposure.test.ts` | Regression-guard assertions flip: `prompt` field is now EXPECTED to be non-null in non-terminal phases. The old assertion "no nextPrompt/assembledPrompt/promptView/projection" still holds — we emit `prompt` with the structured-sections shape, not any of the retired names. |
| `tests/integration/cli-inspect.test.ts` | Same regression-guard flip |
| `tests/unit/prompt-compose.test.ts` (new) | Port prompt-compiler tests from 341bf34 |
| `src/types/playbook.ts` | `Playbook.scope.security.rules[]` optional; `SecurityRuleSchema` added (D10 — rendering input only, not consumed structurally) |
| `src/types/finding.ts` | Rename severity `'critical'` → `'blocking'` (D13). Triggers a C4 sweep across tests + outcome derivation. |
| `src/prompt/security-template.ts` (new) | Built-in security system-prompt template injected at evaluate phase (D10) |
| `src/prompt/authenticity-template.ts` (new) | Built-in authenticity system-prompt template injected at evaluate phase (D11) |
| `src/prompt/compose.ts` | `derivation` section assembles playbook-declared security rules (if any) + injects built-in security + authenticity templates at evaluate phase |
| `src/graph/outcome.ts` | Blocking findings already produce `'block'` outcome; D10/D11 add NO new outcome-derivation logic — authenticity/security failures surface via ordinary `Finding.severity: 'blocking'` |
| `tests/unit/outcome.test.ts` | New case: finding with `category:'authenticity' + severity:'blocking'` → outcome 'block'. No new outcome branches. |
| **NOT changed:** `src/types/evaluation.ts` | Evaluation schema stays at today's shape (findings[] + meetsCriteria). No `securityChecks[]` or `authenticityGate{}` — walked back per operator correction 2026-04-20. |

### ADR 0007 amendment required

ADR 0007 currently reads: *"foreman does not render prompts at runtime."* This ADR clarifies that statement:
- **"Render"** (foreman does not) = LLM-API-specific shaping (system/user/tools). Agent does this.
- **"Compose"** (foreman does) = concatenate structured sections from state + playbook + contract. Mechanical, domain-blind.

ADR 0007's retirement rationale for `PromptAssemblerPort` stands verbatim — the port abstraction + interpolation was the wrong mechanism. `composePrompt()` is a plain function consumed at the handler layer, with no port ceremony. D-20 preserved.

### M5 impact

- **Batch D (8-scenario e2e) must cover:**
  - init-phase composed prompt (agent reads directive → submits initialize)
  - parent composite plan → child lazy-load with inherited derivation
  - per-phase composition for terminal flow
  - composite work-phase consolidation with child summaries in derivation
  - evaluate-phase eval-rules assembly for terminal + composite
  - recursion (composite child that decomposes further)
  - status on uninitialized-but-parent-listed child returns init-prompt
  - status on uninitialized-AND-unlisted child returns `NODE_NOT_FOUND`
- **Batch G (agent-setup) documents:** how agents map the 6 sections to their LLM API surface; reference section-to-role mapping table.

### Test-count impact

- +~40 tests for prompt composition (per-phase unit tests + section-by-section unit tests + cross-thread propagation integration)
- +~10 tests for lazy-child-load flow
- ~5 existing regression-guard tests flip their assertion direction

### Deprecation / revert

None from ADR 0007 is reverted — only *clarified*. The M4-era `PromptAssemblerPort` + `DefaultPromptAssembler` + `PromptView` + `view-mapping` + `nextPrompt` channel stay retired. The regression-guard tests asserting those five names do NOT appear in responses stay valid.

## Alternatives considered

### Alt-1: Keep ADR 0007 as-is; agent composes fully

Rejected. Leaves foreman less capable than its own predecessor. Forces every agent wrapper to re-implement section layout, context trimming, eval-rules assembly, trajectory classification. Breaks the Anthropic-harness-design alignment. Duplicates work across runtimes (Claude Code, Codex CLI, Copilot CLI).

### Alt-2: Foreman renders a single string prompt

Rejected. Couples foreman to a specific LLM API shape (system-vs-user split, tool-definition format). Violates P-1 (foreman gets into LLM-knowledge business). The 341bf34 design already solved this with sections as the contract; agent maps to runtime.

### Alt-3: Agent copies parent-plan into child brief (P2)

Considered. Simpler code (no cross-thread read at prompt composition). Rejected because:
- Duplicates source of truth (parent's plan becomes immutable snapshot in child brief).
- Breaks if parent re-plans (child's snapshot stale; no propagation).
- Agent carries context-shuttling discipline that foreman can handle authoritatively.

P1 (foreman cross-thread-reads) wins: parent is always source of truth; child always sees current parent.plan.

### Alt-4: New `load_playbook` signal distinct from `initialize`

Considered. Rejected because `initialize` already exists, already carries `brief`, and the brief already has `playbook` as a field. Adding a second signal for the same transition doubles API surface. Lazy-load works with `initialize` unchanged — only `brief.parentNodeId` is new.

### Alt-5: Eager child playbook declaration at parent plan

Considered. Parent declares full playbook per child in `plan.children[]`. Rejected because:
- Agent must choose playbooks at parent-plan time without any of the child's phase context (no status yet for child).
- Reduces autonomy — the agent can't reassess which skill best matches a child's responsibility when it gets there.
- Breaks the "lazy is good" principle that matches how humans actually decompose work.

Lazy-load wins.

### Alt-6: Foreman-side pattern detection for security (regex scanners)

Considered for D10. Foreman would run regex-based secret detection, injection pattern matching, and irreversibility classification on work signals before the evaluate phase. Rejected because:
- **Pattern library is domain content.** What counts as "a secret" or "an injection pattern" evolves with the threat landscape — keeping that knowledge inside the foreman runtime would require foreman to track it, contradicting P-1 and ADR 0007.
- **Harness precedent.** `~/Desktop/ai/harness` explicitly delegates all non-scope security checks to the evaluator agent. Same codebase proves the declarative-rules-plus-agent-verdicts model works and keeps the orchestrator pattern-free.
- **Cheaper structural defenses already exist.** Scope allowlist at preflight is foreman's direct boundary; additional pattern checks are best expressed as security rules the agent honors.

Foreman keeps pattern-free. Agents (Claude Code, Codex CLI, Copilot CLI) may run their own scanners as evaluator-side tooling; the `securityChecks[]` they return captures the verdicts.

### Alt-7: Foreman-side authenticity heuristics

Considered for D11. Foreman would apply statistical/structural heuristics (e.g., detect "empty findings + meetsCriteria:true repeatedly" as suspected rubber-stamping; diff proposal vs. report for drift). Rejected because:
- **Authenticity judgement is semantic.** No heuristic approximates "intentionality" or "craft" well; false positives would be disruptive and false negatives would rubber-stamp harder than the agent alone.
- **Harness precedent.** The codebase has ZERO authenticity heuristics; the 4 booleans come purely from the agent.
- **Tool-restriction is the structural defense.** The deeper article-aligned primitive is agent-layer role purity (evaluator with `tools: ['Read', 'Bash', 'Grep']`, no Write) — outside foreman's composition authority.

Foreman reads the four booleans the agent returns. If the agent runs with weak role purity, that's an agent-setup concern (documented in M5 Batch G `agent-setup.md`).

### Alt-8: Structural verdict schema for security + authenticity (`securityChecks[]` + `authenticityGate{}`)

Initially drafted for D10 + D11 based on the `~/Desktop/ai/harness` codebase pattern (agent asserts boolean/tri-state verdicts; foreman tallies severity and drives outcome). **Walked back 2026-04-20 per operator correction**: "security checks and authenticity gate are more like built-in system prompt templates for the prompt builder, because foreman should not know how to check these."

Rejected because:
- **Even AND'ing four booleans means foreman has a semantic model of "authenticity passes"**; the boolean is foreman-interpreted, not just foreman-passed-through. That crosses the P-1 line more than ADR 0007 + 0009 established. Domain-blindness is stricter here than in the harness codebase.
- **Ordinary findings with severity are sufficient.** A finding with `category:'authenticity'` + `severity:'blocking'` produces the same outcome derivation as a structural `authenticityGate.internal_consistency = false` — without foreman having to know what internal_consistency means.
- **Less schema churn.** Keeps `EvaluateSignalSchema` at its current shape; no new required fields; the walk-back preserves backward-compat for early playbook authors.
- **Stronger lineage.** The harness plugin (the domain-blind-core ancestor) uses *prompt-level directives* to frame evaluator behavior, not schema-level verdicts. Foreman matches the ancestor more closely by staying at prompt level.

What we adopt instead (per D10 + D11): foreman ships **built-in prompt templates** describing what to check; agent returns findings with severity; existing truth table produces outcomes. Playbooks may declare additional security rules for prompt rendering (`Playbook.scope.security.rules[]`), but foreman never consumes agent responses structurally against them.

## Prior art + inspiration

- **341bf34 foreman Sprint 6 (2026-04-15)** — the proven design this ADR restores. 351 tests green. See `src/prompt/compiler.ts` and `docs/architecture.md §8` at that commit.
- **Anthropic harness design for long-running apps** (https://www.anthropic.com/engineering/harness-design-long-running-apps) — 6-layer prompt priority + Operator layer + context provision from state.
- **Anthropic effective harnesses for long-running agents** (https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — skeptical-QA evaluator pattern; role-purity boundary (informs eval-rules gate).
- **Anthropic long-running Claude** (https://www.anthropic.com/research/long-running-Claude) — trimmable-vs-non-trimmable section policy for multi-day harnesses.
- **Harness plugin** (`~/.claude/plugins/marketplaces/harness`) — domain-blind core + domain skill suites; foreman is the principled evolution with FSM-in-code and sectioned-prompt-composition.

## Cross-references

- `docs/adr/0003-state-machine-scope-and-mcp-surface.md` — status response envelope amendment
- `docs/adr/0004-composite-independent-threads.md` — cross-thread read direction extended
- `docs/adr/0005-skill-file-format.md` — playbook file format unchanged
- `docs/adr/0007-domain-blind-core.md` — superseded-in-part; retirement of PromptAssemblerPort stands; "foreman does not render prompts" clarified to "foreman does not render, but DOES compose"
- `docs/foundations.md` §D-20 — narrow-port discipline; `prompt/` introduces no ports
- `docs/phase-1-solo/architecture.md` §3 (signal protocol), §4 (layer boundaries)
- `docs/phase-1-solo/m5-development-proposal.md` — Batch D scenarios expanded to cover composed-prompt flow + lazy-child loading

---

**Draft v0.1 done.** Awaiting operator enhancement (more detail on trimming policy, eval-rules expansion, trajectory thresholds, parent-plan-mutation semantics). After enhancement, dispatch Codex GAN rounds; lock at vN; then plan the M5 re-scoping.
