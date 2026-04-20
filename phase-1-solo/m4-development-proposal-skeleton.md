---
status: HISTORICAL — skeleton graduated to m4-development-proposal.md (which was later superseded-in-part by ADR 0007). Banner added 2026-04-19 as part of M5 docs refactor.
milestone: M4 — Skills (first formal port + skill YAML + 1 example skill + skill-driven dogfooding)
date: 2026-04-18
grounding: spec §3.5 (M4 plan); D-17/D-18/D-20/D-21 (foundations); architecture.md §3.3 + §3 Scenario 3; roadmap Gate 6 hedge; dogfooding-protocol.md §12 (M4 deferrals)
depends_on: m3-alpha tagged on 9a859ab (local; not pushed); dogfooding kicked off at M3 close
round_1_gan: 3-agent review (Codex adversarial / Copilot cross-file / architect hexagonal) — 3 BLOCKERs + 7 HIGHs folded into v0.2
round_2_gan: same 3 reviewers on v0.2 — 4 BLOCKERs + 8 HIGHs + 5 MEDIUMs folded into v0.3
round_3_gan: Codex + Copilot hexagonal final — 2 BLOCKERs + 6 HIGHs folded into v0.4 (this file)
---

> **⚠ HISTORICAL — planning artifact, superseded.**
>
> This skeleton graduated to [`m4-development-proposal.md`](m4-development-proposal.md) which was itself later amended by [ADR 0007 (domain-blind core)](../adr/0007-domain-blind-core.md). Retained for git-blame traceability. Current milestone: [`m5-development-proposal.md`](m5-development-proposal.md).

# M4 Development Proposal — Skeleton v0.4 (Round 3 GAN applied)

Per M3 lifecycle: skeleton → v0.1 proposal draft → GAN rounds → vN final → code batches. Round 3 verdicts: Copilot said "GRADUATES with editorial amendments"; Codex said "NOT ready — 2 BLOCKERs". Split verdict reconciled by folding both sets in-line. v0.4 is the final skeleton; next step is promotion to `docs/phase-1-solo/m4-development-proposal.md` v0.1.

## 1. Goal

M4 materializes **D-18 (skill injection)** and **D-20 (narrow port discipline)** with the first formal outbound port (`PromptAssemblerPort`) and one production-ready example skill (`tdd-feature`). Dogfooding switches from **raw-signal mode** (M3 close) to **skill-driven mode**.

### 1.1 Runtime/authoring split (Round 1 BLOCKER — Round 2 tightened)

**Runtime:** `brief.skill` is always a full inline `Skill` object matching M3-shipped `SkillSchema` at `packages/solo/src/types/skill.ts:20-26` (`{ name, methodology: { plan, work, evaluate }, scope? }`). Foreman never reads skill files at runtime per D-18. Runtime adapters accept **only** the parsed inline `Skill` from `brief.skill`. (Round 2 Codex L1: prior "adapters accept YAML or TS" wording weakened this boundary — corrected.)

**Authoring/distribution:** Skill authors publish YAML files (canonical) or TS modules (advanced). Wrappers/CLI (e.g., `foreman validate-skill`) parse YAML/TS into a `Skill` object; they never hand raw YAML to runtime. `foreman validate-skill <path>` is a **transport-layer dev tool** that does **not** depend on `PromptAssemblerPort`.

**Token vending:** `token_request` handler (stubbed as `SCHEMA_VALIDATION_FAILED` at M3) becomes a real pass-through env-vending hook. Implemented as a plain injected function on `createHandlers`, **not** as a formal port per D-20's non-port-seam table.

Secondary goal: ship CLI `inspect` + `validate-skill` subcommands that M3 dogfooding identified as useful for operator debugging + skill authoring.

## 2. Scope

### 2.1 Core (M4 primary scope)

