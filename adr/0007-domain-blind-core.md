---
status: Accepted (operator reality-check 2026-04-18; codified and GAN-reviewed 2026-04-19)
date: 2026-04-19
deciders: Operator (xuzhijie), after mid-M4 reality-check + Codex + Copilot GAN concurrence
supersedes:
  - docs/adr/0006-prompt-assembler-port.md (slot vacated — ADR 0006 was planned but never authored; this ADR takes its place)
superseded_by: none
amends:
  - docs/foundations.md §D-20 (Phase 1 port exemplar retired; D-20 principle unchanged)
  - docs/phase-1-solo/m4-development-proposal.md §3.2 "Runtime/authoring split", §3.7 "PromptAssemblerPort contract", §8 "Invariants introduced at M4" (I-11..I-17 retired)
  - docs/adr/0003-state-machine-scope-and-mcp-surface.md §Consequences (M4 Batch C1 token_request unstub deferred to Phase 2a)
  - docs/adr/0005-skill-file-format.md (amendment note added; format + validate-skill contract unaffected)
related:
  - docs/adr/0001-architectural-style-declaration.md §1.1 (hexagonal narrow-scope declaration)
  - docs/adr/0002-langgraph-1x-adoption-and-deviations.md (LangGraph consumed directly; no FlowEnginePort)
  - docs/foundations.md P-1 (pure infrastructure), D-17 (handler/transport separation), D-18 (skill injection), D-20 (narrow-port discipline)
  - docs/phase-1-solo/m4-development-proposal.md (historical pre-pivot plan; superseded in part by this ADR)
  - docs/phase-1-solo/m5-development-proposal.md (M5 absorbs un-shipped M4 remainder under the domain-blind architecture)
---

# ADR 0007: Domain-blind core — foreman does not render prompts at runtime

## Status and scope

Accepted 2026-04-19 after operator reality-check on 2026-04-18 and concurrent GAN review. Codifies the mid-M4 pivot that reverted `PromptAssemblerPort` + `DefaultPromptAssembler` + `PromptView` + `view-mapping.ts` + `nextPrompt` channel + the 6 hexagonal dep-cruiser rules that gated them.

**Scope:** the runtime boundary between foreman (harness) and the agent (contractor). Specifically, who is responsible for turning `brief.playbook.phases[phase]` strings into concrete agent prompts.

**Out of scope:** the skill file format (ADR 0005 stands as-is); the 2-tool MCP surface (ADR 0003 stands); composite thread routing (ADR 0004 stands); LangGraph engine adoption (ADR 0002 stands).

## Context

### The M4 proposal (locked 2026-04-18) introduced `PromptAssemblerPort`

Proposal `docs/phase-1-solo/m4-development-proposal.md` (v0.6 locked 2026-04-18 after 15 agent-rounds across skeleton + proposal + integrity sweeps) made `PromptAssemblerPort` the Phase 1 exemplar for D-20 narrow-port discipline. Default adapter `DefaultPromptAssembler` (originally `SkillMethodologyAssembler`) would interpolate `view.methodology` against a flat `view.vars: Record<string, string>` bag inside plan/work/evaluate graph nodes, BEFORE the LangGraph `interrupt()`, and deliver the result to the agent via an optional `StepSuccessSchema.nextPrompt` envelope field.

Batches A1 + A2 + B + C2 shipped over 2026-04-18/19:

- `packages/solo/src/ports/prompt-assembler.ts` — port interface (27 lines)
- `packages/solo/src/types/prompt-view.ts` — flat `PromptView` + `AssembledPrompt` branded type (69 lines)
- `packages/solo/src/adapters/default-prompt-assembler.ts` — default adapter (67 lines)
- `packages/solo/src/graph/channels.ts` — extended `inputSignal` wiring + retired-at-pivot `nextPrompt` commentary
- `.dependency-cruiser.cjs` — 6 hexagonal rules (`ports-only-types-and-zod`, `adapters-no-transport-or-handlers`, `handlers-no-adapters`, `handlers-no-ports`, `graph-no-adapters`, `composition-root-isolation`)
- Composition-root wiring in `packages/solo/src/app.ts`
- Contract + unit tests (`tests/contract/prompt-assembler-port.test.ts`, `tests/unit/prompt-assembler-adapter.test.ts`)
- M4 error codes `PROMPT_ASSEMBLY_FAILED` (+ reserved `TOKEN_VENDING_FAILED` + `SKILL_VALIDATION_FAILED`)

