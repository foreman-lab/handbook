---
status: M4 Complete (pivot outcome) — landed scope narrower than proposed; domain-blind architecture locked via ADR 0007
milestone: M4 — Domain-blind pivot (originally planned as "Skills + First Port"; pivoted 2026-04-18)
gate: G2 maintained; G6/G9 partial as-proposed and mostly deferred; G7 unchanged
date: 2026-04-19
methodology: Foreman PGE — Evaluate state (logic-correctness GAN + doc-drift GAN + architectural review)
---

# M4 Evaluation — Domain-Blind Pivot

## Result: PASS (pivot) — landed scope honest, post-pivot baseline verified green

M4 as proposed was ~45-65h of port + adapter + skill-driven-rendering work across 12 batches. Mid-execution (2026-04-18) the operator reality-checked the architectural premise: *"foreman is infrastructure; foreman is more state machine; domain knowledge stays independent."* Prompt rendering was relocated to the agent layer, and the `PromptAssemblerPort` / `DefaultPromptAssembler` / `PromptView` / `view-mapping` / `nextPrompt` surface was reverted. ADR 0007 codifies the decision.

Net M4 landed scope is **narrower than proposed** but **load-bearing**: the pivot itself is the milestone artifact, backed by a green code baseline + doc set + new contract test.

## Gate-by-gate

| #   | Gate                                 | Result  | Notes                                                                                                   |
| --- | ------------------------------------ | ------- | ------------------------------------------------------------------------------------------------------- |
| 1   | `pnpm install --frozen-lockfile`     | ✅ PASS | no dep changes                                                                                          |
| 2   | `pnpm typecheck`                     | ✅ PASS | clean post-revert                                                                                       |
| 3   | `pnpm lint`                          | ✅ PASS | clean                                                                                                   |
| 4   | `pnpm test`                          | ✅ PASS | **284 tests across 33 files** (M3 baseline 270 + M4 Scout/spike + new brief-exposure contract test 3)   |
| 5   | `pnpm format:check`                  | ✅ PASS |                                                                                                          |
| 6   | `pnpm boundary-check`                | ✅ PASS | 53 modules, 126 deps, 0 violations                                                                      |
| —   | G2 (2-tool MCP surface)              | ✅ MAINTAINED | `foreman__status` + `foreman__step` unchanged                                                     |
| —   | G6 (S-1..S-5 materialization)        | ⏸ DEFERRED    | `TokenVendingPort` + Batch C1 unstub moved to Phase 2a per ADR 0007                               |
| —   | G7 (error catalog audit)             | ⏸ DEFERRED    | full audit remains M6; net +2 codes at M4 (was +3 pre-pivot; `PROMPT_ASSEMBLY_FAILED` reverted)   |
| —   | G9 (agent-setup + example skills)    | ⏸ DEFERRED    | slipped to M5 — `examples/tdd-feature/` + `examples/refactor-extract/` + `agent-setup.md`         |

## Landed vs proposed scope (honest)

