# Foreman Foundational Decisions

**Status:** Locked
**Scope:** Applies from Phase 1 onward across all tiers (Solo / Daemon / Team / Enterprise)

These decisions define what foreman IS at its core. They are expensive to change later and are made once, now, before implementation begins.

---

## Mission and quality stance

**Foreman is production software, not a proof-of-concept.** Every decision in this document is made to that bar.

- **Measurable gates, not aspirations.** Every NFR in `phase-1-solo/architecture.md` §8.1 has a concrete verification mechanism (CI job, scenario test, benchmark). "Fast enough" and "reasonably correct" are rejected phrases.
- **Enforceable invariants.** Every architectural invariant (I-1..I-9 in `phase-1-solo/architecture.md` §1.6) has a test. An invariant without a test is a comment.
- **Pinned dependencies.** Exact-version pin for `@langchain/*` and `@modelcontextprotocol/*` (D-7, R-1); `pnpm-lock.yaml` committed and CI uses `--frozen-lockfile` (R-15). No implicit upgrades.
- **Security by design from day 1.** S-1..S-5 are Phase 1 ship criteria (Q-8), not Phase 2+ polish.
- **No "ship it and fix it."** The 1.5× hard cutoff (`phase-1-solo/spec.md` §2.4) forces **scope cuts**, never **quality cuts**. Security tests, checkpoint integrity, MCP validation, and scenario tests are non-negotiable.
- **v1.0.0 is a commitment.** Phase 1 Solo ships at production-grade or it does not ship. The roadmap's 4-tier progression (Solo → Daemon → Team → Enterprise) adds features — it does **not** lower the quality bar.
- **Reproducibility as a test.** P-5 is enforced by determinism tests (Q-4), not by authorial promise.
- **Traceability end-to-end.** Every spec claim traces back through `Appendix B` of `phase-1-solo/architecture.md` + `Appendix A` of `phase-1-solo/spec.md` to a P/S/D or roadmap gate. Untraced requirements are design debt.

POC-grade shortcuts — skipped tests, ambient secrets, ad-hoc version bumps, "just for now" quality cuts, TBD sections in normative docs — are rejected by this stance. If a decision cannot be made to production-grade now, it is **deferred in writing** (see "Deferred Decisions" table), never vagueified.

**Implication for the Phase 1 → Phase 2+ transition:** the dogfooding constraint (`phase-1-solo/spec.md` §15) exists because foreman will be used by its own developers to build subsequent tiers. A POC-grade Phase 1 would compound quality debt across every later tier. A production-grade Phase 1 is the only version that self-hosts safely.

---

## 10 Principles

Ordered by importance: identity → trust → interface → product policy → business model.

### P-1. Pure infrastructure

Foreman never calls LLMs. Never interprets domain content. Never decides how work should be done.

Foreman is the harness. Agents are the contractors. Operators are the judges.

The moment foreman reads a proposal's content and reasons about it, foreman has become an agent — not an orchestrator. That boundary is non-negotiable.

### P-2. Three-actor model

Three roles, strictly separated:

- **Harness** (foreman) — state-machine authority. Owns channels + journal, validates signal sequences, enforces sequence-validity gates (e.g., irreversible `ToolCall` requires prior journaled approval event per spec §8.3). **Identity and authentication are not foreman's concern at any phase** — those are wrapper responsibilities (MCP stdio client at Phase 1; daemon at Phase 2a; team service at Phase 2b; enterprise backend at Phase 3+). See ADR 0003 for the wrapper-owns-auth doctrine.
- **Agent** (AI) — content executor. Produces plans, implementations, evaluations within the brief.
- **Operator** (human) — judge of irreversible decisions. Sets goals, creates playbooks, unblocks security, declares failure.

Cross-boundary actions are compile-time events (types), not runtime conventions.

### P-3. Security by design

The harness gates irreversible actions from Phase 1 onward. Five concrete requirements flow from this principle:

| #   | Requirement                                                                                                                                                        | What happens if violated                       |
| --- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ | ---------------------------------------------- |
| S-1 | **No ambient credentials** — harness never reads env vars on agents' behalf. Scoped tokens via operator approval.                                                  | Credential leak → account takeover             |
| S-2 | **Operator approval mandatory for irreversible/unknown actions.** No "skip approval" flag.                                                                         | Irreversible damage without consent            |
| S-3 | **Action classification** — every ToolCall carries `reversibility: "reversible" \| "irreversible" \| "unknown"` (Zod-validated). Unknown defaults to irreversible. | S-2 cannot be enforced                         |
| S-4 | **MCP transport input validation** — every inbound message validated against Zod before dispatch.                                                                  | Malformed/malicious inputs reach handlers      |
| S-5 | **Checkpointer-only state access** — agents interact with state exclusively through graph operations. No direct SQL, no raw journal writes.                        | Agent compromise escalates to state corruption |

### P-4. Agents are the primary consumer

Design for MCP + CLI dual access. MCP-first for agents. CLI for humans, CI, debugging.

Foreman's success depends on AI coding agents (Claude Code, Codex, Copilot) adopting it natively via MCP. Human-first interfaces would add weight without serving the primary user.

### P-5. Reproducibility

Identical inputs + state → identical action sequences.

No internal randomness. No wall-clock decisions. No implicit ordering.

Without this, debugging agent behavior is impossible and the journal replay (a core architectural bet) does not hold.

### P-6. Data portability

Every persistent artifact has an export format documented from the day it is introduced.

No "dark" data. No proprietary formats. Users can always leave. This is the trust foundation for OSS adoption.

### P-7. No telemetry by default

Opt-in only. Non-negotiable for an OSS framework that sees private repositories, prompts, and sensitive code.

Telemetry is a feature operators turn on, not a default they turn off.

### P-8. Clean breaks between phases

No automatic data migration. Phase N → N+1 is explicit (re-init or documented export/import).

By Phase 2, an export/import story exists for users who want to preserve work. Not before.

### P-9. Demand-driven, not schedule-driven

Concrete tripwires fire phases. Not calendar dates.

Tripwires are written down before Phase 1 ships. "We'll know demand when we see it" is procrastination-gating.

### P-10. Fully open source (Apache 2.0)

All four tiers are Apache 2.0. Monetize via hosted service and enterprise support contracts, not locked features.

Open-core is explicitly rejected: it creates perverse incentives (cripple OSS to sell EE) and damages community trust.

By Phase 2, the hosted-service moat is articulated in writing. If the moat cannot be defined by then, licensing is reconsidered.

---

## 20 Foundational Decisions (D-6 superseded)

Grouped into 4 tiers by impact:

- **Tier A** — Language & Runtime (affects every line)
- **Tier B** — Type Safety & Error Handling (affects every function signature)
- **Tier C** — Architecture Contracts (affects module boundaries)
- **Tier D** — Project Structure & Conventions (affects tooling)

---

### Tier A: Language & Runtime

#### D-1. TypeScript on Node.js 20+

The Node.js ecosystem is where AI coding agents live. Claude Code, Codex, Copilot are all Node.js-first. MCP SDKs target Node.js.

Node.js 20 is the minimum LTS with stable ESM, test runner, and modern features.

Alternatives rejected: Python (wrong ecosystem for MCP), Go (too removed from agent tooling), Deno/Bun (ecosystem immaturity, compatibility risk).

#### D-2. Async/await only

No callbacks. No event emitters for control flow. Events are used only for observability (emitting "something happened") — never for orchestration.

Mixing async paradigms creates impedance mismatch. Pick one, enforce everywhere.

---

### Tier B: Type Safety & Error Handling

#### D-3. Zod schema-first; types via `z.infer`

Schemas are the source of truth. TypeScript types are derived from schemas, never hand-written in parallel.

```typescript
// Always this:
const UserSchema = z.object({ id: z.string(), email: z.string().email() });
type User = z.infer<typeof UserSchema>;

// Never this:
interface User { id: string; email: string; }
const UserSchema = z.object({ ... }); // drifts from interface
```

#### D-4. throw + typed errors internally

Functions throw `BaseError` (or a subclass — see D-5) on failure. No Result types. No discriminated unions in internal APIs.

At the protocol boundary (MCP / CLI responses), a single catch converts exceptions to the wire-level response format:

```typescript
// Wire format only — not in internal code
type WireResponse<T> =
  | { ok: true; data: T }
  | {
      ok: false;
      error: { code: ErrorCode; message: string; details?: unknown };
    };
```

Internal code stays clean. Protocol contract stays explicit.

#### D-5. Error taxonomy

Four error classes, `BaseError` is the base:

| Class             | When to throw                                                    |
| ----------------- | ---------------------------------------------------------------- |
| `BaseError`       | Base — generic or uncategorized failure                          |
| `ValidationError` | Schema validation failure, malformed input                       |
| `SecurityError`   | Unauthorized action, missing operator approval, credential issue |
| `StorageError`    | Database failure, corruption, constraint violation               |

Each carries a `code` (SCREAMING_SNAKE) and optional `details` payload. The taxonomy is extended only by explicit decision, not ad-hoc.

---

### Tier C: Architecture Contracts

#### D-6. [Superseded by D-16] StorageAdapter interface

Replaced by LangGraph's `BaseCheckpointSaver`. State management is now owned by the flow engine, not a bespoke adapter. See D-16. Numbering preserved for historical reference.

#### D-7. MCP SDK wrapped in a single adapter directory

`@modelcontextprotocol/sdk` is pre-1.0. Protocol churn is expected.

All interaction with the MCP SDK lives in one directory (e.g., `packages/solo/src/mcp/adapter.ts`). The rest of the codebase imports our adapter, never the SDK directly.

If/when the SDK breaks, one file changes.

The SDK is pinned to an exact version in `package.json` (no caret). SDK upgrades require an ADR noting the breaking changes addressed.

#### D-8. Config precedence

Resolution order:

```
CLI flags  >  environment variables  >  project config file  >  built-in defaults
```

Documented in one place. Enforced everywhere. No exceptions without an ADR.

---

### Tier D: Project Structure & Conventions

#### D-9. Monorepo (pnpm + turborepo) from day 1

Initial layout:

```
foreman/
├── packages/
│   └── solo/              # @foreman-lab/solo — Phase 1 package
│       └── package.json
├── pnpm-workspace.yaml
├── turbo.json
└── package.json
```

`@foreman-lab/core` (pure domain logic) is extracted when a second consumer appears (Phase 2a daemon). Not before.

#### D-10. vitest

ESM-native, fast, TypeScript-first. Proven across v1's 441 tests.

Alternatives rejected: Jest (ESM issues), Mocha (manual setup), node:test (less mature ecosystem).

#### D-11. eslint + prettier

Standard choice. Configured once in the monorepo root, inherited by all packages.

#### D-12. GitHub Actions CI

Workflow files live in `.github/workflows/`. Standard for OSS projects.

Minimum CI gates: `check` (tsc), `test` (vitest), `lint` (eslint), `format:check` (prettier).

#### D-13. Markdown docs in-repo

All documentation is `.md` files in `docs/`. Renderer (Starlight, Docusaurus, Nextra, etc.) is chosen later when a docs site is needed.

Writing docs as Markdown today ensures forward-compatibility with any renderer.

#### D-14. JSON for machine-to-machine; YAML allowed for human-edited config

- **JSON** for journal entries, state snapshots, MCP messages, API responses — anything a program writes and reads.
- **YAML** allowed for human-edited files (playbook templates, project config) where comments and readability matter.

The "Norway problem" and similar YAML footguns are mitigated by safe parsers (D-15) and Zod validation (D-3).

#### D-15. One YAML library: `yaml@2.x`

Drop `js-yaml` from dependencies.

`yaml@2.x` provides: TypeScript types, strict parsing, round-trip preservation, YAML 1.2 compliance.

---

### Tier C (continued): Flow Engine & Agent Contract

Architecture contracts at the same level as D-6–D-8. Numbered sequentially after Tier D to preserve original D-1..D-15 references.

#### D-16. LangGraph as core flow engine

`@langchain/langgraph` owns the state machine, checkpointing, interrupts, and time travel. Foreman wraps it with domain types (briefs, playbooks, signals) and transport adapters (MCP, CLI).