### The reality-check

Operator statements 2026-04-18 (verbatim across a series of messages):

> "foreman is an infrastructure, and it never call api directly"
>
> "foreman is more state machine, because pure skill always drift, however, domain knowledge are always independent"
>
> "i have toald you look at harness plugin which is a early poc of foreman"

Key realization: **prompt rendering is a domain-content operation** — it takes domain state (plan, work report, prior evaluation) and produces text that *instructs an agent what to do next*. That is identical to what an agent does when it decides how to act on a prompt. By owning prompt rendering, foreman was becoming a trivial agent (P-1 violation: "foreman is pure infrastructure; never interprets domain content").

The pivot relocates the rendering step to the agent layer: foreman emits raw `brief.playbook.phases[phase]` verbatim through the existing `brief` channel, and the agent reads it via `foreman__status` and interpolates placeholders (`{{var}}`, `${var}`, or any convention) using its own state context.

## Decision

Foreman is a **domain-blind state machine**. It does not render prompts. The step envelope carries raw playbook content; external agents own rendering.

Concretely:

1. **Retire `PromptAssemblerPort`, `DefaultPromptAssembler`, `PromptView`, `view-mapping.ts`, `nextPrompt` channel, `StepSuccessSchema.nextPrompt` field, `PROMPT_ASSEMBLY_FAILED` error code, and the 6 M4 hexagonal dep-cruiser rules.** Reverted files preserved in git history; revert commit references this ADR.
2. **Retain D-20 narrow-port discipline unchanged.** The exemplar is retired because the capability it modeled belongs on the agent side of the semantic boundary, not because the discipline was wrong. D-20 now yields *zero* explicit outbound ports at Phase 1. First real port will be `TokenVendingPort` at Phase 2a.
3. **Retain `Playbook` schema, `brief.playbook` runtime contract, D-18 playbook injection, and ADR 0005 playbook file format.** The *what* agents see (phase strings) is unchanged; the *who* renders them (agent, not foreman) shifts.
4. **Redirect the M4 proposal's `inspect` + `validate-playbook` + example-playbook + `agent-setup.md` deliverables to M5.** `validate-playbook` was already a transport-layer dev tool with no runtime port coupling, so its contract is unchanged — it just ships later.
5. **Phase 2a hosted-service moat (scoped token vending, roadmap Gate 6) is unchanged.** The `TokenRequest` signal type remains in `StepSignalSchema` for forward-compat; M4 Batch C1 (pass-through unstub) dropped to Phase 2a alongside the real Vault/HSM adapter. See ADR 0003 amendment.

## Rationale

**Why the pivot is correct**

- **P-1 alignment.** Prompt rendering operates on domain content (plan strings, work reports, evaluations). Any component that renders a prompt is interpreting that content to produce instructions — that is agent behavior, not harness behavior. Having foreman do it creates a trivial-agent inside the harness and weakens P-1.
- **D-20 honored, not violated.** The discipline says "ports exist where real variation lives or a proven alternate is coming." The pre-pivot `PromptAssemblerPort` assumed tier-specific assembly overrides at Phase 2+. But the real variation isn't assembly logic — it's *which agent* with *which placeholder convention*. Agent-side rendering absorbs all that variation without a foreman port.
- **I-14 preserved for free.** Pre-pivot we were going to enforce assembler purity (no I/O, no clock, no randomness) via invariants I-14/I-15/I-16/I-17 and N=100 idempotency regression tests. With the assembler gone, foreman has no place where a domain-content function could be tempted to reach for I/O. The invariant is upheld by absence.
- **Harness-plugin POC alignment.** The operator-referenced "harness plugin" is explicitly described as a "domain-blind core harness" — this ADR matches foreman to that model.

