---
status: SUPERSEDED IN PART 2026-04-18 — locked vN 2026-04-18 and reverted mid-execution the same day per domain-blind pivot; see ADR 0007 + §Supersession notice below. Body retained as historical record of the pre-pivot plan.
milestone: M4 — Skills (first formal port + skill YAML + 1 example skill + skill-driven dogfooding) [pre-pivot intent]
gate: partial G6 (S-1 materialized via token vending) + partial G9 (agent-setup + 1 example skill) [pre-pivot intent — actual M4 landed scope is narrower; see ADR 0007 §Consequences]
date: 2026-04-18
methodology: Foreman PGE (Plan → Work → Evaluate) + GAN review (Codex + Copilot) through hexagonal-architecture lens
depends_on: m3-alpha tagged on 9a859ab (local; not pushed); ADR 0001 (hexagonal) + ADR 0003 (2-tool MCP) + ADR 0004 (composite) locked; D-17/D-18/D-20/D-21 locked
supersedes: m4-development-proposal-skeleton.md v0.1..v0.4 (skeleton retired; lineage preserved in §13)
superseded_by: docs/adr/0007-domain-blind-core.md (post-2026-04-18 pivot; PromptAssemblerPort + PromptView + view-mapping + nextPrompt reverted); docs/phase-1-solo/m5-development-proposal.md (absorbs un-shipped M4 remainder under the domain-blind architecture)
grounding: spec §3.5 M4 plan; foundations D-17/D-18/D-20/D-21; architecture.md §3.3 + §3 Scenario 3; roadmap Gate 6 hedge
---