- Phase 1 Solo uses `SqliteSaver` from `@langchain/langgraph-checkpoint-sqlite`.
- Phase 3 Team swaps in `PostgresSaver` from `@langchain/langgraph-checkpoint-postgres`. One constructor change, no flow-logic diff. Raw Postgres client choice (pg / Drizzle / Prisma) is moot — the checkpoint library wraps it.
- All flow code (nodes, edges, state channels, reducers) lives in the `graph/` directory of the package that owns flow logic. Phase 1: `packages/solo/src/graph/`. Phase 2a+: `packages/core/src/graph/` after D-9 extraction.
- LangGraph SDK imports are confined to that directory. The rest of the codebase sees only foreman's domain types.

Rationale: re-implementing a state machine, journal, checkpoint/restore, and time travel is exactly the "reinvented wheel" the project explicitly rejects. LangGraph provides all four, is MIT-licensed, and is maintained by the primary ecosystem foreman targets.

Agents never see LangGraph types. They interact via the signal protocol (D-17). If LangGraph bumps a major version, one directory changes.

#### D-17. Handler/Transport Separation

Three layers, enforced by module boundaries:

```
┌─ Transport ─────────────────────────────────────────┐
│  packages/solo/src/mcp/     packages/solo/src/cli/  │  ← wire-level adapters
│  Zod-validate input →       translate to signal     │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─ Handler ───────────────────────────────────────────┐
│  packages/solo/src/handlers/ (Phase 1;              │  ← domain logic
│   extracted to packages/core/src/handlers/ at 2a)   │
│  signal → graph.invoke(Command)                     │
└──────────────────────┬──────────────────────────────┘
                       ▼
┌─ Flow ──────────────────────────────────────────────┐
│  packages/solo/src/graph/ (Phase 1;                 │  ← LangGraph nodes/edges
│   extracted to packages/core/src/graph/ at 2a)      │
│  plan/work/evaluate + state channels                │
└─────────────────────────────────────────────────────┘
```

Rules:

- Transport knows nothing about graph. It validates and forwards.
- Handler knows nothing about MCP or CLI. It takes validated signals.
- Flow knows nothing about handlers. It takes `Command` objects.

Consequence: when MCP SDK changes, only transport changes. When LangGraph changes, only flow changes. When domain logic changes, only handlers change. Three axes of change, three directories.

**Enforcement:** import-boundary lint (e.g., `eslint-plugin-boundaries` or `dependency-cruiser`) configured in Phase 1. CI gate — unauthorized cross-layer imports fail the build. Without enforcement these rules are aspirational.

#### D-18. Playbook injection model

Playbooks are not read from the filesystem by foreman. The agent embeds the full playbook contract inline in the initialization signal.

```typescript
const Brief = z.object({
  nodeId: z.string(),
  kind: z.enum(['terminal', 'composite']),
  goal: z.string(),
  playbook: z.object({
    name: z.string(),
    phases: z.object({
      plan: z.string(),
      work: z.string(),
      evaluate: z.string(),
    }),
    scope: z
      .object({
        readOnly: z.boolean(),
        paths: z.array(z.string()).optional(),
      })
      .optional(),
  }),
  children: z.array(z.string()).optional(),
});
```

Rationale:

- **P-1 (pure infrastructure):** foreman reading playbook files would mean foreman interprets phase content. It must not.
- **P-4 (MCP-first):** the agent already has a file system tool. Asking foreman to read files on its behalf duplicates capability and creates a second trust boundary.
- **Agent-agnostic:** no Claude Code tool names, no Copilot-specific fields, no Codex references. The playbook schema is the phase-content contract, not the agent's wiring.

Consequence: foreman stores playbooks only inside briefs inside checkpoints. There is no `playbooks/` directory, no YAML loader, no playbook registry in the harness.

---

### Tier D (continued): Naming & Terminology

#### D-19. Naming conventions and domain terminology

Two layers — the code style (internal identifier shape) and the domain vocabulary (public-facing terms used in types, schemas, MCP tools, CLI output, and docs).

**Code-naming style:**

