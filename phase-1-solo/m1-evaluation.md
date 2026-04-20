---
status: M1 Complete — G2 (schemas side) + G7 (stub) PASS (local)
milestone: M1 — Contracts
gate: G2 (schemas) + G7 (error-codes.md stub)
date: 2026-04-18
methodology: Foreman PGE — Evaluate state
---

# M1 Evaluation — Contracts

## Result: G2 schemas + G7 stub PASS (local verify)

CI verification pending push; local verify on Node 24.14.0 + pnpm 9.15.0 green across all 7 G1-inherited gates, plus M1-specific schema coverage.

## Gate-by-Gate

| #   | Gate                             | Result  | Notes                                                                                                           |
| --- | -------------------------------- | ------- | --------------------------------------------------------------------------------------------------------------- |
| 1   | `pnpm install --frozen-lockfile` | ✅ PASS | zod 3.23.8 added as first prod dep                                                                              |
| 2   | `pnpm typecheck`                 | ✅ PASS | all inferred types compile                                                                                      |
| 3   | `pnpm lint`                      | ✅ PASS | 0 issues across 26 files                                                                                        |
| 4   | `pnpm test`                      | ✅ PASS | **106 tests across 14 files** (target was ≥52; 104% over). Coverage **98.97%** (barrels excluded; target ≥95%). |
| 5   | `pnpm format:check`              | ✅ PASS | prettier 3.4.2                                                                                                  |
| 6   | `pnpm boundary-check`            | ✅ PASS | 37 modules, 81 deps cruised, 0 violations                                                                       |
| 7   | `pnpm licenses:check`            | ✅ PASS | zod@3.23.8 MIT — script fixed to use `pnpm licenses list`                                                       |
| —   | G2 schemas side                  | ✅ PASS | 7 signal envelopes + 7 state channels + WireResponse factory                                                    |
| —   | G7 catalog stub                  | ✅ PASS | `docs/error-codes.md` ships 7 starter codes                                                                     |

## Artifacts Shipped (37 files)

**types/ (13):** skill, finding, error-codes, tool-call, brief, plan, work, evaluation, signal, state, journal, wire-response, index (barrel)

**errors/ (5):** base-error, validation-error, security-error, storage-error, index (barrel)

**tests/unit/ (14):** skill, finding, error-codes, tool-call, brief, plan, work, evaluation, signal, state, journal, wire-response, base-error, error-subclasses

**Package root (1):** `packages/solo/src/index.ts` — top-level barrel re-exporting types + errors

**Docs (2):** `docs/error-codes.md` + `docs/phase-1-solo/m1-evaluation.md` (this)

**Proposal (1):** `docs/phase-1-solo/m1-development-proposal.md` (already committed pre-WORK)

**Manifest (1):** `packages/solo/package.json` — zod 3.23.8 as prod dep

**Config (1):** `packages/solo/vitest.config.ts` — per-package test scope + barrel-excluded coverage

**Script fix (1):** `scripts/check-licenses.mjs` — switched from `pnpm list` to `pnpm licenses list`

## GAN Round 4 (post-WORK) — Fixes Applied

Dispatched after Batches A-F complete + mid-WORK fixes landed.

### Codex — REVISE (2 new must-fix)

14. **`JournalEventSchema` missing spec §9 required fields:** `nodeId: z.string()` + `actor: z.enum(['agent','operator','harness'])`. **Fixed** + 5 new test cases (actors × 3, missing nodeId, missing actor).
15. **Evaluation artifact count arithmetic error:** header claimed 28, enumeration summed higher. **Fixed** to 37.

### Copilot — 13 findings (1 critical)

16. **[CRITICAL] `packages/solo/src/index.ts` still `export {};`** from M0 — package un-importable by consumers. **Fixed** with `export * from './types/index.js'; export * from './errors/index.js';`.
17. **`WorkReportSchema` vs `WorkPayloadSchema` strict-divergence:** Unified — `WorkPayloadSchema = WorkReportSchema` (canonical is now strict).
18. **`EvaluationSchema` vs `EvalPayloadSchema` same issue:** Unified identically.
19. **`StatusPayload` type not exported:** Added to signal.ts type exports block.
20. **`safeSerialize` missing Date/Map/Set/RegExp/Error special cases:** Added with readable surrogate forms.
21. **`safeSerialize` DAG-as-circular bug:** Switched `WeakSet` to `Set` scoped to DFS path; `add` on descent, `delete` on ascent. Non-circular repeated refs no longer flagged.
22. **`FindingSchema.detail` allows empty string:** Added `.min(1)` + rejection test.
23. **Base-error test gaps:** Added tests for Date/RegExp/Map/Set/function/symbol/nested-undefined + DAG repeat-ref case.
24. **`docs/error-codes.md` inconsistencies:** `locked` frontmatter → `baselined`; added M2+ enforcement note on PREFLIGHT / OPERATOR / CHECKPOINTER codes; added D-5 taxonomy cross-reference.

