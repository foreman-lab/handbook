---
status: HISTORICAL — M1 complete (see m1-evaluation.md). Banner added 2026-04-19 as part of M5 docs refactor.
milestone: M1 — Contracts
gate: G2 (schemas side) + G7 (error-codes.md stub)
date: 2026-04-17
methodology: Foreman PGE (Plan → Work → Evaluate) + GAN review (Codex + Copilot)
depends_on: M0 committed (4d7ef04), G1 gate PASS
---

> **⚠ HISTORICAL — milestone complete.**
>
> This proposal is the locked plan for milestone M1 (Contracts), retained as history for git-blame traceability. Evaluation outcome: [`m1-evaluation.md`](m1-evaluation.md). Current milestone: [`m5-development-proposal.md`](m5-development-proposal.md).

# M1 Development Proposal — Contracts (revised)

## 1. Goal

Lock the wire protocol and error taxonomy **before any runtime logic exists.** Every type a handler, node, transport, or CLI will consume must parse/round-trip through a Zod schema by M1 close. Errors thrown in M2+ must carry one of the 7 M1 starter `ErrorCode` values; M2–M5 may add codes during WORK, and M6 sweep locks full catalog per G7.

**Non-goal:** no graph, no handlers, no MCP wiring, no ledger persistence, no CLI commands. M1 ships types + errors + tests only.

## 2. Gate Criteria

| Gate                    | Requirement                                                                   | M1 coverage          |
| ----------------------- | ----------------------------------------------------------------------------- | -------------------- |
| G2 (schemas)            | All 7 signal envelopes + supporting types + 7 state channels have Zod schemas | Ships                |
| G2 (MCP wiring)         | MCP handler dispatch                                                          | Deferred to M2       |
| G7 (error catalog stub) | `docs/error-codes.md` with 7 starter codes from spec §8.3                     | Ships                |
| G7 (full catalog)       | 100% enforcement across thrown errors                                         | Deferred to M6 sweep |

Plus continuing G1: every M0 gate must still be green. M1 cannot regress M0.

## 3. Scope — Files to Create

### 3.1 Types (packages/solo/src/types/)