**Why the revert is safe**

- Tests: 281/281 pass after revert (confirmed pre-commit). Depcruise: 0 violations (53 modules, 126 deps). Typecheck + lint clean.
- Revert blast radius: 14 files (7 staged deletions + 7 modifications); net -568 lines after the code revert completed with ref-cleanup.
- No agent-visible breakage: M4 never had a production consumer of the post-pivot change (agents still use the M3 envelope shape; no `nextPrompt` field was ever shipped).
- M4 landed scope is narrow but real: A2 Scout (`--foreman-dir` propagation fix across `init` / `status` / `step`), A3 spike decision (keep `replaceReducer` + `Annotation inputSignal` — deferred framework migration to Phase 2a), `NodeKind` → `TaskKind` rename, ADR 0005 (skill file format) Accepted.

**Why this ADR not a proposal amendment**

M4 proposal body is retained as historical record of the pre-pivot plan (with a supersession banner). This ADR codifies the decision surface — durable, grep-able, and referenced by every updated doc. ADRs outlive proposals.

## Consequences

### Immediate (committed alongside this ADR)

- `packages/solo/src/{ports,adapters}/` directories removed.
- `types/prompt-view.ts` + `tests/{contract,unit}/prompt-assembler*.test.ts` removed.
- `StepSuccessSchema.nextPrompt`, `NextPromptSchema`, `nextPrompt` channel removed; `types/signal.ts` + `handlers/step.ts` no longer reference them.
- `ERROR_CODE_VALUES`: `PROMPT_ASSEMBLY_FAILED` dropped (11 → 10 net; `TOKEN_VENDING_FAILED` + `SKILL_VALIDATION_FAILED` reserved forward-compat).
- `.dependency-cruiser.cjs`: 6 M4 hexagonal rules retired; M2 layer rules + external-dep containment + `no-orphans` preserved.
- `tests/fixtures/skills/index.ts`: `makeView` + `PromptView` import dropped; `makeSkill` preserved as barrel anchor.
- `docs/foundations.md` D-20 section rewritten (see amendment note).
- `docs/architecture.md` §1.1 + §2.2 + §11.7 rewritten.
- `docs/phase-1-solo/architecture.md` §2/§4/§5.2/§6/§7.5/§8 + Appendix B updated.
- `docs/phase-1-solo/spec.md` dogfooding + M4 scope + TDD table + signal table updated.
- `docs/dogfooding-protocol.md` §12 M4 deferrals updated; `inspect` + `validate-skill` slip to M5.
- `docs/adr/0003` §Consequences amendment note added.
- `docs/adr/0005` pivot-amendment section added; `related:` points at this ADR not at the never-authored 0006.
- `docs/phase-1-solo/m4-development-proposal.md` supersession banner added at top.

### Near-term (M5)

- M5 ships the deferred M4 Batch E + F + G deliverables: `foreman inspect`, `foreman validate-playbook`, `examples/tdd-feature/`, `examples/refactor-extract/`, `docs/agent-setup.md` (with an "Externalized rendering" section documenting canonical placeholder conventions + per-agent snippets).
- M5 adds `PREFLIGHT_BLOCKED` error code (scope-path allowlist enforcement — pure-string match, no I/O).
- M5 ships the G4 scenario test suite (≥8 scenarios: terminal, composite, block/unblock/fail, recovery, 2-playbook interleave) + G9 (2-playbook target).
- M5 adds `docs/phase-1-solo/architecture-traceability.md` with a self-checking snapshot test, mapping every `foundations.md` invariant + discipline to at least one test path.

### Long-term (Phase 2a+)

- `TokenVendingPort` promotes when Vault/HSM adapter lands at Phase 2a. `packages/core/src/ports/` + `packages/core/src/adapters/` directories materialize at that time alongside the D-9 extraction.
- Any future prompt-assembly feature would require a new ADR that supersedes this one + re-charters P-1 + re-evaluates the harness/agent boundary. This is not expected.