### Claude self-reviews

25. **Coverage measurement added:** Excluded barrels from v8 coverage (non-executable re-exports); new coverage = **98.97%** statements, 90.9% branches, 100% functions. Above proposal target ≥95%.
26. **Final test count:** 106 tests across 14 files.

## GAN Round 3 (mid-WORK) — Fixes Applied

Dispatched Codex + Copilot after Batches A-D (types/) complete, before errors/.

### Codex — REVISE (2 spec mismatches, both caught pre-commit)

1. **Signal envelope:** shape was flattened; spec §8.1 specifies `{ type, nodeId, payload }` with per-type discriminated payload. **Fixed:** introduced per-type payload schemas; each envelope is now `{ type, nodeId, payload: PerTypePayload }`. 23 signal tests rewritten.
2. **Skill shape:** invented fields (`version`, `objectives`, etc.); spec §10 specifies `{ name, methodology: { plan, work, evaluate }, scope?: { readOnly, paths } }`. **Fixed** with canonical tdd-feature fixture.

### Copilot — 5 findings

3. **`brief.ts`:** `goal` lacked `.min(1)`. Fixed + test added.
4. **`plan.ts`:** `TerminalPlanSchema` and `CompositePlanSchema` lacked `.strict()`. Fixed + extra-field rejection test.
5. **`plan.ts`:** `CompositePlanSchema.children` allowed empty array. Fixed with `.min(1)` + rejection test.
6. **Signal resume:** `fail` action untested. Added accept + reject-empty-reason tests.
7. **Signal token_request:** fractional `ttl` untested. Added rejection test.

### Claude nice-to-have

8. **Wire-response type inference:** added compile-time assertion test (`r.data` narrows to `string` after `r.ok` check).

## Adjustments from Proposal

- **`vitest.config.ts` per-package:** root-level config pattern didn't match when turbo runs vitest from `packages/solo` CWD. Added local config; no semantic change.
- **`scripts/check-licenses.mjs` rewrite:** `pnpm list --prod --json` doesn't expose license fields (reports "UNKNOWN"). Switched to `pnpm licenses list --prod --json` which groups by SPDX license.

## Decisions Validated

All 12 decisions (D-M1-1..12) held up in WORK. Key validations:

- **D-M1-1 (Zod-first):** all types inferred via `z.infer`; no hand-written TS types for schemas.
- **D-M1-2 (ErrorCode as `z.enum`):** runtime validation works; `wire-response.test.ts` rejects invalid codes at adapter boundary.
- **D-M1-3 / D-M1-10 (per-envelope `.strict()`):** signal tests confirm extra-field rejection on strict envelope, nested permissive on ToolCall args.
- **D-M1-5 (UTC `Z` only):** journal rejects `+08:00`, `+00:00`, no-zone, empty string, non-datetime.
- **D-M1-9 (wireResponseSchema factory `S extends z.ZodTypeAny`):** inference works with both `z.string()` and `BriefSchema` payloads.
- **D-M1-11 (ToolCall.reversibility defaults to "unknown"):** test asserts default applied.
- **D-M1-12 (shared Plan/Work/Evaluation schemas):** imported by both signal.ts AND state.ts — zero drift.

## Open Risks Carried Into M2

1. **CI verification pending.** Local green ≠ CI green. First push tells.
2. **Schema/doc sync (R-M1-8):** `docs/error-codes.md` still manual; CI doc-sync check deferred to M6.

## Transition to M2

**Next milestone:** M2 — Walking slice (spec §3.3, ~1.5-2 weeks).

**Preconditions met:**

- All 7 signal envelopes locked
- 7 state channel schemas locked
- 4-class error taxonomy locked
- WireResponse factory ready for transport/handler boundary
- dep-cruiser ready to enforce `handlers/ → graph/` boundary when M2 lands those dirs

**Recommendation:** Commit M1, run final GAN round 4 on shipped code, push to confirm CI green, then open M2 proposal.