| Construct                               | Convention                                    | Example                                                  |
| --------------------------------------- | --------------------------------------------- | -------------------------------------------------------- |
| Variables, functions, methods           | camelCase                                     | `nodeId`, `applySignal`                                  |
| Types, classes, interfaces, Zod schemas | PascalCase                                    | `Brief`, `TokenRequest`, `BaseError`                     |
| Enum members, string literal unions     | camelCase                                     | `"terminal"`, `"composite"`, `"reversible"`              |
| Error codes                             | SCREAMING_SNAKE                               | `SCHEMA_VALIDATION_FAILED`, `OPERATOR_APPROVAL_REQUIRED` |
| File and directory names                | kebab-case if multi-word, lowercase otherwise | `token-request.ts`, `handlers/`                          |
| CLI command names                       | lowercase, kebab-case if multi-word           | `foreman status`, `foreman validate-playbook`            |
| CLI flags                               | kebab-case, double-dash                       | `--approve`, `--thread-id`                               |
| MCP tool names                          | double-underscore namespace                   | `foreman__step`, `foreman__status`                       |
| Environment variables                   | SCREAMING*SNAKE with `FOREMAN*` prefix        | `FOREMAN_CHECKPOINT_PATH`                                |

**Product-name scope rule:**

The product name (`foreman`) appears **only** in external-surface identifiers visible to users or agents: npm package name, CLI binary, MCP tool namespace prefix, environment-variable prefix, config-file names. Internal code identifiers — types, classes, interfaces, functions, variables, constants (both exported and unexported) — use neutral, descriptive names. The package import path provides sufficient namespace.

| Surface type                                  | Product name appears? | Examples                                                                                                                                               |
| --------------------------------------------- | --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| External (users/agents see these)             | ✅ Required           | `@foreman-lab/solo`, `foreman status`, `foreman__step`, `FOREMAN_CHECKPOINT_PATH`, `.foreman/`                                                         |
| Internal (code consumers see via import path) | ❌ Forbidden          | `BaseError` (not `ForemanError`), `ErrorCode` (not `ForemanErrorCode`), `FlowState` (not `ForemanState`), `WireResponse<T>` (not `ForemanResponse<T>`) |

**Rationale:** (1) product rename should require minimal internal churn — only external surfaces change; (2) product prefix on every internal type is redundant — the import path already carries the namespace; (3) neutral names compose better for consumers who alias at import site to avoid collisions (e.g., `import { ErrorCode as FError } from "@foreman-lab/solo"`).

**Domain terminology (locked core vocabulary):**