### Invariants

- **Retired:** I-10 (assembler kind-blind), I-11 (ports-only-types-and-zod), I-14 (PromptAssembler purity), I-15 (flat PromptView), I-16 (view-mapping phase-isolation), I-17 (PromptView.vars strings-only). All six were scoped to `PromptAssemblerPort` or its adapter; with the port + adapter gone the invariants have no referent. In particular, I-10 was originally defined (M4 proposal §8, skeleton §Invariants) as "Assembler is kind-blind — adapter signature excludes `brief.kind` / `plan.kind`" — an adapter-level constraint only.
- **Retained with narrower framing:** I-12 (no `TokenVendingPort` before Phase 2a — still applies; renamed from the M4 proposal's original `TokenVendorPort` to match the canonical post-pivot spelling used in ADR 0003 amendment + foundations.md D-20). I-13 (composition root — narrowed from "sole ports↔adapters crosser" to "sole site instantiating `openSqliteCheckpointer` + `buildGraph` + `createHandlers` + `createMcpServer` simultaneously").
- **Handler-level kind branching is explicitly authorized by ADR 0004**, not governed by I-10. `handlers/step.ts` reads `existingValues.kind === 'composite'` (stored `state.kind`) then `plan.kind !== 'composite'` (stored `state.plan.kind`) to enforce the composite parent's work-phase sequence-validity pre-check (`COMPOSITE_CHILD_NOT_READY`). ADR 0004 §"Composite parent work phase = consolidation" + foundations D-21 ADR-0004-Option-B clarification cover this. No new invariant (and no retained I-10) is needed because ADR 0004 is the authorizing artifact.

## Alternatives considered

- **A. Keep `PromptAssemblerPort` behind a feature flag.** Rejected: flags on architectural seams become permanent residue. Clean revert is cheaper to maintain than a gated seam we never expect to enable.
- **B. Keep the adapter as a "convenience helper" in handlers, not a port.** Rejected: any in-process rendering is still foreman interpreting domain content. P-1 argument doesn't hinge on port ceremony.
- **C. Author ADR 0006 retroactively codifying the pre-pivot design, then author a follow-up ADR 0007 reverting it.** Rejected: ADR 0006 would immediately be "superseded before acceptance," which has no informational value. Vacating the slot + writing one ADR describing the decision surface is cleaner.
- **D. Rename `PromptAssemblerPort` to `PromptTemplateRendererPort` and keep the shape.** Rejected: the problem is categorical (rendering = domain content), not naming.
- **E. Defer the pivot decision until M5 and ship M4 as-planned.** Rejected: shipping a port we know is retiring at M5 would strand test coverage + documentation + mental model.

## Evaluation + GAN sign-off

- **Codex GAN (2026-04-19):** audited doc drift across 11 files; ranked surgical-rewrite-on-active + banner-on-historical as correct; confirmed ADR 0006 was never authored; confirmed invariants I-10/I-11/I-14/I-15/I-16/I-17 retire (all assembler-scoped), I-12/I-13 survive narrowed.
- **Codex GAN (2026-04-19, second pass — hexagonal step-back review):** caught an over-broad I-10 restatement in an earlier draft of this ADR ("no `brief.kind` / `plan.kind` branching in any signal handler"). The `handlers/step.ts` composite pre-check branches on `kind`, authorized by ADR 0004. Wording narrowed in this final ADR to retire I-10 alongside the other assembler-scoped invariants and point at ADR 0004 for handler-level kind branching. Verdict MODIFY-FIRST was honored by this amendment.
- **Self-review:** independent read of the same 11 files; converged with Codex on the must-edit-now set (foundations + both architecture docs + spec + dogfooding-protocol + ADRs 0003/0005 + ADR 0007 new + M4 proposal banner).
- **Code-integrity gates post-revert:** 281 vitest tests green; dep-cruiser 0 violations (53 modules, 126 deps); `tsc --noEmit` clean; eslint clean.

Operator sign-off + annotated `m4-alpha` tag anchored to the commit that lands this ADR + doc updates + code revert.