- `packages/solo/src/ports/prompt-assembler.ts` — **`PromptAssemblerPort`** interface (first formal port per D-20). **Pure query-only port (CQRS read-side)** per I-14. Implementation takes methodology strings + a narrow phase context; returns an `AssembledPrompt` value.
- `packages/solo/src/adapters/skill-methodology-assembler.ts` — default adapter interpolating `brief.skill.methodology.{plan, work, evaluate}` into prompts. **Pure + idempotent + kind-blind** per I-10, I-14. Does **not** accept `brief.kind` or `plan.kind` as signature inputs (Round 2 Architect MUST: kind exposure via template variables only, never as branching input).
- `packages/solo/src/adapters/passthrough-env-vendor.ts` — token-vending default; **not port-shaped** — plain injected function, no class/interface adapter contract. Naming intentionally avoids `Port`/`Adapter` suffixes.
- `packages/solo/src/graph/nodes/{plan,work,evaluate}.ts` — invoke `promptAssembler` at node body (**Option A call-site** per Q-3c); assembler runs BEFORE `interrupt()` in plan/work nodes so prompt exists at pause time. Purity (I-14) makes re-run on resume safe.
- `packages/solo/src/handlers/step.ts` — `token_request` case: replace M3 `SCHEMA_VALIDATION_FAILED` stub with `vendingHook` dispatch; **widen `err()` helper union** to include new M4 error codes (Round 2 Copilot H2).
- `packages/solo/src/graph/build.ts` — `buildGraph` factory grows optional `promptAssembler?` dep (Round 2 Codex BLOCKER: call-site is graph nodes → DI lives in graph construction, NOT `createHandlers`).
- `packages/solo/src/handlers/index.ts` — `createHandlers` grows **optional `vendingHook?`** dep only; `promptAssembler` stays out. Backward-compat gate: all 10 existing call sites (3 prod/script + 7 test) compile unchanged (CI test).
- `packages/solo/src/app.ts` — composition root wires `SkillMethodologyAssembler` into `buildGraph` + `passthroughEnvVendor` into `createHandlers`. **Sole adapters↔ports binding site** per I-13.

### 2.2 Error-code deltas (Round 1 HIGH)

Add to `packages/solo/src/types/error-codes.ts`:
- `TOKEN_VENDING_FAILED` — replaces stub in `handlers/step.ts:305-314` for `token_request` failures (SecurityError per D-5)
- `PROMPT_ASSEMBLY_FAILED` — adapter throws on missing/malformed methodology interpolation
- `SKILL_VALIDATION_FAILED` — CLI-only code for `validate-skill` failures

Batch A must also update:
- Leading comment block at `types/error-codes.ts:3-11` ("Count remains 8" → 11)
- `err()` helper union at `handlers/step.ts:358-371` to accept new codes

Explicit non-addition: **no `SKILL_NOT_FOUND`** — per D-18, runtime cannot have this condition.

### 2.3 Dep-cruiser layer rules (Round 1 HIGH — Round 2 named-rules update)

`.dependency-cruiser.cjs` already has: `types-only-zod`, `errors-only-types`, `graph-no-downstream`, `handlers-no-transport`, `mcp-no-graph-direct`, `cli-no-graph-direct`, `no-orphans`. M4 adds:

- **`ports-only-types-and-zod`** — `ports/**` may import only `types/`, `zod`
- **`adapters-no-transport-or-handlers`** — `adapters/**` may NOT import `graph/|handlers/|mcp/|cli/`
- **`handlers-no-adapters`** — `handlers/**` may NOT reach `adapters/**` directly (only via ports)
- **`graph-ports-allowed`** — amend `graph-no-downstream` so `graph/` may import `ports/` (not `adapters/`)
- **Ports orphan-exclusion (Batch A-only, time-boxed per R3-9)** — `no-orphans` temporarily excludes new `ports/**` files during Batch A scaffold; exclusion lifts once composition wiring lands in Batch C2. Permanent orphan-exclusion would be a hexagonal smell (unused port = architectural debt).

Layer direction (hexagonal labels, Round 2 Architect SHOULD):