| Term           | Meaning                                                                    | Synonyms to AVOID                                                      |
| -------------- | -------------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| **Node**       | A unit of work under orchestration                                         | ~~task~~, ~~job~~, ~~stage~~                                           |
| **Brief**      | The immutable contract handed to a node at init                            | ~~goal~~, ~~spec~~, ~~ticket~~                                         |
| **Playbook**   | Phase-content contract embedded in brief                                   | ~~role~~, ~~persona~~, ~~agent-type~~, ~~skill~~                       |
| **Plan**       | Agent's proposal for the work                                              | ~~proposal~~, ~~strategy~~                                             |
| **Work**       | Agent's report of what was done                                            | ~~result~~, ~~output~~                                                 |
| **Evaluation** | Agent's assessment of work against brief                                   | ~~review~~, ~~judgment~~                                               |
| **Outcome**    | Derived truth-table result — `pass` / `retry` / `block` / `exhausted`      | ~~status~~, ~~verdict~~                                                |
| **Block**      | Operator-input-required pause with a category                              | ~~pause~~, ~~wait~~, ~~halt~~                                          |
| **Journal**    | Append-only event log                                                      | ~~history~~, ~~trace~~ (plain ~~log~~ is reserved for runtime logging) |
| **kind**       | `"terminal"` or `"composite"` — fixed at init                              | ~~type~~, ~~mode~~                                                     |
| **Checkpoint** | LangGraph-managed state snapshot                                           | ~~save~~, ~~snapshot~~ (unless quoting LangGraph's own API)            |
| **Operator**   | The human who resolves interrupts and approves irreversible actions        | ~~user~~ (ambiguous — agent is also a caller)                          |
| **Agent**      | The AI caller (Claude Code, Copilot CLI, etc.) using foreman's MCP surface | ~~client~~, ~~bot~~                                                    |
| **Harness**    | Foreman itself — the orchestrating process                                 | ~~framework~~, ~~runner~~                                              |

**Enforcement:** terminology appears in Zod schemas, type names, MCP tool names, CLI output strings, error messages, and prose documentation. Consistency is a reviewer's responsibility at PR time. A Phase 1 `docs/style.md` may codify examples, but the source of truth is this decision.

**Change process:** terminology changes follow the same drift-prevention rules as other decisions (ADR + approval + migration plan + doc update). A terminology rename cascades through types, schemas, tests, and docs — migration plans must account for that.

#### D-20. Outbound port discipline (narrow scope)

Foreman defines explicit TypeScript interfaces for outbound dependencies **only where real variation exists or a proven alternate is coming in the next phase**. Ceremony ports (one-implementation interfaces with no alternate ahead) are rejected.

**At Phase 1 (post-2026-04-18 domain-blind pivot), this rule yields ZERO outbound ports.**

Earlier M4 planning (see `docs/phase-1-solo/m4-development-proposal.md`, locked 2026-04-18 and superseded-in-part the same day) introduced `PromptAssemblerPort` as the Phase 1 exemplar on the premise that prompts were rendered inside graph nodes. The pivot reality-checked that premise: foreman is pure infrastructure (P-1), and prompt rendering is a domain-content operation. External agents render prompts from the raw `brief.playbook.phases[phase]` string carried through the existing `brief` channel. No `PromptAssemblerPort`, no `PromptView`, no `view-mapping`, no `nextPrompt` surface survives the pivot. See `docs/adr/0007-domain-blind-core.md` for the full decision.

D-20 itself is unchanged — the principle still holds. The exemplar was retired because the capability it modeled turned out to belong on the agent side of the semantic boundary, not inside foreman.

**Non-port seams at Phase 1** (promote to port later when the promotion-trigger fires):

| Seam                | Phase 1 shape                                                                                                                                                                                                                                                             | Promote when                                                                                                                                                            |
| ------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Clock (P-5)         | Function parameter `(now: () => Date = () => new Date(), ...)`                                                                                                                                                                                                            | 3+ consumers OR an alternate implementation lands                                                                                                                       |
| Prompt rendering    | Handled by the external agent — foreman emits raw `brief.playbook.phases[phase]`; agent interpolates                                                                                                                                                                      | Never — rendering is a domain-content operation (P-1). A future change here would require re-chartering P-1.                                                            |
| Token vending (S-1) | Handler function stub; Phase 1 default is `process.env` pass-through (M3 stub still in place post-M4; Batch C1 dropped to Phase 2a)                                                                                                                                       | Phase 2a daemon adds Vault/HSM — at that point promote to `TokenVendingPort`                                                                                            |
| Checkpointer        | Used directly — LangGraph's `BaseCheckpointSaver` is already a port interface                                                                                                                                                                                             | Never (no wrapping — ceremony per D-16)                                                                                                                                 |
| LangGraph engine    | Used directly from `graph/` directory                                                                                                                                                                                                                                     | Never (D-16 locks the bet; wrapping contradicts "don't reinvent the wheel")                                                                                             |
| Sibling state read  | `deps.graph.getState(checkpointConfig(projectId, childNodeId))` called from `handlers/step.ts` work pre-check (Option B, v0.7). Reuses the already-injected `CompiledStateGraph` abstraction with a differently-keyed config — same existing abstraction, not a new port. | Phase 2a daemon adds remote-aware sibling reads (multi-store or network-hop) OR a third composite-consolidation consumer emerges — promote to `SiblingStateReaderPort`. |

**Existing abstraction vs new port (v0.7 hexagonal clarification):** `graph.getState(checkpointConfig(...))` called with a differently-keyed thread_id is the **same existing abstraction** — no new port required. A multi-store read (e.g., graph state + external system aggregation) would be a new port. The test is: does the new call reuse the already-injected interface with different arguments (existing), or does it introduce a new interface shape (port)?

**Location:** post-pivot there is no `packages/solo/src/ports/` or `packages/solo/src/adapters/` directory. If Phase 2a promotes `TokenVendingPort` (or any future port), it lives at `packages/core/src/ports/` after the D-9 extraction with default adapters at `packages/core/src/adapters/`. Composition root in `packages/<tier>/src/app.ts` wires them.

**Rule:** Adding a new port requires either (a) real variation across tiers/tests/implementations at this phase, OR (b) an alternate implementation landing in the current or next phase. ADR required to introduce new ports.

---

### Tier C (continued): Engine-neutral invariants (added by ADR 0002)

Engine-agnostic rules; framework-specific API names live in their ADRs (see `docs/adr/0002-langgraph-1x-adoption-and-deviations.md`).

### D-21. Harness-identity + protocol-integrity invariant

**Decision:** Foreman is a harness, not an agent runtime. All semantic transitions MUST occur only through validated signals (Zod-parsed at transport edge), graph state channels, and journaled events. Engine convenience features that retry, cache, stream, invoke tools, or run agents MUST NOT bypass the signal / channel / journal protocol.

**Grounding:** P-1 (pure infrastructure), P-2 (three-actor model), P-5 (reproducibility / determinism), D-16 (flow engine consumed directly, not wrapped), D-17 (handler/transport separation), architecture §3 + invariants I-7 (outcome truth table), I-8 (journal-derived retry).

**Consequences:**

- No prebuilt agent primitives adopted from any framework (e.g., LangGraph `createReactAgent`/`ToolNode`, or future equivalents from Temporal, Inngest, etc.).
- No node-level retry/cache policies from the flow engine — retry is journal-derived (I-8) and semantic (outcome rule 4). Cache breaks journal-derived determinism.
- No engine-managed streaming adopted as default telemetry — P-7 opt-in only.
- External agents own tool execution; foreman records `ToolCall` metadata only.

**Enforcement:** dependency-cruiser layer rules (landed in M2 boundary matrix), spec §5.9 preflight + I-7/I-8 semantic guardrails; ADR required to introduce any engine-managed side-channel.

**D-21 clarification (ADR 0003, 2026-04-18):** MCP tool names are **transport affordances**, not semantic boundaries. The semantic boundary per D-21 is the validated **signal envelope + graph channel + journaled event** — not how that signal is carried over MCP. The 2-tool MCP surface (`foreman__status` + `foreman__step`) is a transport grouping decision; the internal 8-variant signal vocabulary (initialize/plan/work/eval/resume/retry/token_request/status) carries the semantic meaning. See ADR 0003 for the full argument and the wrapper-owns-auth doctrine that flows from this distinction.

**D-21 read-vs-transition clarification (ADR 0004 Option B, 2026-04-18):** D-21 governs **transitions** — writes to channels or appends to the journal. Handler-layer **read-only cross-thread access** for sequence-validity pre-checks (e.g., `handlers/step.ts` calling `graph.getState(checkpointConfig(projectId, childNodeId))` to enforce `COMPOSITE_CHILD_NOT_READY` before invoking the graph) does NOT cross the protocol boundary — no channel is written, no journal event is appended, no state is mutated. The rejection is returned as a validated `WireResponse` envelope. Handlers MUST still surface rejections as validated responses (never as silent drops), and the actual transition (if the pre-check passes) still flows through a validated signal via `graph.invoke`. Reads to **inform** transition decisions are allowed; reads that **bypass** the signal protocol to effect transitions are still forbidden.

### D-17 clarification (engine containment)

`types/` may import `zod` only. Engine-specific schema helpers (e.g., LangGraph `withLangGraph` / `schemaMetaRegistry`) are **`graph/`-only** — they MUST NOT enter `types/`. This preserves Zod-first (D-3) and keeps Phase 2a core-package extraction a mechanical move.

### D-18 clarification (harness runtime only)

D-18's "foreman never reads playbook files at runtime" constrains the **harness**. A Phase 2a hosted service MAY maintain a playbook catalog at its service edge (roadmap / Gate 6); the harness still never reads it directly — catalog entries are injected into `brief.playbook` of inbound `initialize` signals the same way as today.

---

## Deferred Decisions (not locked now)

The following are explicitly deferred to the phase that introduces them. They are not foundational and benefit from being made with context.

| Decision                                                                                               | Deferred until                             | Why defer                                                                                                                            |
| ------------------------------------------------------------------------------------------------------ | ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------ |
| Test boundaries (unit/integration/e2e) + mocking policy + coverage floor                               | Phase 1 spec (`docs/phase-1-solo/spec.md`) | Operational discipline, not constitutional; specifics benefit from real codebase context                                             |
| General dependency pinning policy (exact vs caret for non-SDK deps, lockfile commit rules)             | Phase 1 spec                               | SDK pinning handled in D-7; general discipline benefits from real dependency list                                                    |
| Versioning / semver policy (what triggers major bump; pre-1.0 behavior; monorepo package coordination) | Before Phase 1 npm release                 | Must be written before first `npm publish`; cannot be retroactive                                                                    |
| Property-based + mutation testing tools (`fast-check`, `stryker-js`)                                   | Phase 2+                                   | Solo-dev Phase 1 doesn't justify the overhead; outcome truth table is 16 enumerable combinations; revisit when contributor count > 1 |
| Logging library (pino vs console)                                                                      | Phase 2a (daemon)                          | Phase 1 MCP uses stdio — structured logs don't travel separately yet                                                                 |
| HTTP framework (Fastify / Hono / Express)                                                              | Phase 2a                                   | No HTTP server in Phase 1                                                                                                            |
| UI framework (React / Vue / Svelte)                                                                    | Phase 2b                                   | No UI until dashboard                                                                                                                |
| Graph visualization library                                                                            | Phase 2b                                   | No UI until dashboard                                                                                                                |
| Auth library (Passport / Lucia / @fastify/jwt)                                                         | Phase 3                                    | No multi-user until Team tier                                                                                                        |
| SSO provider (SAML / OIDC)                                                                             | Phase 4                                    | Enterprise-only                                                                                                                      |
| Commit convention (Conventional Commits?)                                                              | When 3+ contributors                       | Pre-mature for 1-2 person Phase 1                                                                                                    |
| Docs renderer                                                                                          | When docs site launches                    | Markdown content is forward-compatible                                                                                               |

---

## Drift Prevention

These decisions are not revisited casually. Changes require:

1. An ADR (Architecture Decision Record) in `docs/adr/NNNN-title.md`
2. Explicit user approval
3. Migration plan for existing code
4. Update to this document

Promoting a **deferred** decision to **locked** follows the same process: an ADR, explicit approval, a migration note if the promotion affects existing code, and an update to this document removing the row from the deferred table and adding the locked decision with its next available number.

The principles (P-1 through P-10) have stronger protection than decisions. Amending a principle requires:

1. All four steps above (ADR, approval, migration plan, doc update)
2. A minimum 2-week deliberation window between ADR publication and merge
3. Explicit acknowledgment in the ADR that this constitutes **re-chartering** of the project
4. An updated `MISSION.md` or equivalent artifact so the change is visible at the top of the repo, not buried in an ADR

Amending a principle is a structural event. If P-1, P-4, or P-10 were to change, the project is effectively a different product.

---

## Cross-references

Paths marked `(created in Phase 1)` do not exist yet; they are forward references to the monorepo layout that Phase 1 Solo will establish.

- **P-3 Security by design** → `docs/phase-1-solo/security.md` (created in Phase 1)
- **D-7 MCP adapter** → `packages/solo/src/mcp/adapter.ts` (created in Phase 1)
- **D-16 LangGraph flow engine** → `packages/solo/src/graph/` (created in Phase 1; extracted to `packages/core/src/graph/` at Phase 2a per D-9)
- **D-17 Handler/Transport Separation** → `packages/solo/src/handlers/`, `packages/solo/src/{mcp,cli}/` (created in Phase 1; handlers extracted to `packages/core/` at Phase 2a)
- **D-18 Skill injection model** → `packages/solo/src/types/brief.ts` (created in Phase 1)
- **D-20 Outbound port discipline** → no runtime `ports/` or `adapters/` directory at Phase 1 post-2026-04-18 domain-blind pivot; will materialize at Phase 2a if/when `TokenVendingPort` promotes (see `docs/adr/0007-domain-blind-core.md`)
- **Phase progression** → `docs/roadmap.md` (exists — locked baseline)
- **Overall architecture** → `docs/architecture.md` (exists — revised post-hexagonal review)