| #   | File               | Exports                                                                                                                                                                                                                                                                                                 | Notes                                                                                                                                      |
| --- | ------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| 1   | `skill.ts`         | `SkillSchema` (Zod, `.strict()`) + `Skill` TS type                                                                                                                                                                                                                                                      | D-18 signal-embedded shape; no runtime file reads                                                                                          |
| 2   | `finding.ts`       | `FindingSchema` + `Finding`                                                                                                                                                                                                                                                                             | `severity: info\|warning\|critical`                                                                                                        |
| 3   | `error-codes.ts`   | `ErrorCodeSchema = z.enum(ERROR_CODE_VALUES)` with `ERROR_CODE_VALUES` declared `as const`; inferred `ErrorCode` TS type                                                                                                                                                                                | 7 starter codes per spec §8.3; `as const` preserves literal inference; renamed from `errors.ts` to avoid collision with `errors/` dir      |
| 4   | `tool-call.ts`     | `ToolCallSchema` + `ToolCall`                                                                                                                                                                                                                                                                           | `reversibility: z.enum([...]).default("unknown")` per spec S-3                                                                             |
| 5   | `brief.ts`         | `BriefSchema` (`.strict()` at top level per S-4) + `Brief`                                                                                                                                                                                                                                              | includes `projectId: string` (Gate 6 hedge), `skill: Skill`, `kind: "terminal"\|"composite"`                                               |
| 6   | `plan.ts`          | `PlanSchema` + `Plan`                                                                                                                                                                                                                                                                                   | canonical; shared by signal payload AND state channel (spec §8.4, §9)                                                                      |
| 7   | `work.ts`          | `WorkReportSchema` + `WorkReport`                                                                                                                                                                                                                                                                       | canonical; carries `ToolCall[]`                                                                                                            |
| 8   | `evaluation.ts`    | `EvaluationSchema` + `Evaluation`                                                                                                                                                                                                                                                                       | carries `Finding[]`, `meetsCriteria: boolean`                                                                                              |
| 9   | `signal.ts`        | 7 envelope schemas (each `z.object({...}).strict()`) composed via `z.discriminatedUnion("type", [...])` into `SignalSchema` + `Signal`                                                                                                                                                                  | imports Skill/Brief/ToolCall/Plan/Finding/ErrorCode from 1–8; strictness is per-envelope before the union (per D-M1-10)                    |
| 10  | `state.ts`         | 7 channel schemas: `NodeIdSchema`, `KindSchema`, `BriefChannelSchema` (alias of BriefSchema, not re-export), `PlanSchema.nullable()`, `WorkReportSchema.nullable()`, `EvaluationSchema.nullable()`, `JournalSchema = z.array(JournalEventSchema)`                                                       | one file per spec §9; imports Brief/Plan/Work/Evaluation/JournalEvent from 5/6/7/8/11; alias preserves single-import path                  |
| 11  | `journal.ts`       | `JournalEventSchema` + `JournalEvent`                                                                                                                                                                                                                                                                   | UTC `ts` via `z.string().datetime({ offset: false })`; `type: init\|plan\|work\|eval\|block\|resume` (diverges from SignalType — see §3.5) |
| 12  | `wire-response.ts` | **Factory** `wireResponseSchema<S extends z.ZodTypeAny>(payload: S)` returning `z.discriminatedUnion("ok", [success, error])`; type helper `WireResponse<T> = { ok: true; data: T } \| { ok: false; error: { code: ErrorCode; ... } }` inferred via `z.infer<ReturnType<typeof wireResponseSchema<S>>>` | Zod has no native generics — factory pattern; `S extends z.ZodTypeAny` is more forgiving than `z.ZodType<T>`                               |
| 13  | `index.ts`         | barrel re-export                                                                                                                                                                                                                                                                                        | no value cycles (dep-cruiser will verify at M3)                                                                                            |

### 3.2 Errors (packages/solo/src/errors/)

4-class taxonomy is locked per foundations D-5. No `StateError`.

| #   | File                  | Exports                                                                                                                                        | Notes                                                              |
| --- | --------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------ |
| 14  | `base-error.ts`       | `BaseError extends Error` with fields `code: ErrorCode`, `message: string`, `details?: unknown`, method `toJSON(): { code, message, details }` | neutral name per D-19                                              |
| 15  | `validation-error.ts` | `ValidationError extends BaseError`                                                                                                            | for SCHEMA_VALIDATION_FAILED / BRIEF_KIND_MISSING / NODE_NOT_FOUND |
| 16  | `security-error.ts`   | `SecurityError extends BaseError`                                                                                                              | for OPERATOR_APPROVAL_REQUIRED / PREFLIGHT_SECRET_DETECTED         |
| 17  | `storage-error.ts`    | `StorageError extends BaseError`                                                                                                               | for CHECKPOINTER_WRITE_FAILED / JOURNAL_CORRUPTION_DETECTED        |
| 18  | `index.ts`            | barrel                                                                                                                                         | including `BaseError` for `instanceof` narrowing                   |

### 3.3 Docs

| #   | File                  | Content                                                                                                                           |
| --- | --------------------- | --------------------------------------------------------------------------------------------------------------------------------- |
| 19  | `docs/error-codes.md` | 7 starter codes from spec §8.3 verbatim; notes that M2–M5 may introduce additional codes during WORK; M6 sweep locks full catalog |

### 3.4 Package Manifest Update