> **⚠ SUPERSEDED IN PART — Post-2026-04-18 domain-blind pivot**
>
> This proposal was locked on 2026-04-18 and reverted the same day after an operator reality-check: *"foreman is an infrastructure, and it never call api directly"* + *"foreman is more state machine, because pure skill always drift, however, domain knowledge are always independent."*
>
> The following sections are **no longer load-bearing** — their code was reverted before any agent consumed it:
>
> - §3.2 "Runtime/authoring split" — runtime rendering path retired; agents render prompts externally
> - §3.3 File enumeration (`ports/prompt-assembler.ts`, `adapters/default-prompt-assembler.ts`, `types/prompt-view.ts`, `graph/view-mapping.ts`, `tests/contract/prompt-assembler-port.test.ts`, `tests/unit/prompt-assembler-adapter.test.ts`) — all deleted
> - §3.4 Error-code deltas — `PROMPT_ASSEMBLY_FAILED` removed; `TOKEN_VENDING_FAILED` + `SKILL_VALIDATION_FAILED` reserved forward-compat only
> - §3.5 Dep-cruiser rule-name list — 6 M4 hexagonal rules retired
> - §3.6 Layer table — `ports/` + `adapters/` rows removed (directories do not exist)
> - §3.7 `PromptAssemblerPort` contract — retired in full
> - §4 Dogfooding integration skill-driven flow — steps 2-5 referencing node-side assembly retired; M5 replaces with agent-rendered flow
> - §5 Risk register R-M4-1/2/4/7 — all tied to assembler behavior; no longer apply
> - §7 Batch layout — Batches A1/A2/B/C2 **reverted** (code shipped then un-shipped); Batches D1/D2/E/F/G/H/I **deferred to M5** under the domain-blind architecture
> - §8 Invariants introduced at M4 — I-11/I-14/I-15/I-16/I-17 retired; I-10/I-12/I-13 retained with narrower framing per ADR 0007
>
> **What M4 actually shipped (landed scope):**
>
> - Batch A2 Scout fix — `--foreman-dir` propagation bug fix across `init` / `status` / `step` (still green)
> - Batch A3 spike decision — keep `replaceReducer` + `Annotation inputSignal` pattern (documented won't-fix for M4; Phase 2a framework migration)
> - Rename cascade — `BriefKind` → `NodeKind` → `TaskKind` (operator feedback)
> - ADR 0005 "Skill file format" — Accepted (amended 2026-04-19 with post-pivot note; format + distribution + `validate-skill` CLI contract unchanged)
> - ADR 0007 "Domain-blind core" — Accepted (this is the pivot decision record)
> - Revert commit — 14-file pivot-revert cleanup (7 deletions + 7 modifications) with 281/281 tests green + 0 dep-cruiser violations
>
> **Read this proposal for history only.** For the active plan see:
>
> - `docs/adr/0007-domain-blind-core.md` — decision surface
> - `docs/phase-1-solo/m4-evaluation.md` — M4 evaluation against landed-vs-proposed scope (to be authored)
> - `docs/phase-1-solo/m5-development-proposal.md` — the forward plan under the domain-blind architecture
> - `docs/foundations.md` D-20, `docs/architecture.md` §2.2, `docs/phase-1-solo/architecture.md` §2/§4/§5 — updated doc set

# M4 Development Proposal — Skills + First Port (vN — locked 2026-04-18, superseded-in-part same day)

**Lineage:** Skeleton v0.1 → v0.4 ran 3 GAN rounds (Codex + Copilot + architect, 3 reviewers/round = 9 agent-rounds) before promotion. Cross-references labeled "Round 2/3 X" refer to the **skeleton-tier** rounds; proposal-tier rounds start at "Proposal Round 1" (applied → v0.2).

## 1. Goal

M4 materializes **D-18 (skill injection)** and **D-20 (narrow port discipline)** by introducing the first formal outbound port (`PromptAssemblerPort`) and shipping one production-ready example skill (`tdd-feature`). Dogfooding switches from **raw-signal mode** (M3 close) to **skill-driven mode**: operator authors a brief with a full inline `Skill` object, foreman assembles prompts inside graph nodes via the port, agent consumes assembled prompts in the step envelope.

Secondary: unstub the M3 `token_request` handler into a real pass-through env-vending hook (S-1 "no ambient credentials" materialization per D-5); ship CLI `inspect` + `validate-skill` subcommands identified as M3 dogfooding pain points.

**Non-goal (locked):** see §10.

## 2. Gates

| Gate  | Closes fully? | Partial?  | Notes |
|-------|---------------|-----------|-------|
| G2    | Maintained    | —         | MCP surface stable at 2 tools (asserted by new `contract/mcp-tool-count.test.ts`) |
| G4    | No            | Possibly  | Skill-driven scenario variants may land here; may defer to M5 |
| G6    | No            | Yes       | S-1 "no ambient credentials" materialized (token vending real, not stub); S-2..S-5 still M5 |
| G7    | No            | Partial   | Net +3 error codes at M4 (3 added; none removed); full catalog audit stays M6 |
| G9    | No            | Yes       | `agent-setup.md` + `tdd-feature` ship; 2-skill + scenario-suite target at M5/M6 |

Test-count delta: M3 ships ~270 passing; M4 adds ~40 (unit assembler + contract port + integration graph-assembler + interrupt-resume regression + CLI inspect/validate-skill + backward-compat + dep-cruiser gate).

## 3. Scope

### 3.1 Dependency changes

- **Default:** no new npm dependency. YAML parsing uses an already-present dep; verify during Batch F.
- **Contingency:** if YAML parser absent, exact-pin `yaml@2.x`.

### 3.2 Runtime/authoring split

- **Runtime:** `brief.skill` is always a full inline `Skill` object matching M3-shipped `SkillSchema` (`packages/solo/src/types/skill.ts:20-26`). Foreman never reads skill files at runtime per D-18. Runtime adapters accept **only** parsed inline `Skill` from `brief.skill`.
- **Authoring/distribution:** skill authors publish YAML files (canonical) or TS modules (advanced). Wrappers/CLI (e.g., `foreman validate-skill`) parse YAML/TS into a `Skill` object; they never hand raw YAML to runtime.
- **`foreman validate-skill <path>`** is a **transport-layer dev tool** that does **not** depend on `PromptAssemblerPort`. Resolves `<path>` relative to `cwd`; does NOT declare `--foreman-dir` (avoid "accept-but-ignore" antipattern).

### 3.3 File enumeration

**New files (M4):**

- `packages/solo/src/types/prompt-view.ts` — **NEW**: single flat `PromptView` interface + `PromptViewSchema` (Zod) + `AssembledPrompt` branded type. Shape: `{ phase, methodology, vars: Record<string,string> }`. Co-located with other domain schemas per M3 precedent (PromptView Round 2 GAN unanimous decision (d): flat vars bag; see §3.7 for rationale).
- `packages/solo/src/ports/prompt-assembler.ts` — `PromptAssemblerPort` interface only; imports `PromptView` + `AssembledPrompt` TS types from `types/prompt-view.ts`. No Zod schemas live here.
- `packages/solo/src/ports/index.ts` — barrel
- `packages/solo/src/adapters/skill-methodology-assembler.ts` — default `PromptAssemblerPort` implementation; pure + idempotent + kind-blind (I-10, I-14)
- `packages/solo/tests/fixtures/skills/index.ts` — **barrel-style** (R3 Copilot MEDIUM-2) shared `Skill` fixture helpers (M5 can add `skills/refactor-extract.ts` without import-path churn)
- `packages/solo/src/adapters/passthrough-env-vendor.ts` — token-vending default (plain function; not port-shaped per I-12)
- `packages/solo/src/adapters/index.ts` — barrel
- `packages/solo/src/graph/view-mapping.ts` — `Brief`/`Plan`/`WorkReport` → `PromptView` projection helpers (one helper per phase: `planView(state)`, `workView(state)`, `evalView(state)`; each returns a `PromptView` with `phase` + `methodology` + `vars`). Domain→view boundary lives in graph/use-case layer per I-15/I-16.
- `packages/solo/src/cli/inspect.ts` — `foreman inspect [--node <id>] [--pretty]` subcommand
- `packages/solo/src/cli/validate-skill.ts` — `foreman validate-skill <path>` subcommand
- `docs/adr/0005-skill-file-format.md` — **EXISTING (Draft v1 authored 2026-04-18 at v0.5; pending operator sign-off)**
- `docs/adr/0006-prompt-assembler-port.md` — **Batch H deliverable**; authored against locked flat vars bag shape (§3.7; Q-2 (d) via PromptView R2 GAN); covers port contract, call-site, I-14 purity prohibition list, I-10 kind-blind, I-15/I-16/I-17 projection rules
- `docs/agent-setup.md` — MCP config snippets for Claude Code, Copilot CLI, Cursor
- `examples/tdd-feature/skill.yaml` — first example skill (creates `examples/` directory)
- `packages/solo/tests/unit/prompt-assembler-adapter.test.ts` — `{{var}}` token extraction; replacement correctness; missing-var throws `PROMPT_ASSEMBLY_FAILED`; kind-blind fixture matrix (I-10) across `{terminal, composite} × {plan, work, evaluate}`; purity/idempotency (I-14) N=100; vars-strings-only (I-17)
- `packages/solo/tests/contract/prompt-assembler-port.test.ts` — single `PromptViewSchema` shape assertion + `AssembledPrompt` branded-type assertion (1 fixture, not 3 per v0.4 flat shape)
- `packages/solo/tests/contract/mcp-tool-count.test.ts` — transport regression (G2 assertion)
- `packages/solo/tests/contract/composition-root.test.ts` — `app.ts` binding site + I-13 enforcement
- `packages/solo/tests/integration/graph-assembler.test.ts` — Option A: assembler invocation inside plan/work/evaluate nodes
- `packages/solo/tests/integration/interrupt-resume-assembler.test.ts` — purity regression across N resumes
- `packages/solo/tests/unit/handler-backward-compat.test.ts` — all 10 existing `createHandlers` call sites compile

**Updated files (M4):**

- `packages/solo/src/types/error-codes.ts` — +3 codes (`TOKEN_VENDING_FAILED`, `PROMPT_ASSEMBLY_FAILED`, `SKILL_VALIDATION_FAILED`); rewrite leading JSDoc per M4 narrative; update count 8→11
- `packages/solo/src/handlers/step.ts` — **extend existing `StepSuccessSchema`** (already exported at `handlers/step.ts:41-52`, re-exported from `handlers/index.ts:28` — Proposal R2 Copilot B1 correction of v0.2 factual error) with optional `nextPrompt?: { phase, text }` field (Q-3a α lock); thread `nextPrompt` from node output through `okFor()` helper (lines 351-356); `token_request` case replaces `SCHEMA_VALIDATION_FAILED` stub with `vendingHook` dispatch; widen `err()` helper union (lines 358-371)
- `packages/solo/src/types/signal.ts` — add `NextPromptSchema` (Zod; `{ phase: 'plan'|'work'|'evaluate', text: AssembledPrompt }`); referenced by the extended `StepSuccessSchema`
- `packages/solo/src/graph/channels.ts` — add `nextPrompt` channel (`Annotation.Root` entry; `replaceReducer`) so plan/work/evaluate node returns can emit assembled prompt for envelope projection (Proposal R2 Copilot H2). Semantics are **per-invoke projection, not durable history**: because `Annotation.Root` channels are checkpointed, the handler must clear `nextPrompt` to `null` at the start of every `graph.invoke`; a node that emits a prompt replaces that null in the same invoke. This prevents stale prompt replay and state bloat while still using LangGraph's channel mechanism.
- `packages/solo/src/handlers/index.ts` — `createHandlers` grows optional `vendingHook?` dep; `promptAssembler` stays OUT of handlers
- `packages/solo/src/graph/build.ts` — **`BuildGraphDeps` extended** with `promptAssembler?: PromptAssemblerPort`; **threaded through** `createPlanNode({ now, promptAssembler })`, `createWorkNode({ now, promptAssembler })`, `createEvaluateNode({ now, maxRetries, promptAssembler })` factory signatures (Batch A delta)
- `packages/solo/src/graph/nodes/{plan,work,evaluate}.ts` — **Batch A scaffold**: factory signatures gain `promptAssembler?` (accepted but unused; absent → M3 behavior). **Batch D1/D2 invocation**: node body builds a flat `PromptView` via `view-mapping` and invokes `promptAssembler.assemble(view)` BEFORE `interrupt()` so prompt is available at pause (I-14 purity makes resume re-run safe). Split deliberate (Codex R4 Check 1): A cannot invoke because adapter + view-mapping don't yet exist.
- `packages/solo/src/app.ts` — composition root wires `SkillMethodologyAssembler` into `buildGraph`, `passthroughEnvVendor` into `createHandlers`; I-13 sole instantiator of adapters
- `packages/solo/src/cli/index.ts` — register `inspect` + `validate-skill`; fix pre-existing `--foreman-dir` inheritance bug (Scout; Batch A prereq)
- `packages/solo/src/cli/{status,step}.ts` — read `--foreman-dir` via `program.opts()` not per-subcommand opts (B1 bug fix)
- `packages/solo/src/cli/init.ts` — **separate fix (R2 Codex H8)**: `runInit` currently hardcodes `join(opts.cwd, '.foreman')` (`cli/init.ts:19`), ignoring the `--foreman-dir` flag entirely. Semantics lock: `foreman init --foreman-dir <path>` creates the override directory directly (not `cwd/.foreman`); if flag absent, default stays `cwd/.foreman`. This is structurally different from `status`/`step` (which use `resolveConfig`), so init gets its own regression test
- `.dependency-cruiser.cjs` — +6 named rules (`ports-only-types-and-zod`, `adapters-no-transport-or-handlers`, `handlers-no-adapters`, `handlers-no-ports`, `graph-no-adapters`, `composition-root-isolation` — §3.5) + time-boxed `no-orphans` exclusion for `ports/**`
- `docs/phase-1-solo/spec.md` — §8.4 `token_request` row updated (handler delegates to `vendingHook`)
- `docs/phase-1-solo/architecture.md` — §3 concretized with dep-cruiser rule-name list + Option A call-site + "assembler is query-only / CQRS read-side" paragraph
- `docs/dogfooding-protocol.md` — mark `inspect` + `validate-skill` as shipped; shrink §12 deferrals; add skill-authoring friction template
- `packages/solo/vitest.config.ts` — decide coverage thresholds block (Batch A Scout): either add thresholds or strike G5 ≥80% claim
- `scripts/demo.ts` — verify `brief.skill` inline hydration still valid; no change expected

### 3.4 Error-code deltas (parallel to M3 §3.10 style per R2 Copilot M5)

**`TOKEN_VENDING_FAILED`** — **DEFERRED TO PHASE 2a (Batch C1 dropped 2026-04-18)**
- **M4 status:** code registered in catalog for forward-compat; NOT thrown at runtime. M3 stub at `handlers/step.ts:313-321` stays in place; `token_request` signals return `SCHEMA_VALIDATION_FAILED` at Phase 1.
- **Drop rationale (operator + GAN consensus):** foreman is infrastructure that does not call external APIs; the agent holds its own credentials. Phase 1 `passthroughEnvVendor` provides zero security value vs the agent reading env directly — no scoping, no revocation, no audit beyond the existing journal. D-20 narrow-port discipline says promote seams only when real variation arrives, which happens at Phase 2a with Vault/HSM.
- **Phase 2a revival:** when a real vending backend lands, `TOKEN_VENDING_FAILED` is thrown by the (then-to-be-created) `handlers/step.ts` `token_request` handler after the vending adapter rejects. `details` schema at that point: `{ scope: string, reason: string }` (no secrets; D-5 SecurityError class).

**`PROMPT_ASSEMBLY_FAILED`**
- **Location:** thrown by `SkillMethodologyAssembler` adapter during template interpolation
- **Trigger:** `view.methodology` string references a `{{var}}` token not present as a key in `view.vars` (flat vars bag; no per-phase DTO fields). PromptView Round 2 GAN (d) lock.
- **`details` schema:** `{ phase: 'plan'|'work'|'evaluate', missingVars: string[] }`
- **Remediation hint:** "Prompt assembly failed for phase `<phase>`: methodology references `{{<var>}}` not provided by view-mapping. Either update skill methodology to use a declared var or extend `graph/view-mapping.ts` to include the var for this phase."

**`SKILL_VALIDATION_FAILED`**
- **Location:** thrown by `cli/validate-skill.ts` (CLI-only; never from runtime handlers)
- **Trigger:** Zod parse of YAML/TS input fails against `SkillSchema`; shallow template-var extraction warnings for undeclared `{{var}}` are reported by `validate-skill` but do not fail the command (Q-1b)
- **`details` schema:** `{ path: string, zodIssues: z.ZodIssue[] }`
- **Remediation hint:** "Skill file at `<path>` failed validation: `<first-issue>`. Run `foreman validate-skill <path>` for full Zod issue list."

**Explicit non-addition:** no `SKILL_NOT_FOUND` — per D-18, runtime cannot have this condition (foreman never reads skill files at runtime).

**Existing error-shape grounding (Round 3):** `packages/solo/src/errors/` exists. `BaseError.details` is currently `unknown` and `toJSON()` safe-serializes arbitrary details, while `ValidationError` / `SecurityError` / `StorageError` are thin subclasses. The schemas above are therefore M4-specific error-code contracts layered on top of the existing permissive details field; Batch C/F tests must assert these shapes because the base error class does not.

### 3.5 Dep-cruiser rule-name list

Add to `.dependency-cruiser.cjs` (all rules include `severity: 'error'` + `comment` fields matching existing cjs convention):

```js
// New rules at M4:
{
  name: 'ports-only-types-and-zod',
  severity: 'error',
  comment: 'Ports are pure contracts; may only depend on types/ and zod.',
  from: { path: '(^|/)packages/solo/src/ports/' },
  to: { pathNot: '^(packages/solo/src/types/|zod)' },
},
{
  name: 'adapters-no-transport-or-handlers',
  severity: 'error',
  comment: 'Adapters implement ports; must not reach into transport or handlers.',
  from: { path: '(^|/)packages/solo/src/adapters/', pathNot: 'app\\.ts$' },
  to: { path: '(^|/)packages/solo/src/(graph|handlers|mcp|cli)/' },
},
{
  name: 'handlers-no-adapters',
  severity: 'error',
  comment: 'Handlers never instantiate or import concrete adapters. Composition root (app.ts) is the only carve-out.',
  from: { path: '(^|/)packages/solo/src/handlers/', pathNot: 'app\\.ts$' },
  to: { path: '(^|/)packages/solo/src/adapters/' },
},
{
  name: 'handlers-no-ports',
  severity: 'error',
  comment: 'Handlers stay transport-adjacent; prompt assembly ports are graph-only and token vending remains a plain injected function in M4.',
  from: { path: '(^|/)packages/solo/src/handlers/' },
  to: { path: '(^|/)packages/solo/src/ports/' },
},
{
  name: 'graph-no-adapters',
  severity: 'error',
  comment: 'Graph nodes depend on ports only; complements graph-no-downstream.',
  from: { path: '(^|/)packages/solo/src/graph/' },
  to: { path: '(^|/)packages/solo/src/adapters/' },
},
{
  name: 'composition-root-isolation',
  severity: 'error',
  comment: 'I-13: app.ts is the SOLE site that simultaneously imports adapters/ and ports/. Any other file importing both is a composition-root violation.',
  from: { path: '(^|/)packages/solo/src/(?!app\\.ts)', pathNot: '(^|/)app\\.ts$' },
  // Enforcement: combine with a contract test — tests/contract/composition-root.test.ts
  // scans the file graph for any non-app.ts module importing both adapters/ and ports/.
  to: { path: '(^|/)packages/solo/src/adapters/' },
},
```

**I-13 enforcement note:** The `composition-root-isolation` rule above is a coarse ban paired with the `composition-root.test.ts` contract test (§3.3) that does the actual "single binder" scan. The rule blocks non-`app.ts` files from reaching into `adapters/`; the test verifies `app.ts` is the only file crossing the `ports/` ↔ `adapters/` boundary.

Rule name chosen: **`graph-no-adapters`** (Copilot H2: prior `graph-ports-allowed` was misleading — the rule forbids adapters; ports access is implicit). Round 3 adds **`handlers-no-ports`** to keep `PromptAssemblerPort` out of handlers and preserve D-17; handlers receive only graph outputs and the plain `vendingHook` function.

Plus time-boxed `no-orphans` exclusion for `ports/**` (lifted after Batch C2 wiring lands).

### 3.6 Layer table (hexagonal labels)

| Layer | Role | May import | May NOT import |
|---|---|---|---|
| `ports/` | port contracts | `types/`, `zod` | everything else |
| `adapters/` | infrastructure adapters | `ports/`, `types/`, `errors/`, framework SDKs | `graph/`, `handlers/`, `mcp/`, `cli/` |
| `graph/` | **application orchestration / LangGraph-bound adapter layer** (framework SDK lives here per ADR 0002) | `ports/`, `types/`, `errors/`, `@langchain/langgraph` | `adapters/`, `handlers/`, `mcp/`, `cli/` |
| `handlers/` | transport-adjacent request handlers | `types/`, `errors/`, `graph/` (compiled) | `ports/`, `adapters/`, `mcp/`, `cli/` |
| `mcp/`, `cli/` | transport delivery | `handlers/`, `types/`, `errors/` | `graph/`, `adapters/`, `ports/` directly |
| `app.ts` | **composition root** — I-13 sole instantiator of `adapters/`; dep-cruiser carve-out | all | — |

### 3.7 `PromptAssemblerPort` contract (flat `PromptView` + vars bag; sync)

**PromptView Round 2 GAN unanimous decision (d)** — Codex + Copilot both flipped from prior positions (Codex from (b) Base+extend; Copilot from (a) distinct) after re-examining the repo. Rationale for flat vars bag over per-phase DTOs:

- **M3 DU precedent doesn't structurally apply.** `StepSignalSchema`/`PlanSchema`/`WireResponseSchema` use `z.discriminatedUnion` because variants carry genuinely different payloads (`PlanSchema` vs `WorkReportSchema` vs `EvaluationSchema`). Plan/work/evaluate prompt views are "same shape with different keys populated in a bag" — not structural divergence.
- **Claude-Code parallel.** Prompt construction is template + context, not per-phase typed schemas. The skill methodology IS the template; vars are the context.
- **Narrow-port discipline (D-20).** Port footprint: ~20 lines for (d) vs ~70 lines for per-phase DUs. Capability expressed: "render a string template with a vars bag" — not per-phase state-semantic reasoning.
- **Runtime already enforces vars.** `PROMPT_ASSEMBLY_FAILED` on missing `{{var}}` (§3.4) is the authoritative check; static per-phase typing would be a weaker second check that doesn't cover skill template drift anyway.
- **Phase 2a migration trivial.** Tier-specific vars (e.g., `reviewer_context`) added to the bag by tier view-mapping; port contract unchanged.

**I-14 purity locks the port as sync** — string interpolation forbids I/O, so `Promise<>` wrapping serves no purpose.

```typescript
// types/prompt-view.ts
//
// Flat template + vars bag. "vars" = preconditions (not "summaries"): keys
// carry forward data from prior phases that condition the current prompt.
// View-mapping projection (I-16) pulls view.methodology from
// skill.methodology[view.phase]; vars are populated per phase by helpers.
export const PromptViewSchema = z.object({
  phase: z.enum(['plan', 'work', 'evaluate']),
  methodology: z.string().min(1),
  vars: z.record(z.string(), z.string()).readonly(),
}).strict();

export type PromptView = z.infer<typeof PromptViewSchema>;

export type AssembledPrompt = string & { readonly __brand: 'AssembledPrompt' };
export const makeAssembledPrompt = (text: string): AssembledPrompt =>
  text as AssembledPrompt;  // single boundary per skeleton R3-2
```

```typescript
// ports/prompt-assembler.ts
import type { PromptView, AssembledPrompt } from '../types/prompt-view.js';

export interface PromptAssemblerPort {
  assemble(view: PromptView): AssembledPrompt;  // sync per I-14
}
```

**View-mapping per phase** (`graph/view-mapping.ts`):

| Phase    | `view.methodology`              | `view.vars` keys                            |
|----------|---------------------------------|---------------------------------------------|
| plan     | `skill.methodology.plan`        | `goal`, `kind`, `scope_readonly`, `scope_paths`     |
| work     | `skill.methodology.work`        | `goal`, `kind`, `plan`, `scope_readonly`, `scope_paths` |
| evaluate | `skill.methodology.evaluate`    | `goal`, `kind`, `plan`, `work`                      |

**`kind` exposure** (Codex R4 Check 2): `kind` is available as a template variable `{{kind}}` so skill authors can conditionally vary prose by `terminal` vs `composite` — but the adapter remains **kind-blind** per I-10 (it does not branch on `view.vars.kind`, it only substitutes). Authors using `{{kind}}` write templates that read cleanly in both kind contexts.

Skill methodology templates reference `{{goal}}`, `{{plan}}`, `{{work}}`, `{{scope_readonly}}`, `{{scope_paths}}`. Missing referenced var → `PROMPT_ASSEMBLY_FAILED` at runtime.

**Adapter implementation** (`SkillMethodologyAssembler`) is ~10 lines:

```typescript
assemble(view: PromptView): AssembledPrompt {
  const referenced = extractVarTokens(view.methodology);  // regex: /\{\{(\w+)\}\}/g
  const missing = referenced.filter((v) => !(v in view.vars));
  if (missing.length) throw new BaseError('PROMPT_ASSEMBLY_FAILED', { phase: view.phase, missingVars: missing });
  const rendered = Object.entries(view.vars).reduce(
    (tmpl, [k, v]) => tmpl.replaceAll(`{{${k}}}`, v),
    view.methodology,
  );
  return makeAssembledPrompt(rendered);
}
```

**Phase 2a strangler migration (pseudocode):** tier vars are additive; port contract unchanged.

```typescript
// Phase 2a TeamAssembler just augments vars; no port shape change.
class TeamPromptAssembler implements PromptAssemblerPort {
  constructor(
    private readonly solo: PromptAssemblerPort,
    private readonly teamVars: (view: PromptView) => Record<string, string>,
  ) {}
  assemble(view: PromptView): AssembledPrompt {
    const augmented: PromptView = {
      ...view,
      vars: Object.freeze({ ...view.vars, ...this.teamVars(view) }),
    };
    return this.solo.assemble(augmented);
  }
}
```

Notably absent: `brief.kind`, `plan.kind`, full `Brief`, full `Plan`, full `WorkReport`, per-phase DTOs. Kind exposed only as a template variable (e.g., `{{kind}}`) inside the methodology string if the skill author needs it.

## 4. Dogfooding integration

M3 launched **raw-signal dogfooding** (operator hand-crafts JSON signals). M4 switches to **skill-driven dogfooding**:

1. Operator authors a brief with `brief.skill` as a full inline `Skill` object, e.g. `foreman validate-skill examples/tdd-feature/skill.yaml | jq '.data'` (validate-skill emits `WireResponse<Skill>` JSON to stdout on success — see Batch F acceptance). **`jq` prereq** (R2 Codex Check 9): Unix users install via `brew install jq` / `apt install jq`. Windows alt without `jq`: `foreman validate-skill ... | node -e "process.stdin.on('data', d => console.log(JSON.stringify(JSON.parse(d).data)))"` or PowerShell `(foreman validate-skill ... | ConvertFrom-Json).data | ConvertTo-Json -Depth 10`
2. `foreman__step type=initialize` seeds state; plan node assembles planning prompt via `PromptAssemblerPort`, delivers to agent in `StepSuccessSchema.nextPrompt`
3. Agent plans against the assembled prompt; `foreman__step type=plan` transitions to work phase
4. Work node assembles work prompt similarly; agent executes; `foreman__step type=work` transitions to evaluate
5. Evaluate node assembles eval prompt; agent evaluates; outcome cascades
6. Operator logs friction into `docs/dogfooding-log.md` — canonical M4 success signal

**Batch H acceptance includes one full skill-driven dogfooding iteration against `examples/tdd-feature`.**

## 5. Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| R-M4-1: LangGraph resume re-executes assembler twice | LOW | MEDIUM | Re-execution is acceptable after Round 3 because `PromptAssemblerPort` is sync, pure, and read-side only (I-14/D-21); `tests/integration/interrupt-resume-assembler.test.ts` asserts byte-identical output across N=100 resumes and no stale `nextPrompt` replay |
| R-M4-2: ~~Q-2 port shape flips late~~ CLOSED — Q-2 locked (d) flat vars bag in Proposal R2 PromptView GAN; residual risk = "vars-bag shape regret" (LOW/LOW) if Phase 2a reveals structured vars need | LOW | LOW | Runtime enforcement (`PROMPT_ASSEMBLY_FAILED`) + I-17 (strings-only) bound the shape; escalate to ADR amendment if Phase 2a hits structured-data need |
| R-M4-3: `--foreman-dir` inheritance bug cascades beyond Batch A scope (cascade points enumerated: `cli/status.ts`, `cli/step.ts`, `cli/init.ts` — 3 subcommands; plus `inspect` + `validate-skill` land post-fix) | MEDIUM | MEDIUM | Isolated Scout fix Batch A prereq; 3 regression tests (one per existing subcommand) before touching other subcommands |
| R-M4-4: `StepSuccessSchema.nextPrompt?` extension breaks M3 transport contract | LOW | HIGH | Optional field only; `contract/mcp-tool-count.test.ts` + envelope shape snapshot |
| R-M4-5: Hexagonal purity vs LangGraph framework coupling creates dep-cruiser false positives | MEDIUM | LOW | `graph/` labeled "LangGraph-bound" not "use-case"; `graph-no-adapters` + `handlers-no-ports` rules narrowly scoped |
| R-M4-6: Composite skill author writes non-consolidation-flavored `work` prose | LOW | LOW | Document composite-skill authoring pattern in **ADR 0006** (prompt assembly; ADR 0005 scopes out prompt semantics) + dogfooding-protocol skill-authoring template (Batch H); example `tdd-feature` is terminal-only — composite example deferred to M5 |
| R-M4-7: Dogfooding friction reveals Q-3a α choice wrong (prompt delivery shape) | MEDIUM | HIGH | Lock Q-3a in proposal Round 1; descope to γ (APIs only) if friction confirms |
| R-M4-8: 10 batches extend beyond 1.5-week spec estimate | MEDIUM | MEDIUM | Descoping rule: push `inspect` or `validate-skill` to M5 before cutting core port/adapter work |
| R-M4-9: Soft `validate-skill` warnings for undeclared `{{var}}` tokens don't fail the command, but missing vars still throw `PROMPT_ASSEMBLY_FAILED` at runtime — operators who ignore stderr warnings ship broken skills (Codex R4 Check 5) | MEDIUM | MEDIUM | Phase-var allowlist shipped alongside `view-mapping.ts` + documented in ADR 0005 §3; runtime error carries phase + missing-var list for fast diagnosis; Batch F acceptance asserts stderr contains warning lines on undeclared vars |
| R-M4-10: ADR 0005 status "Draft pending sign-off" persists post-proposal-lock — Q-1a grounding implicitly provisional until operator signs both (Codex R4 Check 3/4) | LOW | MEDIUM | Operator sign-off on ADR 0005 and proposal vN must be simultaneous; ADR status flips to Accepted in same commit |
| R-M4-11: `token_request` Phase 1 behavior unchanged from M3 — batch C1 dropped 2026-04-18 after operator reality-check ("foreman is infrastructure, doesn't call APIs"). Agents still receive `SCHEMA_VALIDATION_FAILED` on `token_request` until Phase 2a delivers real Vault/HSM integration | LOW | LOW | M3 stub + error code reserved-for-Phase-2a; Phase 2a ADR on token vending will revive Batch C1 scope with real backend |

## 6. Timing table

| Batch | Scope | Est. hours | Risk level |
|-------|-------|-----------|------------|
| A1 | ports/adapters dirs + Zod schemas + error codes + `StepSuccessSchema` extension + `NextPromptSchema` + channels `nextPrompt` + backward-compat test | 5-7 | MEDIUM |
| A2 | dep-cruiser rules (incl. I-13 `app.ts` composition-root carve-out rule) + --foreman-dir Scout (3 subcommands) + coverage threshold decision + `BuildGraphDeps` extension + node factory signature threading scaffold | 4-5 | LOW |
| B | `SkillMethodologyAssembler` adapter (pure, kind-blind) + `view-mapping.ts` | 4-5 | LOW |
| ~~C1~~ | ~~`step.ts` `token_request` unstub + `vendingHook` plumb~~ **DROPPED 2026-04-18** — deferred to Phase 2a per operator + GAN (R-M4-11) | 0 (dropped) | — |
| C2 | `app.ts` composition-root wiring + orphan-exclusion lift | 3-4 | LOW |
| D1 | plan-node assembler call-site + re-run regression test | 4-5 | MEDIUM |
| D2 | work + evaluate node call-site + cross-node integration | 5-9 (buffer for HIGH risk per R2 Codex L14) | HIGH |
| E | CLI `inspect` + journal projection | 4-5 | LOW |
| F | CLI `validate-skill` + YAML parse + stdout envelope | 3-4 | LOW |
| G | `examples/tdd-feature/skill.yaml` + `agent-setup.md` + npm packaging decision | 4-5 | LOW |
| H | ADR 0006 + spec/architecture updates + dogfooding-protocol + **skill-driven dogfooding run** | 6-8 | MEDIUM |
| I | post-WORK GAN + `m4-alpha` tag | 2-4 | LOW |
| **Total** | | **42-61 hours** (rebased 2026-04-18 after C1 drop: −3-4h) | |

Spec §3.5 estimates ~1.5 weeks. Codex Proposal final eval flagged original 34-49h as unrealistic vs M3 actual (~56h / 8 batches). **Rebased to 45-65h / 12 batches at v0.6** — closer to M3 density; accounts for ~40 new tests + human dogfooding gate + 12 batches vs M3's 8. Fits 1.5-2 weeks of focused work.

## 7. Batch layout with acceptance criteria

### Batch A1/A2 — types + ports/adapters scaffold + dep-cruiser + node factory threading

*(Per R2 H3 split, tightened in Round 3: A1 owns schemas/types/channels and `StepSuccessSchema`; A2 owns dep-cruiser, Scout fixes, and dependency-threading signatures. There is no duplicate threading ownership.)*

**A1 ships:** `ports/` + `adapters/` dirs (empty barrels); port DTO/Zod contracts; `error-codes.ts` +3 codes with count 8→11 + leading JSDoc rewrite; `err()` helper union widened; **existing `StepSuccessSchema` extended** with optional `nextPrompt?` field; `NextPromptSchema`; `channels.ts` `nextPrompt` per-invoke projection channel; `tests/unit/handler-backward-compat.test.ts`.

**A2 ships:** **`BuildGraphDeps` extended with `promptAssembler?`** and factory signatures threaded through `createPlanNode`/`createWorkNode`/`createEvaluateNode` (accepted but unused until Batches D1/D2); 5 new dep-cruiser named rules + `app.ts` carve-out + time-boxed ports orphan-exclusion; `--foreman-dir` inheritance bug Scout fix for `status`/`step`/`init` + regression test; coverage threshold decision.

**Acceptance (binary pass/fail gates per R1 Codex MEDIUM):**
- `pnpm exec depcruise` passes with 5 new rules (`ports-only-types-and-zod`, `adapters-no-transport-or-handlers`, `handlers-no-adapters`, `handlers-no-ports`, `graph-no-adapters`)
- `packages/solo/src/types/error-codes.ts` leading JSDoc comment says "Count: 11"
- All 10 existing `createHandlers` call sites compile unchanged (backward-compat test green)
- **`StepSuccessSchema` backward-compat**: existing `transport-handler.test.ts:434` literal-envelope assertion still passes, and a broader regression enumerates every current `ok:true` `step` variant to assert no double-wrap (`data.data`) and no `error` field on success. M4 prompt cases separately assert optional `nextPrompt` is additive-only.
- Node factories accept `promptAssembler?` param and compile without call-site updates beyond `buildGraph`; **node body invocation is intentionally deferred to D1/D2** to avoid a Batch A CI gate that depends on adapter/view-mapping behavior.
- **Sync-call acceptance gate** (R2 Codex Check 2, enforced in D1/D2, not A): node code performs a direct sync `promptAssembler.assemble(view)` call before `interrupt()` with no `await`/`Promise`/microtask boundary (enforced via lint rule or test assertion)
- `foreman status --foreman-dir /tmp/foo`, `foreman step --foreman-dir /tmp/foo`, `foreman init --foreman-dir /tmp/foo` all actually use `/tmp/foo` (3 regression tests green)
- **Coverage binary gate**: EITHER `packages/solo/vitest.config.ts` contains `test.coverage.thresholds` block (lines + branches + statements ≥80%) AND `pnpm test --coverage` exits 0, OR `docs/phase-1-solo/spec.md` G5 "≥80%" claim is struck in same batch (grep `spec.md` for `80%` returns zero matches against G5)

### Batch B — SkillMethodologyAssembler + view-mapping

**Ships:** `adapters/skill-methodology-assembler.ts` (pure, kind-blind, idempotent); `graph/view-mapping.ts` (domain→view projection helpers); `tests/unit/prompt-assembler-adapter.test.ts` (kind-blind matrix + purity); `tests/contract/prompt-assembler-port.test.ts` (shape + brand).

**Acceptance:**
- Fixture matrix across `{terminal, composite} × {plan, work, evaluate}` produces identical output when kind varies but methodology holds (I-10)
- N=100 invocations with identical inputs → identical output; zero side-effect assertions (I-14)
- ESLint/dep-cruiser regex bans `brief.kind` / `plan.kind` reads inside `adapters/skill-methodology-assembler.ts`

### Batch C1 — ~~`token_request` unstub + vendingHook~~ **DROPPED 2026-04-18**

**Status:** dropped per operator + GAN reality-check. foreman is infrastructure and does not call external APIs; Phase 1 passthrough vending provides zero security value vs the agent reading env directly. D-20 says promote seams only when real variation arrives → Phase 2a with Vault/HSM.

**M4 behavior:** M3 stub at `handlers/step.ts:313-321` stays in place; `token_request` signals return `SCHEMA_VALIDATION_FAILED` at Phase 1. `TOKEN_VENDING_FAILED` error code remains in catalog as forward-compat declaration (see §3.4 + `types/error-codes.ts` JSDoc).

**Phase 2a revival:** new ADR on token vending will define a real vending adapter (Vault/HSM/session-scoped) and the Batch C1 work plan will be reconstituted against that ADR.

### Batch C2 — app.ts composition + orphan-exclusion lift

**Ships:** `app.ts` wires `SkillMethodologyAssembler` into `buildGraph`, `passthroughEnvVendor` into `createHandlers`; `tests/contract/composition-root.test.ts` (I-13 enforcement); dep-cruiser `no-orphans` exclusion lifted for `ports/`.

**Acceptance:**
- `app.ts` is sole file importing from both `adapters/` and `ports/`
- Override path tested (composition root accepts alternate port implementations)
- `tests/contract/mcp-tool-count.test.ts` asserts surface = 2 tools

### Batch D1 — plan-node assembler + re-run regression

**Ships:** `graph/nodes/plan.ts` builds `PromptView` via `planView(state)` and invokes `promptAssembler.assemble(view)` BEFORE `interrupt()`; `tests/integration/graph-assembler.test.ts` (plan-only); `tests/integration/interrupt-resume-assembler.test.ts` modeled after existing `tests/integration/step-m3.test.ts:270-310` interrupt-snapshot pattern (Copilot H5).

**Acceptance:**
- Plan node returns `nextPrompt` in `StepSuccessSchema` envelope
- **N=100 consecutive resumes** on same plan produce byte-identical prompt (Codex MEDIUM — 3 was too weak; property-style regression)
- Absent assembler keeps the M3 path: no `nextPrompt`, no async boundary, no behavior change for existing initialize/plan/work/eval happy paths
- Handler invoke input clears `nextPrompt: null`; a non-prompt-emitting invoke after a prompt-emitting invoke cannot replay the prior prompt
- Mid-batch GAN: Codex + Copilot review re-run semantic before merging

### Batch D2 — work + evaluate nodes + cross-node integration

**Ships:** `graph/nodes/work.ts` + `graph/nodes/evaluate.ts` wired analogously; cross-node integration test; interrupt/resume regression extended to work phase.

**Acceptance:**
- All 3 phases (plan/work/evaluate) deliver assembled prompts via `nextPrompt`
- Composite parent uses `methodology.work` for consolidation prompt (Q-5 kind-blind pattern validated)
- Absent assembler guard remains consistent across work/evaluate: no `nextPrompt` and no changed M3 transition semantics

### Batch E — CLI `inspect`

**Ships:** `cli/inspect.ts` + `inspect` subcommand registration; `tests/integration/cli.test.ts` extended with inspect coverage.

**Acceptance:**
- `foreman inspect` dumps state + last N journal events as JSON
- `foreman inspect --pretty` produces multi-section human-readable output
- `foreman inspect --foreman-dir /tmp/...` honors the flag post-B1 fix

### Batch F — CLI `validate-skill`

**Ships:** `cli/validate-skill.ts`; YAML parse; stdout contract = `WireResponse<Skill>` JSON on success (enables §4 dogfooding `jq '.data'` piping — Codex + Copilot HIGH); `tests/integration/cli.test.ts` extended with validate-skill exit-code matrix (valid / malformed / missing file) + stdout shape assertion.

**Acceptance:**
- `foreman validate-skill examples/tdd-feature/skill.yaml` exits 0; stdout parses as `{ ok: true, data: Skill }` matching `SkillSchema`
- Undeclared template variable case exits 0 with stdout still `{ ok: true, data: Skill }`; warning goes to stderr
- `foreman validate-skill <malformed.yaml>` → exit 1; stdout `{ ok: false, error: { code: 'SKILL_VALIDATION_FAILED', ... } }`
- `foreman validate-skill <missing-file>` → exit 1; stderr contains path; stdout still valid `WireResponse<Skill>` error envelope
- `--foreman-dir` NOT declared on subcommand (skeleton Round 2 Copilot H1)

### Batch G — example skill + agent-setup docs

**Ships:** `examples/tdd-feature/skill.yaml` (creates `examples/` directory); `docs/agent-setup.md` (MCP config snippets for Claude Code + Copilot CLI + Cursor); npm packaging audit for `examples/` (Copilot H1).

**Acceptance:**
- `pnpm foreman validate-skill examples/tdd-feature/skill.yaml` exits 0
- `examples/` not matched by `.gitignore` (grep assertion); **location decision (R2 Copilot M2)**: since `examples/` lives at repo root (not inside `packages/solo/`), it does NOT currently ship in the npm tarball (`packages/solo/package.json:15-19` `files` = `["dist", "README.md", "LICENSE"]`). Batch G picks one: **(a)** relocate to `packages/solo/examples/` + add to `files`, **or (b)** keep at root and document npm-exclusion rationale. Default: (b) — examples are ops/dogfooding aids, not shipped artifacts
- `agent-setup.md` walkthrough passes objective gate: reviewer executes instructions on clean machine; `foreman__step type=initialize` returns `ok:true` and `nextPrompt` contains the tdd-feature plan methodology string (Copilot M4)

### Batch H — ADRs + docs + skill-driven dogfooding

**Ships:** `docs/adr/0006-prompt-assembler-port.md` (locked contract; flat vars bag shape per §3.7 (d) lock); `docs/phase-1-solo/spec.md` §8.4 update (`token_request` handler delegates to `vendingHook`) **and** one-pass rename of 10 occurrences of `Q-11a` / `Q-11b` → `Perf-Q-a` / `Perf-Q-b` throughout spec.md (Copilot R3 HIGH-2 + Codex R3 M8); architecture.md §3 concretization (dep-cruiser rule list + Option A call-site + "assembler is CQRS read-side" paragraph); dogfooding-protocol.md §12 shrink + skill-authoring template; **one full skill-driven dogfooding iteration logged to `docs/dogfooding-log.md`**.

**Acceptance:**
- ADR 0006 references final Q-2 (flat vars bag) + Q-3a (envelope α) + Q-11 (factory migration) locks
- `docs/phase-1-solo/spec.md`: `rg "Q-11[ab]"` returns zero matches outside M3 historical change-log entries; `rg "Perf-Q-[ab]"` finds the 10 renamed occurrences (grep-assertable)
- Dogfooding-log ≥1 entry following existing template (`### YYYY-MM-DD — <title>` header) **with at least one actionable friction item** (owner + disposition columns filled) OR an explicit "no friction encountered" line plus a second-reviewer sign-off (prevents vanity-logging per Codex + Copilot MEDIUM). **Gate type:** human review gate, not CI-enforceable; CI may grep for the entry/sign-off markers, but the owner/disposition quality is reviewed in Batch H post-WORK GAN.
- All cross-doc references verified (grep for stale "raw-signal dogfooding" post-M3 wording returns zero results outside M3 history)

### Batch I — post-WORK GAN + tag

**Ships:** Round N GAN review (Codex + Copilot retained); any drift fixes; `m4-alpha` tag.

**Acceptance:**
- All 3 reviewers (Codex + Copilot + architect optional) sign off
- `m4-alpha` tagged with annotated message documenting the invariants (I-10..I-15) materialized at M4
- No outstanding BLOCKERs or HIGHs

## 8. Invariants introduced at M4

- **I-10** "Assembler is kind-blind" — adapter signature excludes `brief.kind` / `plan.kind`; enforced by fixture-matrix test + ESLint/dep-cruiser regex
- **I-11** "Ports import only `types/` + `zod`" — dep-cruiser `ports-only-types-and-zod` rule
- **I-12** "No `TokenVendorPort` export before Phase 2a" — dep-cruiser bans `ports/token-vendor*.ts` pattern
- **I-13** "`app.ts` is the sole `adapters/` instantiator" — dep-cruiser: only `app.ts` may simultaneously import from both `adapters/` and `ports/`; `buildGraph` accepts `PromptAssemblerPort`, while `createHandlers` receives no ports and only accepts the plain M4 `vendingHook` function
- **I-14** "`PromptAssemblerPort` implementations are pure + idempotent" — implementations must NOT perform: network I/O, filesystem I/O, clock reads, randomness, input mutation, credential access. **Port is sync** (returns `AssembledPrompt` directly; no `Promise<>`) per Proposal R1 Codex HIGH — since purity forbids I/O, async serves no purpose. Test: N=100 invocations yield identical outputs with no side-effects.
- **I-15** "`PromptAssemblerPort` accepts a flat `PromptView` (methodology + vars bag) decoupled from domain schemas" — port never receives full `Brief`/`Plan`/`WorkReport`; vars are `string` values projected by view-mapping from phase-appropriate domain sources. Updated at v0.4 post-PromptView Round 2 GAN (d).
- **I-16** "View-mapping phase-isolation" — `view-mapping.ts` populates `view.methodology` from `skill.methodology[view.phase]` only (sibling methodology strings NOT leaked); `view.vars` keys are drawn exclusively from phase-relevant domain sources (e.g., `evaluate` phase gets `plan` + `work` vars; `plan` phase does NOT get `work`). Enforced by unit test over the projection helpers.
- **I-17** "`PromptView.vars` values are `string` only" — view-mapping serializes any structured data before injecting into the vars bag; the port never receives `Record<string, unknown>` or nested objects. Prevents creeping domain leakage into the template context.

## 9. Pre-v0.1 actions (blocking before this proposal locks)

1. Confirm M3-tag → M4-start window is captured in `docs/dogfooding-log.md` (1 entry as of 2026-04-18 per Copilot N2; no additional M4-relevant friction expected)
2. Confirm `.foreman/` materializes via clean-clone `pnpm install && pnpm build && foreman init`
3. Verify `scripts/demo.ts:38-51` still hydrates inline `brief.skill` per expected shape
4. Re-read ADR 0001 §1.1 — confirm `PromptAssemblerPort` exemplar scope does NOT lock port method shape (historically debated: 3-method vs single-polymorphic vs flat vars bag; resolved v0.4 to flat vars per PromptView Round 2 GAN (d))
5. ~~Author `docs/adr/0005-skill-file-format.md`~~ **DONE at v0.5** — Draft v1 authored 2026-04-18 locks YAML canonical + TS optional + `validate-skill` transport-only + runtime inline-only (D-18 reaffirmation). Pending operator sign-off. Q-1a grounding complete.

## 10. Non-goals (locked; 13 items)

1. Full preflight allowlist (M5)
2. Scenario test suite G4 closure (M5)
3. Performance benchmarks + spec `Perf-Q-a`/`Perf-Q-b` targets (M6) — *renamed from "Q-11a/b" per R2 Copilot M4 to eliminate naming collision with M4 Q-11 (`createHandlers` factory migration); spec.md update belongs in Batch H*
4. v1.0.0 packaging + README (M6)
5. TTL/stale detection (Phase 2a; v0.3 YAGNI)
6. Daemon / HTTP / RBAC / UI (Phase 2a+)
7. `refactor-extract` skill (M5 per roadmap.md:71)
8. Skill hot-reload (Phase 2a)
9. `TokenVendorPort` formal port (I-12; Phase 2a promotion trigger = Vault/HSM)
10. ADR 0001 §1.1 amendment (`PromptAssemblerPort` already mentioned as exemplar; method shape not locked by ADR 0001 — locked at v0.4 to flat vars bag per §3.7)
11. `SKILL_NOT_FOUND` error code (D-18)
12. `brief.skill` by reference (D-18)
13. `compositeWork` field on `SkillSchema` (I-10 kind-blind)

## 11. Open questions for proposal-lifecycle GAN rounds

**Q-1a LOCKED (Proposal v0.5 — grounded by ADR 0005):** YAML canonical + TS optional; `validate-skill` transport-only; runtime inline-only. See `docs/adr/0005-skill-file-format.md` (Draft v1 authored 2026-04-18).

**Q-1b LOCKED (Proposal R3 — adjusted for PromptView (d) lock):** `validate-skill` input depth = Zod parse + shallow `{{var}}` token extraction from the methodology string. Compare extracted tokens against the **phase-var allowlist** maintained alongside `view-mapping.ts` (not per-phase DTO fields, since (d) has none). Undeclared `{{var}}` references emit warnings to stderr while stdout remains `WireResponse<Skill>`; only schema/file parse failures emit `SKILL_VALIDATION_FAILED`. Runtime remains parsed inline `Skill` only.

**Q-2 LOCKED (Proposal R2 — PromptView GAN decision (d), v0.4):** `PromptAssemblerPort.assemble(view: PromptView)` with **flat vars bag**; sync return; no discriminated union. Shape: `{ phase, methodology, vars: Record<string,string> }`. Rationale: 3 states don't justify DTO proliferation; assembler is pure interpolation; Claude-Code parallel (template + context); narrowest port per D-20; runtime enforcement via `PROMPT_ASSEMBLY_FAILED` sufficient; Phase 2a migration additive (just inject vars). Full contract + adapter + migration snippet in §3.7. ADR 0006 codifies.

**Q-3a LOCKED (Proposal R2):** Option α — `StepSuccessSchema.nextPrompt?` envelope extension on the existing schema at `handlers/step.ts:41-52`. β rejected (no 3rd MCP tool; status-extension only as fallback if α proves unworkable mid-batch). γ is M4 goal rewrite. Trigger to switch α→β mid-implementation: if envelope extension causes transport-regression failures that can't be resolved without breaking G2 2-tool gate.

**Q-3b LOCKED** (conditional on α): `nextPrompt?: { phase, text }` on `StepSuccessSchema` only; signal schemas unchanged.

**Q-3c LOCKED:** Option A call-site; assembler runs BEFORE `interrupt()` in plan/work; purity (I-14) makes resume re-run safe.

**Q-4 LOCKED:** TokenVendor inline function only; no port.

**Q-5 LOCKED:** Kind-blind assembler; no `compositeWork` field.

**Q-7 LOCKED:** `tdd-feature` only at M4.

**Q-9 LOCKED (Proposal R1):** `inspect` output matches `WireResponse<{state: FlowState, journal: JournalEvent[]}>` on success (follows existing `cli/status.ts` envelope pattern per Copilot L3); `--pretty` wraps in human-readable sectioning without changing the JSON-schema contract.

**Q-10 LOCKED:** `inspect` honors `--foreman-dir`; `validate-skill` does NOT declare it.

**Q-11 (M4-Q11; createHandlers factory migration) LOCKED:** `createHandlers({ graph, projectId, vendingHook?, mutex?, now?, maxRetries? })`; `promptAssembler?` on `buildGraph`; all 10 existing call sites compile. *(Distinct from spec `Perf-Q-a/b` in §10 non-goal #3.)*

**Round 3 status (closed):** all substantive design questions resolved; the Q-1a grounding blocker (ADR 0005 absent) was closed at v0.5 when ADR 0005 was authored. Final integrity sweep applied; proposal ready for lock.

## 12. Next steps

1. **Pre-v0.1-lock checklist** (§9 items 1-5) — complete before proposal locks to vN final
2. ~~Proposal Round 1 GAN~~ (applied → v0.2; Q-2 + Q-9 locked)
3. ~~Proposal Round 2 GAN~~ (applied → v0.3; Q-3a + Q-1b unblock-path locked; `StepSuccessSchema` provenance corrected; `WireResponse` envelope; `channels.ts` addition; `init.ts` semantics)
4. ~~Proposal Round 3 final GAN~~ (applied → v0.4; all substantive design questions resolved; only residual = ADR 0005 grounding gap)
5. ~~PromptView Round 2 GAN~~ (applied → v0.4; Codex + Copilot unanimous flip to flat vars bag (d); §3.7 rewritten, I-15/I-16/I-17 revised, Q-2 amended)
6. ~~ADR 0005 authored~~ (applied → v0.5; Draft v1 at `docs/adr/0005-skill-file-format.md`; Q-1a grounded; Q-1b extraction contract ratified)
7. ~~Final integrity sweep~~ (applied → v0.5; orphan-reference cleanup, frontmatter + §11 Q-1a + §12 past-tense sync)
8. ~~Final evaluation round~~ (Codex + Copilot review) — Copilot: LOCK; Codex: v0.6 required. Applied Codex delta → **v0.6**: test paths scoped to `packages/solo/tests/`, timing rebased 34-49→45-65h, I-13 `composition-root-isolation` dep-cruiser rule added, `kind` exposure documented, §3.3 scaffold-vs-invocation split clarified, R-M4-6 composite ADR ref fixed, R-M4-9/R-M4-10 risks added
9. **Operator sign-off** — lock proposal to vN final (simultaneous ADR 0005 status flip Draft→Accepted per R-M4-10); author ADR 0006 against locked flat-vars-bag (d) shape (Batch H deliverable)
9. **Batch A starts** — per-batch mid-WORK + post-WORK GAN (M3 pattern)
10. **Round-N post-WORK sign-off** → tag `m4-alpha`

**ADR 0006 readiness (v0.5):** ADR 0006 is now **authorable** — all prerequisite Q-locks grounded (Q-1a via ADR 0005; Q-2 via PromptView R2; Q-3a/b/c via Proposal R1/R2; Q-4/5/7/9/10/11 all locked). Authoring tracked as Batch H deliverable against the locked flat `PromptView { phase, methodology, vars }` shape in §3.7.

## 13. Lineage

This proposal is the endpoint of a multi-stage planning pipeline. Cross-references in-text labeled "skeleton Round N X" refer to the skeleton-tier rounds below; references labeled "Proposal Round N X" refer to the proposal-tier rounds.

### Skeleton tier (retired; file preserved at `docs/phase-1-solo/m4-development-proposal-skeleton.md` for historical traceability)

- **Skeleton v0.1** — initial scope sketch; 10 open design questions
- **Skeleton Round 1 GAN** (Codex adversarial + Copilot cross-file + architect hexagonal) → folded 3 BLOCKERs + 7 HIGHs → **v0.2**
- **Skeleton Round 2 GAN** (same 3 reviewers) → 4 BLOCKERs + 8 HIGHs + 5 MEDIUMs folded → **v0.3**
- **Skeleton Round 3 GAN** (Codex + Copilot, hexagonal lens) → 2 BLOCKERs + 6 HIGHs folded → **v0.4** (skeleton graduated)

### Proposal tier (this file)

- **v0.1** — authored from skeleton v0.4; §1-§12 + Gates + Invariants + Risk register + Per-batch acceptance criteria + Timing table + Dogfooding integration
- **Proposal Round 1 GAN** (Codex + Copilot) → BLOCKER (Q-2 under-justified) + 3 HIGHs + 4 MEDIUMs → **v0.2** (Q-2 polymorphic single-method + sync port + validate-skill stdout)
- **Proposal Round 2 GAN** (Codex + Copilot) → BLOCKER (`StepSuccessSchema` provenance) + 6 HIGHs + 8 MEDIUMs → **v0.3** (`WireResponse<Skill>`, `channels.ts` `nextPrompt` channel, init.ts cascade, Batch A split A1/A2, §3.4 per-code expansion)
- **Proposal Round 3 final GAN** (Codex + Copilot, hexagonal lens) → 1 BLOCKER (ADR 0005 absent) + 6 HIGHs + 4 MEDIUMs → **v0.4 pre-ADR** (Zod in types/ not ports/, `handlers-no-ports` rule, `nextPrompt` per-invoke null-clearing)
- **PromptView Round 2 GAN** (ad-hoc, operator-prompted after reviewing 3-type vs generic options) → Codex + Copilot unanimous flip to flat vars bag (d) → **v0.4** (§3.7 rewrite, I-15/I-16/I-17 revised, Q-2 amended)
- **ADR 0005 authored** → **v0.5** (Q-1a + Q-1b grounded; ADR 0006 prerequisites closed)
- **Final integrity sweep** (Codex + Copilot completeness) → text-sync amendments only → **v0.5** (this file)

### Totals

- **6 skeleton-tier agent-rounds** + **7 proposal-tier agent-rounds** + **1 PromptView-specific round** + **1 final integrity sweep** = **15 agent-rounds** across planning lifecycle
- Net new code at M4: 1 port, 2 adapters, 1 view-mapping helper, 2 CLI subcommands, 1 example skill, ~40 tests
- Net new invariants: I-10..I-17 (8 invariants)
- Net new ADRs: 0005 (authored) + 0006 (Batch H deliverable)

Batch A starts post operator sign-off.