| Layer | Role | May import | May NOT import |
|---|---|---|---|
| `ports/` | port contracts | `types/`, `zod` | everything else |
| `adapters/` | infrastructure adapters | `ports/`, `types/`, `errors/`, framework SDKs | `graph/`, `handlers/`, `mcp/`, `cli/` |
| `graph/` | **application orchestration / LangGraph-bound adapter layer** (not a pure hexagonal use-case layer — framework SDK lives here intentionally per ADR 0002) | `ports/`, `types/`, `errors/`, `@langchain/langgraph` | `adapters/`, `handlers/`, `mcp/`, `cli/` |
| `handlers/` | transport-adjacent request handlers | `ports/`, `types/`, `errors/`, `graph/` (compiled) | `adapters/`, `mcp/`, `cli/` |
| `mcp/`, `cli/` | transport delivery | `handlers/`, `types/`, `errors/` | `graph/`, `adapters/`, `ports/` directly |
| `app.ts` | **composition root** — I-13 sole **instantiator** of `adapters/` concretions; exempt from `handlers-no-adapters` + `adapters-no-transport-or-handlers` rules via carve-out | all | — |

### 2.4 CLI (deferred-from-M3 scope)

- `packages/solo/src/cli/inspect.ts` — `foreman inspect [--node <id>]`: dump full state + last N journal events as JSON. Honors `--foreman-dir` (reads checkpoint state) **after** the Scout follow-up below fixes the pre-existing bug.
- `packages/solo/src/cli/validate-skill.ts` — `foreman validate-skill <path>`: Zod-parse a skill YAML/TS file; report errors; exit 0/1. **Does NOT declare `--foreman-dir`** (Round 2 Copilot H1: "accept but ignore" is antipattern; `validate-skill` resolves `<path>` relative to `cwd`, no checkpoint access needed).
- Update `cli/index.ts` + `app.ts` to register both subcommands.

**Round 2 Copilot BLOCKER B1 — Scout follow-up (Batch A prereq):** `--foreman-dir` is declared at program level (`cli/index.ts:21`) but subcommand `.action()` callbacks read `opts.foremanDir` from the subcommand opts object. Commander program-level options live on `program.opts()`, not per-subcommand — **the flag is silently ignored today**. No test covers it. Batch A must either (a) re-declare the flag per subcommand, or (b) read via `program.opts()` inside each action. Add regression test covering status/step/inspect.

### 2.5 Skills

- `examples/` directory **created by Batch G** — does not currently exist in repo (Round 2 Copilot M4 verified)
- `examples/tdd-feature/skill.yaml` — first example skill methodology (YAML canonical per Q-1a proposed lock)
- **Non-goal:** `examples/refactor-extract/skill.*` deferred to M5 per roadmap.md:71 (see §6)

### 2.6 Docs + ADRs (Round 2 Codex HIGH + Architect MUST)