| #   | Change                                                                                           | Rationale                                                                                                                                                                                                    |
| --- | ------------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 20  | `packages/solo/package.json`: add `"dependencies": { "zod": "3.23.8" }` (exact pin per spec §12) | Production dependency because `types/` and `errors/` are imported by runtime code from M2 onward. Patch bumps within 3.23.x allowed on CVE without re-approval; minor bump to 3.24.x requires M6 audit sweep |

### 3.5 JournalEvent vs Signal — Naming Divergence (intentional)

Per spec: `JournalEvent.type` uses `init|plan|work|eval|block|resume`; `SignalType` uses `initialize|plan|work|eval|resume|token_request|status`. They diverge on purpose:

- `init` (journal) ≠ `initialize` (signal): journal uses short form.
- `block` appears only in journal (emitted when outcome rules → block, not a signal).
- `token_request` / `status` produce no journal entries (pure reads or out-of-band).

M1 tests must assert the divergence, not collapse it.

### 3.6 Tests (packages/solo/tests/unit/)

Per DoD: every Zod schema ≥2 tests (accept + reject). Per spec §5.9: `.strict()` extra-field rejection on top-level; nested permissive. Per spec S-3: `.default("unknown")` on `ToolCall.reversibility`. Diamond pyramid (§5.2) — M1 is 100% unit.

| Module                     | Cases | Notes                                                                                                                                                                                                                                                                                                                                                                                             |
| -------------------------- | ----- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `skill.test.ts`            | 2+    | accept canonical; reject missing/empty fields                                                                                                                                                                                                                                                                                                                                                     |
| `finding.test.ts`          | 2+    | accept; reject unknown severity                                                                                                                                                                                                                                                                                                                                                                   |
| `error-codes.test.ts`      | 3+    | enum round-trip; unknown code rejection; assert `z.enum` values length === 7                                                                                                                                                                                                                                                                                                                      |
| `tool-call.test.ts`        | 3     | accept; reject invalid reversibility; **assert default `"unknown"` applies when omitted**                                                                                                                                                                                                                                                                                                         |
| `brief.test.ts`            | 4+    | accept; reject missing `kind`; reject missing `projectId`; **assert top-level `.strict()` rejects extra fields**                                                                                                                                                                                                                                                                                  |
| `plan.test.ts`             | 2+    | accept; reject malformed                                                                                                                                                                                                                                                                                                                                                                          |
| `work.test.ts`             | 2+    | accept; reject malformed                                                                                                                                                                                                                                                                                                                                                                          |
| `evaluation.test.ts`       | 2+    | accept; reject malformed                                                                                                                                                                                                                                                                                                                                                                          |
| `signal.test.ts`           | 19+   | 7 envelopes × 2 = 14; **first failing test per spec §3.2: `InitializeSignal` missing `kind`** (= 1 of the 14); discriminated union: 2 (correct disc + wrong disc rejection); **per-envelope `.strict()` extra-field rejection: 2 (one on a strict member + one to prove nested ToolCall[] stays permissive)**; union composition: 1 (Zod parses unknown `type` → `SCHEMA_VALIDATION_FAILED` path) |
| `state.test.ts`            | 17+   | 7 channels × 2 = 14; **explicit null-acceptance for PlanSchema.nullable() / WorkReportSchema.nullable() / EvaluationSchema.nullable(): 3**                                                                                                                                                                                                                                                        |
| `journal.test.ts`          | 6     | accept UTC `Z`; reject `+08:00`; reject `+00:00` (must be literal `Z`); reject no-zone; reject empty string; reject non-datetime (`"not-a-date"`); assert `new Date().toISOString()` output is accepted                                                                                                                                                                                           |
| `wire-response.test.ts`    | 6+    | success + error × 2 inner types (unknown + Brief) = 4; **error-branch `error.code` validates against `ErrorCodeSchema`: 1**; **error-branch with invalid code rejects: 1**                                                                                                                                                                                                                        |
| `base-error.test.ts`       | 6     | construction; `instanceof Error`; `toJSON()` shape (happy path); **`toJSON()` with `details = circular ref` does not throw**; **`toJSON()` with `details = BigInt` handles safely (stringify or drop)**; **`toJSON()` with `details = undefined` omits the field**                                                                                                                                |
| `error-subclasses.test.ts` | 3     | each subclass carries the right code                                                                                                                                                                                                                                                                                                                                                              |