| M4 Batch | Proposed | Actual |
|---|---|---|
| A1 scaffold (port + Zod + channels + `NextPromptSchema`) | ship | **shipped then reverted** (commit 96d0256 → revert) |
| A2 hexagonal dep-cruiser rules + threading | ship | **shipped then reverted** (commit 945221a → revert) |
| A2 Scout — `--foreman-dir` propagation fix | ship | ✅ **LANDED** (commit 8b5db94) |
| A3 spike (replaceReducer + Annotation inputSignal) | decide | ✅ **LANDED** (commit 8a24fb5, won't-fix for M4) |
| Renames BriefKind → NodeKind → TaskKind | (emerged) | ✅ **LANDED** (commits 3aeca1d, 8b948e0) |
| B `DefaultPromptAssembler` adapter | ship | **shipped then reverted** (commit 487eab8 → revert) |
| C1 `token_request` unstub | ship | **dropped** pre-revert to Phase 2a (commit 885f718; ADR 0003 + ADR 0007) |
| C2 composition-root wiring | ship | **shipped then reverted** (commit 7e8a523 → revert) |
| D1/D2 plan/work/evaluate node call-sites | ship | never started (dependent on reverted port) |
| E `foreman inspect` CLI | ship | deferred to M5 |
| F `foreman validate-skill` CLI | ship | deferred to M5 |
| G `examples/tdd-feature/` + `agent-setup.md` | ship | deferred to M5 |
| H ADR 0005 + ADR 0006 + dogfooding update | ship | ADR 0005 ✅ **Accepted + amended**; ADR 0006 **never authored** (slot vacated); ADR 0007 **new — codifies pivot**; dogfooding-protocol updated |
| I post-WORK GAN + `m4-alpha` tag | ship | GAN rounds completed; tag pending operator sign-off |

## Invariants — actual outcome

| Invariant                            | Proposed (M4) | Actual (post-pivot)                                                           |
| ------------------------------------ | ------------- | ------------------------------------------------------------------------------ |
| I-10 "assembler kind-blind"          | introduce     | **retired** — adapter-scoped, no referent post-pivot (ADR 0007)                |
| I-11 "ports only types + zod"        | introduce     | **retired** — no ports/ dir                                                    |
| I-12 "no TokenVendingPort pre-P2a"   | introduce     | **retained** (renamed from original `TokenVendorPort`; Phase 2a trigger holds) |
| I-13 "composition root isolation"    | introduce     | **retained narrowed** — now "sole site instantiating openSqliteCheckpointer + buildGraph + createHandlers + createMcpServer" |
| I-14 "PromptAssembler purity"        | introduce     | **retired** — no assembler                                                     |
| I-15 "flat PromptView"               | introduce     | **retired** — no PromptView                                                    |
| I-16 "view-mapping phase-isolation"  | introduce     | **retired** — no view-mapping                                                  |
| I-17 "PromptView.vars strings-only"  | introduce     | **retired** — no PromptView                                                    |

Handler-level `kind === 'composite'` branching in `handlers/step.ts` is explicitly authorized by ADR 0004, not by an invariant.

## New artifacts

- **`docs/adr/0007-domain-blind-core.md`** — locks the pivot decision; supersedes the never-authored ADR 0006 slot; amends ADR 0003 §Consequences + ADR 0005 (0005 itself unchanged on format; amended-with-note) + M4 proposal supersession banner.
- **`docs/adr/0005-skill-file-format.md`** — Accepted simultaneously with M4 proposal vN, then amended 2026-04-19 with post-pivot note.
- **`packages/solo/tests/contract/brief-exposure.test.ts`** — 3-test contract suite codifying the post-pivot agent-readable-methodology claim (closes Codex logic-GAN MED findings 6 + 7; replaces the 333 lines of coverage deleted with `PromptAssembler*` tests).

## GAN rounds (evaluation phase)

5 GAN passes conducted with Codex (independent review) + self:

1. **Sprint-8 proposal review** (agent `add7d74d1f95e47dd`) — found M4 doc scope stale; directed to MODIFY for rewrite under M-naming.
2. **Hexagonal step-back** (agent `a73e9de1c68add236`) — flagged I-10 over-broad restatement in ADR 0007 ("no kind branching in any signal handler" contradicted `handlers/step.ts` composite pre-check). Verdict MODIFY-FIRST.
3. **Doc-drift close-the-loop** (agent `a8df2de8d22014d95`) — after path-A narrow fix, caught 3 more drift items: ADR 0007 L129 `signal.payload.kind` → `plan.kind`; ADR 0007 L128 `TokenVendorPort` → `TokenVendingPort`; `handlers/step.ts:306-313` + `tests/contract/transport-handler.test.ts:401-418` "M4" → "Phase 2a" wording. Verdict MODIFY-AGAIN.
4. **Logic-correctness** (agent `a322761995fcc2647`) — after operator push-back (*"are you just keep changing document, instead of check whether the logic is correct?"*), switched GAN framing from doc-drift to behavior correctness. Found 2 MED: the deleted `PromptAssembler` tests left the agent-readable-methodology contract untested. Verdict MODIFY-AGAIN with narrow fix.
5. **Brief-exposure close-the-loop** (agents `abd379492b4a6315a` + `a69945970646fa37c`) — phase-survival test in first draft didn't assert plan transition; next draft does. Final pass analytical gates 1-3 all PASS. Local targeted `npx vitest run tests/contract/brief-exposure.test.ts` → 3/3 green.

## Lessons (for M5 + future milestones)

- **GAN framing is load-bearing.** 3 doc-drift rounds got 3 doc-drift answers. 1 logic-correctness round caught the real gap. At least one logic GAN per commit going forward.
- **Contract tests deleted = claims lost coverage.** Before committing a revert that deletes tests, enumerate which claims lose coverage and add replacement tests for the NEW contract.
- **"Tests pass" ≠ "behavior is proven".** 281/281 pre-fix included zero tests for the post-pivot `brief.skill.methodology` exposure. Silent-pass risk is the failure mode.
- **Reality-check mid-execution is not waste.** The ~10h of Batches A1/A2/B/C2 work reverted in this pivot was cheaper than shipping a port we'd retire at M5.

## Artifacts shipped (this commit)

**Code (14 files):** 7 staged deletions (ports/, adapters/, prompt-view.ts, 2 test files) + 7 modifications (app.ts, graph/build.ts, graph/channels.ts, 3 graph/nodes/*.ts, handlers/step.ts, types/index.ts, types/error-codes.ts).

**New:** `packages/solo/tests/contract/brief-exposure.test.ts` (3 tests, ~230 lines).

**Docs (11 modified + 2 new):** foundations.md, architecture.md, phase-1-solo/architecture.md, phase-1-solo/spec.md, dogfooding-protocol.md, adr/0003, adr/0005, phase-1-solo/m4-development-proposal.md (supersession banner), `.dependency-cruiser.cjs`, plus the 2 code test files above (error-codes.test.ts + fixtures/skills/index.ts + composition-root.test.ts + transport-handler.test.ts); **new** `docs/adr/0007-domain-blind-core.md` + `docs/phase-1-solo/m4-evaluation.md` (this file). Untracked: `docs/phase-1-solo/m5-development-proposal.draft.md` (v0.1 workspace draft, not staged for commit).

**Out of scope this commit:** `copilot-plugin/` (untracked workspace plugin, unrelated to M4 pivot).

## Next: `m4-alpha` tag

Operator sign-off on this evaluation + the commit it describes → annotated tag `m4-alpha` anchored to the same SHA. M5 (domain-blind ship) Batch A starts after.