- `docs/agent-setup.md` — MCP config snippets for Claude Code, Copilot CLI, Cursor (npx warning per architecture §7.1)
- `docs/dogfooding-protocol.md` — mark `inspect` + `validate-skill` as shipped; shrink §12 deferrals; add skill-authoring friction template
- `docs/phase-1-solo/spec.md` §8.4 `token_request` row — M4: handler delegates to `vendingHook`
- `docs/phase-1-solo/architecture.md` §3 — concretize; add dep-cruiser rule-name list + Option A call-site + **"assembler is query-only / CQRS read-side" paragraph** (Round 2 Architect SHOULD Q#9)

**ADR sequencing (Round 3 BLOCKER R3-10 — claims demoted to pre-v0.1 actions):**
- **`docs/adr/0005-skill-file-format.md`** — **pre-v0.1 action, NOT yet authored** (see §10 item 5). Will lock: YAML canonical + TS optional; `validate-skill` transport-only; runtime inline-only (D-18 reaffirmation). Prerequisites already met (D-18 + Q-1a direction).
- **`docs/adr/0006-prompt-assembler-port.md`** — **post-Round-3 proposal-v0.1 action** (authored during proposal-lifecycle GAN rounds). Must cover: contract shape; Option A call-site; kind-blind consolidation (I-10); pure/idempotent (I-14) with prohibition list; `AssembledPrompt` branded type + `makeAssembledPrompt()` boundary (Round 3 R3-2); prompt-view DTO mapping location.
- **No ADR 0001 amendment** — ADR 0001 §1.1 row mentions `PromptAssemblerPort` only as an exemplar; it does NOT lock the 3-method vs single-method shape.

## 3. Open questions remaining for Round 3

Round 2 locked or closed several; remaining open questions:

**Q-1a Skill distribution format: YAML canonical + TS optional.**
ADR 0005 authors the lock during Round 2→3. Round 3 GAN verifies.

**Q-1b `validate-skill` input depth? — depends on Q-2 DTO lock.**
Zod parse only (shape) vs deeper (template-interpolation sanity). Proposed: Zod parse + template-variable extraction (warn on undeclared `{{var}}`). **Q-1b deep-check resolution DEPENDS on Q-2 `*PromptView` DTO field names** (template vars declared by DTO shape). Proposal v0.1 must lock Q-2 before finalizing Q-1b.

**Q-2 `PromptAssemblerPort` contract shape — OPEN; Round 3 BLOCKER R3-1/R3-7 rewrite.**
3-method (phase-specific type narrowing) vs single polymorphic `assemble(phase, context)`. Round 2 Codex clarified: ADR 0001 does NOT lock this. Round 3 Codex BLOCKER: passing full `Brief`/`Plan`/`WorkReport` still couples the port to domain evolution, defeating D-20's "narrow port" discipline and breaking Phase 2a mechanical-swap claim.

**Revised draft — stable prompt-view projections (not domain objects):**

```typescript
// Prompt-view DTOs: stable, readonly, contain ONLY template-safe fields.
// Decoupled from Brief/Plan/WorkReport evolution per I-15 below.
interface PlanPromptView {
  readonly methodology: { readonly plan: string };
  readonly goal: string;
  readonly scope?: { readonly readOnly: boolean; readonly paths: readonly string[] };
}

interface WorkPromptView {
  readonly methodology: { readonly work: string };
  readonly goal: string;
  readonly planSummary: string;
  readonly scope?: { readonly readOnly: boolean; readonly paths: readonly string[] };
}

interface EvalPromptView {
  readonly methodology: { readonly evaluate: string };
  readonly goal: string;
  readonly planSummary: string;
  readonly workSummary: string;
}

type AssembledPrompt = string & { readonly __brand: 'AssembledPrompt' };

interface PromptAssemblerPort {
  assemblePlanPrompt(view: PlanPromptView): Promise<AssembledPrompt>;
  assembleWorkPrompt(view: WorkPromptView): Promise<AssembledPrompt>;
  assembleEvalPrompt(view: EvalPromptView): Promise<AssembledPrompt>;
}
```

Domain→view mapping happens inside graph nodes (use-case layer), not inside the assembler adapter. Phase 2a tier-specific assemblers receive the same `*PromptView` DTOs — truly mechanical swap. Proposal v0.1 must specify the mapping function location + test coverage.

Notably: no `brief.kind` / `plan.kind` parameter (Round 2 Architect MUST + I-10) — assembler is kind-blind; kind exposed as template variable inside `methodology.*` strings only.

**Q-3a Assembled-prompt return-shape preference — Round 2 narrowed to α or γ.**
Option α: extend `StepSuccessSchema` with `nextPrompt?: { phase, text }` field — delivers assembled prompt at pause time.
Option β (REJECTED): new parallel read tool would break G2 two-tool gate; Codex MEDIUM says β may only mean "extend `foreman__status`", never a third tool.
Option γ: ship assembler APIs/tests only; no runtime prompt delivery (defers skill-driven dogfooding to M5 — Round 2 Codex flags this as a goal rewrite).

**Proposed lock for Round 3: Option α + §1 skill-driven-dogfooding goal intact.** β-as-status-extension is a fallback if α breaks schema stability.

**Q-3b If α, envelope extension details?**
`nextPrompt?: { phase: 'plan'|'work'|'evaluate', text: AssembledPrompt }` on `StepSuccessSchema` only. Signal schemas (agent→foreman) remain unchanged.

**Q-3c Call-site: Option A LOCKED + interrupt-semantic clarified.**
Plan/work nodes: assembler runs BEFORE `interrupt()` so the prompt is available at pause. On resume, the node body re-runs from the top (LangGraph semantic that burned M3); the assembler's purity + idempotency (I-14) makes this safe — same inputs → same prompt; no duplicate side-effects.
Evaluate node: has no interrupt; assembler runs when handler dispatches evaluate.
Revised wording for rejected alternatives (Round 2 Codex MEDIUM): "Option B (handler pre-invoke) rejected for weaker transition locality; D-17/D-20 do not themselves forbid it. Option C (outside foreman) defeats D-20's justification."

**Q-4 TokenVendor remains inline function — LOCKED per D-20 + I-12.**

**Q-5 Composite + skills: kind-blind assembler LOCKED.**
Assembler signature excludes `brief.kind` / `plan.kind`. Composite-capable skills author a consolidation-flavored `methodology.work`. No `compositeWork` schema field.

**Q-7 Example skill: `tdd-feature` only at M4. LOCKED.**

**Q-9 CLI `inspect` output: JSON canonical + `--pretty` flag for formatted.**
Round 3 to confirm structure (extend `status` envelope + journal array).

**Q-10 CLI `--foreman-dir` subcommand behavior — LOCKED with B1 caveat.**
`inspect`: honors it (but first fix B1 pre-existing bug).
`validate-skill`: does not declare it at all.

**Q-11 `createHandlers` signature migration — LOCKED (Round 2 Codex BLOCKER).**
**Option i: optional deps on existing `createHandlers({ graph, projectId, vendingHook?, mutex?, now?, maxRetries? })`.** `promptAssembler?` moves to `buildGraph({ checkpointer, now, promptAssembler? })` since call-site is graph nodes. Backward-compat test gate: all 11 existing call sites compile unchanged.

## 3.6 Test inventory (Round 2 Copilot M3 + Codex M1 — mapped to existing test dirs)

Existing dirs: `packages/solo/tests/{unit,integration,contract,scenario}/`.

- **`tests/unit/prompt-assembler-adapter.test.ts`** — interpolation, kind-blind (I-10) fixture matrix, purity/idempotency (I-14)
- **`tests/contract/prompt-assembler-port.test.ts`** — port shape fixtures, `AssembledPrompt` branded-type assertion
- **`tests/integration/graph-assembler.test.ts`** — Option A: assembler called inside plan/work/evaluate nodes with correct phase context; integration with `buildGraph` DI
- **`tests/integration/interrupt-resume-assembler.test.ts`** — call-count regression: N resumes produce identical prompt + no duplicate side-effects
- **`tests/integration/handlers-extra.test.ts`** (extend) — `token_request` dispatches through `vendingHook`; happy + error paths
- **`tests/integration/cli.test.ts`** (extend) — `inspect` JSON + `--pretty` + `--foreman-dir` regression (post-B1 fix); `validate-skill` exit-code matrix
- **`tests/unit/handler-backward-compat.test.ts`** — `createHandlers` with no new deps still compiles + runs M3 scenarios
- **`tests/contract/mcp-tool-count.test.ts`** (new) — transport regression asserting MCP surface = 2 tools
- **`tests/contract/composition-root.test.ts`** — `app.ts` wires defaults; override path tested; I-13 enforcement
- **CI gate:** `pnpm exec depcruise --config .dependency-cruiser.cjs packages/solo/src` asserts new named rules pass

Add handler-level concurrent-signal smoke (Copilot Round 10 LOW) into `tests/unit/handler-backward-compat.test.ts`.

## 4. Dependencies from M3

- `SkillSchema` / `BriefSchema` — unchanged
- `StepSignalSchema` — unchanged (agent→foreman signals not affected; prompts are foreman→agent in envelope α)
- `StepSuccessSchema` — extended with optional `nextPrompt?` field (Q-3a α lock pending Round 3)
- `handlers/step.ts` — `token_request` unstub; `err()` helper union widened
- `handlers/index.ts` — `createHandlers` grows optional `vendingHook?` only (Q-11 locked)
- `graph/build.ts` — `buildGraph` grows optional `promptAssembler?` (Round 2 Codex BLOCKER: correct injection site)
- `.dependency-cruiser.cjs` — add 4 named rules + orphan-exclusion (§2.3)
- `app.ts` — wires `SkillMethodologyAssembler` into graph + `passthroughEnvVendor` into handlers; I-13 sole binding site
- `types/error-codes.ts` — +3 codes; comment block count 8→11

## 5. Gates touched

- **G2** — MCP surface stable (2 tools; asserted by `tests/contract/mcp-tool-count.test.ts`)
- **G6** — S-1 "no ambient credentials" materialized (token vending real)
- **G9** — `agent-setup.md` + 1 skill partial; full to close at M6
- Possibly **G4** — skill-driven scenario variants may land here or defer to M5

## 6. Explicit non-goals (locked; 13 items)

1. Full preflight allowlist (M5)
2. Scenario test suite G4 closure (M5)
3. Performance benchmarks + spec Q-11a/b targets (M6) — **note: "Q-11a/b" here refers to SPEC perf targets, NOT M4 Q-11 createHandlers migration** (Round 2 Codex NIT)
4. v1.0.0 packaging + README (M6)
5. TTL/stale detection (Phase 2a; v0.3 YAGNI)
6. Daemon / HTTP / RBAC / UI (Phase 2a+)
7. `refactor-extract` skill (M5 per roadmap.md:71)
8. Skill hot-reload (Phase 2a)
9. `TokenVendorPort` formal port (I-12; Phase 2a promotion trigger = Vault/HSM)
10. ADR 0001 §1.1 amendment (`PromptAssemblerPort` already mentioned; 3-method shape not locked by 0001)
11. `SKILL_NOT_FOUND` error code (D-18)
12. `brief.skill` by reference (D-18)
13. `compositeWork` field on `SkillSchema` (I-10 kind-blind)

## 6.1 New invariants introduced at M4

- **I-10** "Assembler is kind-blind" — adapter signature excludes `brief.kind` / `plan.kind`; enforced by fixture-matrix test + ESLint/dep-cruiser regex banning `brief.kind` reads in `adapters/skill-methodology-assembler.ts`
- **I-11** "Ports import only `types/` + `zod`" — dep-cruiser `ports-only-types-and-zod` rule
- **I-12** "No `TokenVendorPort` export before Phase 2a" — dep-cruiser bans `ports/token-vendor*.ts` pattern until Phase 2a
- **I-13** "`app.ts` is the sole `adapters/`↔`ports/` binding site" — dep-cruiser: only `app.ts` may simultaneously import from both
- **I-14** "`PromptAssemblerPort` implementations are pure + idempotent" — adapter implementations **must not** perform: (a) network I/O, (b) filesystem I/O, (c) `Date.now()` / `performance.now()` or any clock reads, (d) `Math.random()` / crypto randomness, (e) object mutation of input DTOs, (f) credential / env-var access. Async is allowed only for interface uniformity with future-potentially-async variants. Test: N invocations with identical inputs yield identical outputs; no side-effects; enables safe re-run after LangGraph resume.
- **I-15** "PromptAssemblerPort DTOs decouple from domain evolution" — port accepts `*PromptView` stable projections, never full `Brief`/`Plan`/`WorkReport`. Domain→view mapping lives in graph nodes (use-case layer).

## 7. Timing sketch

Spec §3.5 estimates **~1.5 weeks**. Round 2 splits moved batch count 8 → 10 (Batch C → C1/C2, Batch D → D1/D2). Descoping rule if Round 3 or mid-WORK GAN expands scope: push CLI `inspect` or `validate-skill` to M5 before cutting core port/adapter work.

## 8. GAN sequencing

- Round 1 (applied) — 3 BLOCKERs + 7 HIGHs folded into v0.2
- Round 2 (applied) — 4 BLOCKERs + 8 HIGHs + 5 MEDIUMs folded into v0.3
- Round 3 (applied) — 2 BLOCKERs + 6 HIGHs folded into v0.4 (this file)
- **Next: v0.4 skeleton → `docs/phase-1-solo/m4-development-proposal.md` v0.1** (proposal-lifecycle promotion)
- Proposal v0.1 → vN — GAN rounds with Codex + Copilot (hexagonal lens retained); ADR 0005 authored pre-proposal; ADR 0006 authored during proposal-lifecycle once Q-2/Q-3a/Q-11 lock
- Batch A-I code — per-batch mid-WORK + post-WORK GAN (M3 pattern)
- Final — Round N post-WORK + tag `m4-alpha`

### 8.1 Concrete batch layout (Round 2 MEDIUM — splits applied)

- **Batch A** — types (ports/ + adapters/ dirs scaffold) + error codes (+comment fix + `err()` widening) + dep-cruiser named rules + `--foreman-dir` inheritance fix (Scout) + backward-compat test
- **Batch B** — `SkillMethodologyAssembler` adapter (pure, kind-blind)
- **Batch C1** — `step.ts` `token_request` unstub + `vendingHook` plumb via `StepHandlerDeps`
- **Batch C2** — `app.ts` composition-root wiring of `passthroughEnvVendor` + backward-compat validation
- **Batch D1** — plan-node assembler call-site integration + re-run regression test
- **Batch D2** — work + evaluate node call-site + cross-node integration test
- **Batch E** — CLI `inspect` + journal projection
- **Batch F** — CLI `validate-skill` + YAML parse
- **Batch G** — `examples/tdd-feature/skill.yaml` + `agent-setup.md` (creates `examples/` directory)
- **Batch H** — ADR 0005 (finalized) + ADR 0006 (finalized) + spec/architecture updates + dogfooding-protocol update
- **Batch I** — post-WORK GAN + `m4-alpha` tag

## 9. Scout follow-ups (inherited from M3 close + Round 2 discoveries)

- **`--foreman-dir` inheritance bug (Batch A prereq)** — Round 2 Copilot BLOCKER B1; pre-existing M3 bug; see §2.4
- **vitest coverage thresholds (Batch A decision)** — M3 §12 flagged G5 "≥80%" claim aspirational; Batch A must either add thresholds block to `vitest.config.ts` or strike the G5 claim. No further deferral.
- **Handler-level concurrent-signal smoke test** — fold into `tests/unit/handler-backward-compat.test.ts` at Batch A

## 10. Pre-v0.1 actions (not GAN questions)

1. Audit `docs/dogfooding-log.md` for M4-relevant friction between M3 tag and Batch A start
2. Confirm `.foreman/` materializes via clean-clone `pnpm install && pnpm build && foreman init` (Round 3 Copilot LOW-2 — `pnpm init` creates package.json, not what's needed)
3. Verify `scripts/demo.ts:38-51` still hydrates inline `brief.skill` per expected shape
4. Re-read ADR 0001 §1.1 line item for `PromptAssemblerPort` — confirm scope of what it does/doesn't lock (Round 2 Codex clarified: it does NOT lock 3-method shape)
5. **Author `docs/adr/0005-skill-file-format.md`** (Round 3 Codex R3-10 BLOCKER resolution): YAML canonical + TS optional; `validate-skill` transport-only; runtime inline-only (D-18 reaffirmation). No prerequisite Q lock — ready to draft.

## 11. Next action

**Promote v0.4 skeleton → `docs/phase-1-solo/m4-development-proposal.md` v0.1** (full proposal with risk register, per-batch acceptance criteria, timing table per Round 3 Copilot MEDIUM-4/5/6). Dispatch Round 1 GAN on proposal v0.1 (Codex + Copilot retained; same hexagonal lens). Author ADR 0005 as pre-v0.1 action; author ADR 0006 during proposal-lifecycle GAN rounds once Q-2/Q-3a/Q-11 final-lock.

Lifecycle:
1. Skeleton v0.1 → v0.2 (Round 1) → v0.3 (Round 2 ✅ — this file)
2. Round 3 final hexagonal GAN → v0.1 proposal draft
3. Proposal v0.1 → vN (operator approval)
4. Batch A starts; per-batch mid-WORK + post-WORK GAN
5. Round-N post-WORK sign-off → tag `m4-alpha`

No code until proposal matures. No skeleton commit until operator approves scope direction.