**Target:** ≥52 cases minimum (recount after round-2 adjustments); ~70 cases realistic. Coverage ≥95% on `types/` and `errors/`.

## 4. Decisions

| ID      | Decision                                                                                                                                                                                                                                                                                        | Rationale                                                                                                                                                                                                           |
| ------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| D-M1-1  | Zod-first: schemas declare, TS infers via `z.infer<typeof X>`                                                                                                                                                                                                                                   | Single source of truth; runtime validation = type boundary                                                                                                                                                          |
| D-M1-2  | `ErrorCode` is **runtime Zod enum** + inferred TS type (not pure TS union)                                                                                                                                                                                                                      | Runtime validation needed by `WireResponse.error.code` per spec §8.2; D-19 naming respected because `error-codes.ts` module                                                                                         |
| D-M1-3  | Signals: each of the 7 envelopes declares `.strict()` on its `z.object({...})` **before** composition into `z.discriminatedUnion("type", [...])`                                                                                                                                                | Zod 3.x: `.strict()` on a discriminated union is invalid; strictness must be per-envelope so each extra-field rejection maps to `SCHEMA_VALIDATION_FAILED`. Matches spec S-4; exhaustiveness via TS `never` pattern |
| D-M1-4  | Error classes extend `Error`                                                                                                                                                                                                                                                                    | `instanceof Error` stays true for Node stack traces; LangGraph tolerates real Errors                                                                                                                                |
| D-M1-5  | All journal `ts` rejected unless literal `Z` suffix. `z.string().datetime({ offset: false })` rejects both `+HH:MM` offsets AND `+00:00` (only `Z` is accepted); no-zone strings also rejected                                                                                                  | Spec §9; prevents determinism drift (R-12); ensures canonical UTC form for hash-comparison across timezones                                                                                                         |
| D-M1-6  | Barrel `index.ts` in `types/` and `errors/`                                                                                                                                                                                                                                                     | Single import point; dep-cruiser verifies no value cycles                                                                                                                                                           |
| D-M1-7  | Zod validation only at adapter boundaries                                                                                                                                                                                                                                                       | Hexagonal pattern; internal code trusts inferred types                                                                                                                                                              |
| D-M1-8  | Strict TDD for M1                                                                                                                                                                                                                                                                               | Pure data types; negligible cost; catches schema drift                                                                                                                                                              |
| D-M1-9  | `WireResponse<T>` is a factory function `wireResponseSchema<S extends z.ZodTypeAny>(payload: S)` returning `z.discriminatedUnion("ok", [success, error])`; TS helper `WireResponse<T> = { ok: true; data: T } \| { ok: false; error: { code: ErrorCode; message: string; details?: unknown } }` | Zod has no native generics; `S extends z.ZodTypeAny` is more forgiving at call sites than `z.ZodType<T>` which can break inference chains                                                                           |
| D-M1-10 | **Per-envelope** `.strict()` on `SkillSchema`, `BriefSchema`, and each of the 7 signal envelopes (before discriminatedUnion); **nested** `ToolCall[]`/`Finding[]` remain permissive                                                                                                             | Spec S-4: top-level strictness must be per-object, not per-union (Zod 3.x constraint); nested permissive for forward-compat metadata                                                                                |
| D-M1-11 | `ToolCall.reversibility` uses `z.enum([...]).default("unknown")`                                                                                                                                                                                                                                | Spec S-3: missing → `"unknown"` → treated as irreversible; do NOT reject                                                                                                                                            |
| D-M1-12 | Shared canonical schemas (Plan, WorkReport, Evaluation) live in own files, imported by both `signal.ts` and `state.ts`                                                                                                                                                                          | Prevents duplicate schema drift between signal payloads and checkpoint channels                                                                                                                                     |

## 5. Task Order (WORK state, TDD-interleaved)

Each batch writes the failing test first, then the schema, then verifies.

**Batch A — Primitives (parallel, no deps):**

1. `skill.test.ts` → `skill.ts`
2. `finding.test.ts` → `finding.ts`
3. `error-codes.test.ts` → `error-codes.ts`
4. `journal.test.ts` → `journal.ts` (standalone — `type` enum + `ts` datetime; no dep on A#1-3)

**Batch B — Composed primitives (depend on A):**

5. `tool-call.test.ts` → `tool-call.ts`
6. `brief.test.ts` → `brief.ts` (depends on skill from A#1)

**Batch C — Canonical shared schemas (depend on A+B):**

7. `plan.test.ts` → `plan.ts`
8. `work.test.ts` → `work.ts` (depends on tool-call from B#5)
9. `evaluation.test.ts` → `evaluation.ts` (depends on finding from A#2)

**Batch D — Protocol (depend on A+B+C):**

10. `signal.test.ts` (first failing test per spec §3.2: InitializeSignal missing `kind`) → `signal.ts`
11. `state.test.ts` → `state.ts`
12. `wire-response.test.ts` → `wire-response.ts`
13. `types/index.ts` barrel

**Batch E — Error classes (parallel with C/D; starts as soon as A#3 `error-codes.ts` lands):**

14. `base-error.test.ts` → `base-error.ts`
15. `error-subclasses.test.ts` → `validation-error.ts` + `security-error.ts` + `storage-error.ts`
16. `errors/index.ts` barrel

**Batch F — Docs + manifest:**

17. `docs/error-codes.md`
18. `packages/solo/package.json`: add `dependencies.zod = "3.23.8"`; run `pnpm install` to refresh lockfile

**Batch G — Verify + commit:**

- `pnpm install --frozen-lockfile && pnpm typecheck && pnpm lint && pnpm test && pnpm format:check && pnpm boundary-check && pnpm licenses:check` — all green
- ≥52 test cases, 0 skipped
- Commit M1

## 6. Out of Scope for M1

- LangGraph flow, nodes, composition root (M2)
- MCP server, CLI commands (M2+)
- Ledger / atomic commit / journal writer (M2+)
- **Skill runtime / prompt assembly (M4 close)** — per spec §2.5, §3.3; M3 uses raw signals
- Preflight secret scan (M2)
- Mutex / TTL (M3)
- Error code additions from M2-M5 work (deferred to M6 sweep)

## 7. Risks

| ID      | Risk                                                                                                               | Likelihood                     | Mitigation                                                                                                                                                                                                                    |
| ------- | ------------------------------------------------------------------------------------------------------------------ | ------------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| R-M1-1  | Zod 3 → 4 migration mid-phase                                                                                      | Low                            | Exact-pin version per spec §12; M6 deps sweep handles upgrade with audit                                                                                                                                                      |
| R-M1-2  | `ErrorCode` enum additions during M2-M5 are **certain** (per spec §8.3); **disruption** to callers could be medium | Medium (impact, not frequency) | Adapter-boundary narrowing (D-M1-7) isolates callers; M6 sweep locks final catalog; during M2-M5 any new code requires an ADR-lite entry in `docs/error-codes.md`                                                             |
| R-M1-3  | Signal discriminated union becomes unwieldy                                                                        | Low                            | 7 signals max per spec; manageable                                                                                                                                                                                            |
| R-M1-4  | Scope creep into M2 graph reducers                                                                                 | Medium                         | Hard stop: state.ts has channel SCHEMAS only, no reducer logic                                                                                                                                                                |
| R-M1-5  | TDD discipline slips under time pressure                                                                           | Low                            | Spec §5.1 + DoD: no test = no merge                                                                                                                                                                                           |
| R-M1-6  | `.strict()` breaks forward-compat at a wrong boundary                                                              | Medium                         | Spec S-4 explicit: top-level strict, nested permissive (D-M1-10)                                                                                                                                                              |
| R-M1-7  | **Zod cold-start cost** exceeds MCP handshake budget (Q-11a)                                                       | Low                            | Arch §8.1 includes Zod compile in budget; benchmark at M2 first-handshake scenario                                                                                                                                            |
| R-M1-8  | **Schema / doc divergence** (schemas are source of truth per spec §8)                                              | Medium                         | CI check at M6 — doc-sync job verifies docs match z.infer output                                                                                                                                                              |
| R-M1-9  | **Barrel imports create value cycles** dep-cruiser misses at M1                                                    | Low                            | M3 adds no-deep-imports rule; manual review of `types/index.ts` + `errors/index.ts` at M1 close                                                                                                                               |
| R-M1-10 | `WireResponse<T>` factory usage inconsistency across call sites                                                    | Medium                         | M3 task (tracked in `docs/phase-1-solo/m3-development-proposal.md` when drafted): add dep-cruiser `no-deep-imports` rule forcing imports via `@foreman-lab/solo/types` barrel only, not deep paths like `types/wire-response` |

## 8. GAN Review Log

### Round 1 — Codex (verdict: REVISE) + Copilot (10 findings)

**Convergent MUST-FIX (both reviewers):**

1. `state.ts` underspecified — now enumerates 7 channels with canonical shared schemas (Plan/WorkReport/Evaluation in own files)
2. `ErrorCode` runtime validation — switched to `z.enum` + `z.infer`; renamed module to `error-codes.ts`
3. `WireResponse<T>` factory function (D-M1-9)
4. Added `.strict()` policy per S-4 (D-M1-10)
5. Test count corrected — ≥44 minimum, ~60 realistic

**Codex-only MUST-FIX:** 6. Dropped "frozen for M2+" → "starter set; M2-M5 add codes; M6 sweep locks" 7. Removed any `StateError` suggestion — D-5 taxonomy locked 8. `ToolCall.reversibility` → `.default("unknown")` per S-3 (D-M1-11) — this was a spec misread 9. "Skill runtime (M3+)" → "Skill runtime/prompt assembly (M4 close)" per §2.5 10. Task order reshuffled so Plan/WorkReport/Evaluation ship in Batch C before signal + state

**Copilot-only MUST-FIX:** 11. TDD batch ordering — tests now interleaved per batch, not a single final Batch F 12. `BaseError.toJSON()` added to spec 13. Explicit task #20 for `zod` as prod dependency in `packages/solo/package.json` 14. §3.5 documents JournalEvent vs SignalType naming divergence

**Nice-to-haves adopted:**

- Risks R-M1-7 (Zod cold-start), R-M1-8 (schema/doc sync), R-M1-9 (barrel cycles), R-M1-10 (factory inconsistency)
- Dropped "first prod dep" phrasing (not grounded per spec)
- Don't pin "v3.x" — exact `3.23.8` pin

**Rejected:** none; Codex REVISE verdict fully addressed.

### Round 1 — Claude self-review

- Batch C introduction (shared Plan/Work/Evaluation) eliminates schema-drift risk between signal payloads and state channels — Codex's #9 was load-bearing.
- `error-codes.ts` module name resolves Copilot's #10 naming collision cleanly.
- Test count recalibration: ≥44 vs my original 27 matches the DoD literally.

### Round 2 — Codex (verdict: REVISE, 1 must-fix) + Copilot (11 findings, no blockers)

**Round 2 fixes applied:**

1. **Codex#1 (Zod strictness bug):** `.strict()` cannot sit on a `z.discriminatedUnion(...)` in Zod 3.x. Moved to per-envelope — §3.1 row 9 and D-M1-3/D-M1-10 rewritten so each of the 7 `z.object({...})` envelopes declares `.strict()` **before** union composition. S-4 extra-field rejection now actually fires at runtime.
2. **Codex nice-to-have 1:** `state.ts` uses `BriefChannelSchema = BriefSchema` alias (not re-export) to preserve single-import path.
3. **Codex nice-to-have 2:** R-M1-10 upgraded from prose to concrete M3 task (dep-cruiser `no-deep-imports` rule forcing barrel imports).
4. **Codex nice-to-have 3:** `ErrorCodeSchema` values declared `as const` for literal inference (§3.1 row 3).
5. **Codex nice-to-have 4:** `wireResponseSchema<S extends z.ZodTypeAny>` signature preferred over `z.ZodType<T>` (§3.1 row 12 + D-M1-9).
6. **Copilot#1 imports:** §3.1 row 9 now reads "imports Skill/Brief/ToolCall/Plan/Finding/ErrorCode from 1–8" (was "5–8"); row 10 now reads "imports Brief/Plan/Work/Evaluation/JournalEvent from 5/6/7/8/11".
7. **Copilot#2 signal.test.ts arithmetic:** 14 + 2 + 2 + 1 = **19** (was 18+).
8. **Copilot#3 BaseError.toJSON safety:** added 3 tests (circular ref, BigInt, undefined) — §3.6 `base-error.test.ts` = 6 cases.
9. **Copilot#4 WireResponse error.code:** added 2 tests (valid code accepted, invalid code rejected) — §3.6 `wire-response.test.ts` = 6+ cases.
10. **Copilot#5 journal edge cases:** added empty-string and non-datetime rejects; added `+00:00` rejection (only `Z` accepted) — §3.6 `journal.test.ts` = 6 cases.
11. **Copilot#6 state null-acceptance:** added 3 explicit nullable tests — §3.6 `state.test.ts` = 17+ cases.
12. **Copilot#7 journal → Batch A:** moved since it has no dep on A#1-3.
13. **Copilot#8 Batch E parallelism:** clarified "parallel with C/D; starts as soon as A#3 lands."
14. **Copilot#9 datetime clarification:** D-M1-5 now says "`+00:00` also rejected; only literal `Z` accepted."
15. **Copilot#10 Zod pin policy:** patch bump within 3.23.x allowed on CVE without re-approval; minor bump (3.24.x) requires M6 audit — §3.4 note.
16. **Copilot#11 R-M1-2 wording:** additions are certain; disruption is medium (semantics fixed).

**Total test count:** ≥52 cases minimum (recount), ~70 realistic.

**Rejected:** none.

### Round 2 — Claude self-review

- Codex's strictness fix is the single load-bearing correction of round 2; other items are rigor improvements.
- Copilot's test-gap catches (toJSON safety, error.code validation, null channels) prevent real M2+ bugs.
- Batch reordering (journal → A) unlocks small amount of additional parallelism; not critical for solo dev but correct dependency modeling.

## 9. Evaluation Protocol (post-WORK)

1. `pnpm install --frozen-lockfile && pnpm typecheck && pnpm lint && pnpm test && pnpm format:check && pnpm boundary-check && pnpm licenses:check` — all green.
2. Inspect test output: ≥52 assertions ran, 0 skipped.
3. Coverage report — ≥95% on `packages/solo/src/types/` and `packages/solo/src/errors/`.
4. `docs/error-codes.md` renders cleanly in GitHub preview.
5. Dispatch Codex + Copilot round 3 (post-WORK review of shipped artifacts vs this proposal).
6. If all clean → tag `m1-complete` after user approval.

## 10. Estimated Duration

Per spec: ~1 week. Lower bound ~3 days; upper bound ~7 days if round-2 GAN surfaces additional spec gaps (less likely now given round-1 depth).
