---
status: HISTORICAL — M3 complete (Batches A-G shipped; no standalone m3-evaluation.md — closed via M4 start). Banner added 2026-04-19 as part of M5 docs refactor.
milestone: M3 — Solo Alpha (complete signal surface + mutex + interrupt/resume + retry + composite eval + dogfooding kickoff)
gate: G3 closed (complete 4-node flow via 2-tool MCP); partial G5 + G7
date: 2026-04-18
methodology: Foreman PGE (Plan → Work → Evaluate) + GAN review (Codex + Copilot)
depends_on: M2 committed (b8a2c88); ADR 0003 approved v2 (3fe9c0d); ADR 0004 drafted (6f2aa48); C3 spike PASS (0d813bd); composite/Send spike (6f2aa48)
amended_by: (this file — v0.2..v0.6 planning merges; v0.7 LangGraph-docs + Option B + post-WORK amendments)
supersedes: v0.1..v0.6; m3-development-proposal-skeleton.md retired
commits: A (286bf03) → A-f (54d115e) → B (a8dcb29) → B-f (3222970) → C (4dedc86) → C-f (ab403c0) → D-progress (1328199) → D (7d5e211) → v0.7 docs (6c1b799) → E.1 (20e4baa) → E.3 (20c836c) → F (114da19) → G.1 (85c0356) → G.2 (b2b3dce) → G.3 (2d2c208)
---

> **⚠ HISTORICAL — milestone complete.**
>
> This proposal is the locked plan for milestone M3 (Solo Alpha), retained as history for git-blame traceability. No standalone `m3-evaluation.md` exists — M3 closed when M4 started. Current milestone: [`m5-development-proposal.md`](m5-development-proposal.md).

# M3 Development Proposal — Solo Alpha (v0.7 — A-G shipped)

## 1. Goal

M3 completes Solo Alpha: `foreman__step` handles all seven writes (initialize / plan / work / eval / resume / retry / token_request); `foreman__status` stays read-only; mutex, interrupt/resume (block state), retry reset (capped state), composite evaluation branch, and `foreman step --signal <json>` CLI subcommand all land. Raw-signal dogfooding starts at M3 close per spec §2.5/§15, under ADR 0002/0003/0004 and foundations D-17/D-18/D-21.

**Non-goal:** ports/adapters (M4), full preflight allowlist (M5), scenario test suite G4 close (M5), performance benchmarks (M6), skill-based orchestration (M4+), **TTL/stale detection (deferred to Phase 2a; v0.3 YAGNI)**.

## 2. Gates

| G-gate | Closes fully? | Closes partially? | Notes                                                                                            |
| ------ | ------------- | ----------------- | ------------------------------------------------------------------------------------------------ |
| G3     | Yes           | No                | Spec §4: complete 4-node flow; roadmap Gate 3 references spec.md `[?]`                           |
| G4     | No            | No                | Spec G4: 7 scenarios at M5                                                                       |
| G5     | No            | Partial           | Spec G5: ≥80% branch coverage on `graph/` + `handlers/` — landed earlier at M2 threshold         |
| G6     | No            | No                | Spec G6: S-1..S-5 integration at M5                                                              |
| G7     | No            | Partial           | Net +0 new error codes at M3 (one added, one M2 code removed — see §3.10); full catalog audit M6 |

Test-count target: M2 ships 190 passing; M3 adds ~80 new (handlers + mutex + contract tests + regression migration), totaling ~270. Round-9 correction from v0.1's ~80 was for the fuller contract test scope; v0.3 reduces it by deferring `handler-graph.test.ts` to M5 (keep `transport-handler.test.ts` only).

## 3. Scope

### 3.1 Dependency changes

- **Default:** no new npm dependency; implement mutex as in-handler `Map<threadId, Promise>` queue with guaranteed `try/finally` cleanup.
- **Contingency:** exact-pin `async-mutex` only if the Batch E spike rejects the map. No `p-queue`.

### 3.2 File enumeration

**New files (M3):**

- `packages/solo/src/handlers/step.ts` — polymorphic dispatch handler (authored in Batch C)
- `packages/solo/src/handlers/mutex.ts` — async lock map + try/finally cleanup (authored in Batch B)
- `packages/solo/src/cli/step.ts` — CLI step subcommand (authored in Batch F)
- `docs/dogfooding-protocol.md` — written at **Batch G**; operator walkthrough (mechanism lives in ADRs; protocol lives here)
- `tests/contract/transport-handler.test.ts` — ≥30 boundary cases, authored in **Batch C** alongside `step.ts` (handler-graph.test.ts deferred to M5)

**Updated files (M3):**

- `packages/solo/src/types/signal.ts` — split into `StepSignalSchema` (7 writes) + `StatusSignalSchema` (1 read); drop top-level `SignalSchema` export; **consolidate `kind` field** (drop `payload.kind` from `InitializePayloadSchema`; `brief.kind` is single source of truth — v0.3 M2 retrospective cleanup)
- `packages/solo/src/types/journal.ts` — extend `JournalEventTypeSchema` with `retry-reset`; narrow `data` shape for retry-reset events to `{ nodeId: string }` (no `reason` field — v0.3 YAGNI; actor + ts stay on the envelope)
- `packages/solo/src/types/error-codes.ts` — **remove `BRIEF_KIND_MISSING`** (never-thrown starter code — v0.3 M2 retrospective cleanup); **add `COMPOSITE_CHILD_NOT_READY`**; net ±0 error codes
- `packages/solo/src/types/index.ts` — barrel update
- `packages/solo/src/handlers/{index,status}.ts` — remove initialize/plan re-exports; status stays (lazy TTL logic inlined if TTL revived later; not present at M3)
- `packages/solo/src/graph/nodes/evaluate.ts` — **kind-agnostic** (Option B, v0.7). Interrupt wiring (`interrupt()` on block outcomes) + retry-count derivation. Composite aggregation is NOT here — it lives in `handlers/step.ts` work pre-check.
- `packages/solo/src/handlers/step.ts` (Batch D extension) — composite work pre-check (Option B, v0.7): when `state.kind === 'composite'`, enumerate `plan.children`, call `graph.getState(checkpointConfig(projectId, childNodeId))` for each; reject `COMPOSITE_CHILD_NOT_READY` if any child has `evaluation === null`; otherwise invoke graph with the agent's consolidated `WorkReport`. Also: `resume` variant uses `new Command({resume})` only (invoke-input Commands accept only `resume` per LangGraph 1.x docs).
- `packages/solo/src/graph/retry-count.ts` — retry-reset marker awareness: `max(0, evalsAfterLatestResetMarker - 1)`, fallback to `max(0, totalEvals - 1)`
- `docs/phase-1-solo/spec.md` — §8.3 error-code table: strike/remove `BRIEF_KIND_MISSING` row; add `COMPOSITE_CHILD_NOT_READY` row; fix `PREFLIGHT_SECRET_DETECTED` remediation hint (currently references `foreman__token_request` tool which does not exist under 2-tool surface) — rewrite as `foreman step --signal '{"type":"token_request",...}'` with note that M3 rejects until M4 ships (v0.6 Codex finding)
- `packages/solo/src/graph/build.ts` — unknown-signal default throws `SCHEMA_VALIDATION_FAILED` (not silent END)
- `packages/solo/src/mcp/adapter.ts` — collapse to 2 tools (no fallback seam per v0.3 YAGNI)
- `packages/solo/src/cli/index.ts`, `app.ts` — wire the `step` subcommand
- `docs/error-codes.md` — mark `BRIEF_KIND_MISSING` removed; add `COMPOSITE_CHILD_NOT_READY` (added M3)
- `.dependency-cruiser.cjs` — allow `handlers/step.ts` → `graph/*` + `handlers/mutex.ts`
- `tests/unit/signal.test.ts` — update imports post-split (no longer imports removed `SignalSchema`)
- `tests/unit/error-codes.test.ts` — count stays 8 (BRIEF_KIND_MISSING out, COMPOSITE_CHILD_NOT_READY in)

**Deleted files (M3):**

- ✅ `packages/solo/src/handlers/initialize.ts`, `packages/solo/src/handlers/plan.ts` — folded into `step.ts` (shipped Batch G.2, commit `b2b3dce`)

**Migrated tests + demo scripts (Batch G explicit — 8 files total; `docs/error-codes.md` is owned by Batch A, not duplicated here):**

- ✅ `packages/solo/tests/scenario/walking-slice.test.ts` — rewrote `handlers.initialize` / `handlers.plan` calls to `handlers.step` (shipped Batch G.1, commit `85c0356`)
- `packages/solo/tests/scenario/walking-slice/transcript.jsonl` — update expected events post-collapse
- `packages/solo/tests/integration/handlers.test.ts` — M2 handler unit tests updated for collapsed handler
- `packages/solo/tests/integration/handlers-extra.test.ts` — same as above
- `packages/solo/tests/integration/cli.test.ts` — step-subcommand coverage
- ✅ `scripts/demo.ts` (repo root) — migrated handler call sites to `handlers.step`; fixed D-M3-16 payload.kind regression (shipped Batch G.3, commit `2d2c208`; now in CI via `pnpm demo`)

**Structural count (Codex Round 11 acknowledgment):** `packages/solo/src/cli/` = **8 files** post-Batch F (`bin, index, init, mcp-mode, output, status, step, types`). Earlier planning text sometimes said 7 — the 8th file is `types.ts` (extracted for D-17 boundary discipline). Accepted as correct.

- `scripts/walkthrough.ts` (repo root) — same
- `docs/dogfooding-protocol.md` — write operator walkthrough

**Intermediate-state CI note:** Batches C–F leave `walking-slice.test.ts` failing (~4 broken tests) until Batch G migrates them. This is intentional (migration cost lives with the demo/regression batch). Do not block batch commits on walking-slice failures; Batch G closes the gap.

### 3.6 Test inventory (v0.3 — YAGNI-trimmed)

- **Unit** (~10 tests): schema accept/reject for 7 writes + status; retry α-only shape; `retry-reset` journal event shape (`data: { nodeId }`, no `reason` field — v0.3 YAGNI); mutex concurrency + cleanup on throw; retry-count after reset (incl. double-reset, no-reset fallback).
- **Integration** (~30 tests): 7 step pre-checks × happy + error paths; interrupt + resume × approve/override/fail routing; retry from capped; composite `getState` aggregation; `COMPOSITE_CHILD_NOT_READY` pre-check; MCP/CLI step round-trip.
- **Contract** (~30 tests — `transport-handler.test.ts` only; `handler-graph.test.ts` deferred to M5): boundary-focused coverage of Zod parse → handler → response envelope shape.
- **Regression migration**: walking-slice rewrite; M2 handler tests updated for collapse.

Rough total: M2 (190) + M3 additions (~70 new + ~10 migrated) → ~270 at M3 close.

### 3.7 dep-cruiser updates

- Allow `handlers/step.ts` → `graph/*`, `handlers/mutex.ts`, `types/*`, `errors/*`.
- Keep D-17: `mcp/` / `cli/` no direct `graph/`; `graph/` no `handlers/` or transports.
- Keep D-21: LangGraph imports stay `graph/`-only. If `Command` type must cross into `handlers/`, proxy via `graph/` barrel export.
- Keep D-18: no skill-file reads; signal-embedded skill only.

### 3.10 Error code changes (v0.3 net-zero)

**M3 removes (v0.3 M2 retrospective cleanup):**

- `BRIEF_KIND_MISSING` — defined in M2 starter catalog but **never thrown by production code** (verified: zero throw sites; Zod parse surfaces as `SCHEMA_VALIDATION_FAILED` with `details.issues`). Delete from `ERROR_CODE_VALUES`, from M2 `error-codes.md` starter table (with a "removed M3" note), and from `tests/unit/signal.test.ts` / `error-codes.test.ts` assertions.

**M3 adds:**

- `COMPOSITE_CHILD_NOT_READY` — composite parent `eval` signal arrives before all children reach terminal state. Thrown by `handlers/step.ts` sequence-validity pre-check.

**M3 does not add (v0.3 YAGNI):**

- ~~`STALE_NODE`~~ — TTL deferred to Phase 2a.
- ~~`NODE_LOCKED`~~ — mutex deadlocks are bugs, not operator-surfaced codes; map to existing codes internally if needed.
- ~~`NOT_IMPLEMENTED`~~ — `token_request` handler returns `SCHEMA_VALIDATION_FAILED` with `details: { reason: 'token_request landing at M4' }`, consistent with the INVALID_RESUME_STATE pattern we already established.

`OPERATOR_APPROVAL_REQUIRED` remains as a sequence-validity check per ADR 0003 §4.

**Net count:** 8 (M2) − 1 (BRIEF_KIND_MISSING) + 1 (COMPOSITE_CHILD_NOT_READY) = **8 codes**. `error-codes.test.ts` assertion stays 8. M2→M3 is a **cleanup**, not an expansion.

## 4. Decisions (D-M3-1..D-M3-15; 3 dropped in v0.3)

| ID          | Topic                      | Resolution                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  | Grounding                                         |
| ----------- | -------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| D-M3-1      | Mutex shape                | In-handler async lock map keyed by `${projectId}:${nodeId}`; `try/finally` cleanup mandatory; Batch E spike verifies cleanup under `interrupt()`-thrown `GraphInterrupt` before Batch C + D merge                                                                                                                                                                                                                                                                                                                                                                                                                           | Spec §3.4, D-17; round-9 Codex                    |
| D-M3-2      | Signal split               | `StepSignalSchema` (7 writes) + `StatusSignalSchema` (1 read); drop top-level `SignalSchema` export. Batch A updates `tests/unit/signal.test.ts` imports concurrently                                                                                                                                                                                                                                                                                                                                                                                                                                                       | ADR 0003 §2–§3; round-9 Codex                     |
| D-M3-3      | Handler collapse           | Polymorphic `handlers/step.ts`; delete M2's `initialize.ts` + `plan.ts`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     | ADR 0003 §2; M2→M3 refactor                       |
| D-M3-4      | Dispatch pattern           | Exhaustive discriminant `switch` on `signal.type` with `default:` branch that throws `SCHEMA_VALIDATION_FAILED` (not silent END). Also update `graph/build.ts` dispatch-on-signal default to throw                                                                                                                                                                                                                                                                                                                                                                                                                          | D-21, S-4, spec §5.9; round-9 Codex               |
| D-M3-5      | MCP surface                | 2 tools: `foreman__status` + `foreman__step`; drop 3 legacy tool registrations. No pre-factored fallback seam (v0.3 YAGNI — Amendment A reserves to M4 if needed)                                                                                                                                                                                                                                                                                                                                                                                                                                                           | ADR 0003 §2; C3 spike PASS                        |
| D-M3-6      | Retry-reset schema         | Append `{type:'retry-reset', nodeId, actor, ts}` journal event (no `reason` field at M3 — v0.3 YAGNI; forward-compat to re-add). Clear `evaluation` channel to `null`. **Write path:** mutations go through the graph state reducers + checkpointer API (per architecture §5.2) — no raw SQLite writes, no direct journal-file writes. Handler calls `graph.updateState()` or an equivalent reducer-driven path (v0.6 Codex clarification)                                                                                                                                                                                  | ADR 0003 §2.5                                     |
| D-M3-7      | Retry-count algorithm      | `retry-count.ts` returns `max(0, evalsAfterLatestResetMarker - 1)`; fallback to `max(0, totalEvals - 1)` if no reset marker exists (M2 compatibility). Double-reset yields 0, not negative                                                                                                                                                                                                                                                                                                                                                                                                                                  | ADR 0003 §2.5, I-8; round-9 Codex                 |
| D-M3-8      | Resume Command shape       | **Amended v0.7** (LangGraph docs verification): handler sends `new Command({resume: payload})` as invoke input — per official docs, invoke-input Commands accept **only** `resume`; `goto`/`update`/`graph` are node-return-only. Fail-routing lives inside `evaluate.ts` as node-return `Command({update: {journal}, goto: END})`; approve/override return plain state-update and the static `n_evaluate → END` edge handles routing. `{ends: [END]}` on `n_evaluate.addNode` registers END as a valid dynamic goto target.                                                                                                | ADR 0003 §2.5 (v0.7 amended), LangGraph 1.x docs  |
| D-M3-9      | Composite work pre-check   | **Amended v0.7 (Option B, user decision 2026-04-18):** composite aggregation lives in `handlers/step.ts` `work` pre-check, NOT in `evaluate.ts`. When `state.kind === 'composite'`, handler calls `graph.getState(checkpointConfig(projectId, childNodeId))` for each child in `plan.children`; rejects `COMPOSITE_CHILD_NOT_READY` if any lacks evaluation. Composite parents HAVE a work phase (agent consolidates children's outputs using `brief.skill.methodology.work` prompt); evaluate node is kind-agnostic. Rationale in ADR 0004 §"Composite parent work phase = consolidation" + hexagonal rationale paragraph. | ADR 0004 (v0.7 Option B amendment)                |
| ~~D-M3-10~~ | ~~TTL policy~~             | **Dropped in v0.3.** TTL deferred to Phase 2a; see §6 out-of-scope. Rationale: disabled-by-default at M3 = dead code in Solo (single-user, no opt-in pressure)                                                                                                                                                                                                                                                                                                                                                                                                                                                              | v0.3 YAGNI pass                                   |
| ~~D-M3-11~~ | ~~Stale threshold~~        | **Dropped in v0.3.** No `ttlHours` consumer at M3 after D-M3-10 drop. Config key stays defined (M2 shipped it) but unconsumed; M4+ wires or removes                                                                                                                                                                                                                                                                                                                                                                                                                                                                         | v0.3 YAGNI pass                                   |
| D-M3-12     | CLI shape                  | `foreman step --signal <json>`; argument `-` reads signal JSON from stdin. No `--signal-file` flag (shell + stdin cover the ergonomic case). Drop `foreman resume --action` subcommand                                                                                                                                                                                                                                                                                                                                                                                                                                      | ADR 0003 §5, spec §2.5; round-9 + operator review |
| D-M3-13     | Dogfooding kickoff         | Start at M3 close with raw JSON signals. **Batch G** writes `docs/dogfooding-protocol.md` — what signals to send, what to observe, what passes/fails                                                                                                                                                                                                                                                                                                                                                                                                                                                                        | Spec §2.5/§15                                     |
| D-M3-14     | token_request at M3        | Handler rejects with `SCHEMA_VALIDATION_FAILED` + `details: { reason: 'token_request lands at M4' }` (v0.3 YAGNI — no dedicated `NOT_IMPLEMENTED` code). Signal schema stays in `StepSignalSchema` for forward-compat; test verifies rejection                                                                                                                                                                                                                                                                                                                                                                              | ADR 0003; round-9 + YAGNI pass                    |
| ~~D-M3-15~~ | ~~Composite parent retry~~ | **Dropped in v0.3.** The ADR 0004 "non-cascading" semantic is the _natural default_ behavior — retry on any node (terminal or composite) re-enters plan with counter reset, affects that thread only. No special handling needed; no decision to lock in M3 proposal                                                                                                                                                                                                                                                                                                                                                        | ADR 0004 natural default                          |
| D-M3-16     | M2 retrospective cleanup   | (added v0.3) In Batch A: (a) **Remove `BRIEF_KIND_MISSING`** from `ERROR_CODE_VALUES` — never-thrown starter code; Zod rejection surfaces as `SCHEMA_VALIDATION_FAILED + details`. (b) **Consolidate `kind` field** — drop `payload.kind` from `InitializePayloadSchema`; `brief.kind` is single source of truth                                                                                                                                                                                                                                                                                                            | M2 retrospective YAGNI (Codex + Copilot)          |

## 5. Batch order (A–H) — v0.3 merged + YAGNI-trimmed

| Batch                                                                 | Items covered                                                                                                                                                                                                                               | Depends on             | Est. files / LoC / hours |
| --------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------- | ------------------------ |
| A — Types + errors + M2 cleanup                                       | D-M3-2 (signal split + `signal.test.ts` update + `kind` consolidation per D-M3-16), D-M3-6 (extend `JournalEventTypeSchema` with `retry-reset`), §3.10 error-code delta (remove BRIEF_KIND_MISSING, add COMPOSITE_CHILD_NOT_READY)          | none                   | 7 files / ~260 LoC / 5h  |
| B — Mutex primitive                                                   | D-M3-1 (mutex with try/finally cleanup tests)                                                                                                                                                                                               | A                      | 3 files / ~180 LoC / 5h  |
| C — Step handler (prelim) + transport contract                        | D-M3-3, D-M3-4 (switch default throws), D-M3-6 (writes retry-reset marker), D-M3-8 handler-side (invoke input `Command({resume})` only), D-M3-14 (token_request → SCHEMA_VALIDATION_FAILED stub); `transport-handler.test.ts` authored here | A, B                   | 8 files / ~560 LoC / 11h |
| D — Evaluate (kind-agnostic) + composite work pre-check + retry-count | D-M3-7 (algorithm fix), D-M3-8 evaluate-side (`interrupt()` on block + node-return `Command({update, goto: END})` for fail), D-M3-9 **Option B** (composite readiness pre-check in `handlers/step.ts` work handler — NOT in evaluate node)  | A, B, C (prelim)       | 5 files / ~420 LoC / 9h  |
| **E — Mid-WORK spike + GAN (round 9.5)**                              | Unified spike: interrupt-cleanup + resume-goto + composite-retry integration in one script. Dispatch Codex + Copilot on C + D. **Fires before C + D final merge per D-M3-1**; any findings fold into C/D remediation commits                | C (prelim), D (prelim) | spike + review / ~6h     |
| F — Transport: MCP adapter + CLI `step` (merged)                      | D-M3-5 (2-tool collapse), D-M3-12 (`--signal` + stdin)                                                                                                                                                                                      | C, E                   | 7 files / ~380 LoC / 7h  |
| G — Regression + demo + dogfooding protocol                           | R-M3-7 migration (8 files per §3.2), D-M3-13 (write `docs/dogfooding-protocol.md`), `scripts/demo.ts` + `scripts/walkthrough.ts` call-site update                                                                                           | D, F                   | 8 files / ~420 LoC / 8h  |
| H — Post-WORK GAN (round 10) + verify + commit                        | full tree walk, evaluation protocol                                                                                                                                                                                                         | G                      | — / — / 5h               |

Totals: ~38 files touched, ~2180 LoC, 13 active decisions + 1 unified spike, **56h** execution (v0.5: −1 file −30 LoC from de-duplicating `docs/error-codes.md` from Batch G; v0.4: +1 file +80 LoC +1h from transport-handler.test.ts → Batch C; v0.3: 55h; v0.2: 63h).

## 6. Out of Scope for M3

- `PromptAssemblerPort`, `SkillMethodologyAssembler`, example skills (M4).
- `TokenVendorPort` + real token vending (M4; D-M3-14 uses SCHEMA_VALIDATION_FAILED stub).
- **TTL / stale detection** (v0.3 YAGNI defer to Phase 2a — D-M3-10/11 dropped).
- Daemon / HTTP / RBAC / UI / dashboard / telemetry / cross-project aggregation (Phase 2a+).
- Composite child auto-spawn or parent auto-evaluate — wrapper drives per ADR 0004.
- Scenario-suite G4 closure (M5), security-suite G6 closure (M5), v1 release packaging (M6).
- Contract test suite `handler-graph.test.ts` (graph-shape boundary — stable from M2; migrate at M5 alongside scenarios).
- 4-tool MCP fallback implementation + pre-factored seam (v0.3 YAGNI; Amendment A reserves M4 if host rendering fails in dogfooding).

## 7. Risks (v0.3 — R-M3-9 dropped with TTL)

| ID          | Risk                                                                                                                | Likelihood | Mitigation                                                                                                                 |
| ----------- | ------------------------------------------------------------------------------------------------------------------- | ---------- | -------------------------------------------------------------------------------------------------------------------------- |
| R-M3-1      | LangGraph 1.x `interrupt()` / `Command({goto})` surprises (akin to ADR 0002 composite surprise)                     | Medium     | Batch E unified spike before Batch C/D final logic                                                                         |
| R-M3-2      | Lock deadlock around interrupted graph writes                                                                       | Medium     | Try/finally mutex; Batch B tests cover throw-during-invoke; D-M3-1 explicit                                                |
| R-M3-3      | Polymorphic `foreman__step` renders poorly in some MCP hosts despite C3 protocol PASS                               | Medium     | C3 PASS; Amendment A reserves to M4 if dogfooding hits the wall (no pre-factored seam — v0.3 YAGNI)                        |
| R-M3-4      | Retry-reset marker off-by-one or double-reset                                                                       | Medium     | D-M3-7 restated as `max(0, … - 1)`; unit matrix covers no-marker, single-marker, double-reset, empty journal               |
| R-M3-5      | Composite child readiness ambiguity                                                                                 | Medium     | `COMPOSITE_CHILD_NOT_READY` pre-check; wrapper-driven aggregation (ADR 0004)                                               |
| R-M3-6      | Transport drift after MCP collapse                                                                                  | Low        | Integration asserts only two registered tools                                                                              |
| **R-M3-7**  | Old M2 tests assume three tools — specifically `walking-slice.test.ts` + `transcript.jsonl` + M2 handler unit tests | **High**   | Batch G explicit migration with file names listed in §3.2                                                                  |
| R-M3-8      | Dogfooding at M3 close surfaces UX friction (raw JSON signals unergonomic)                                          | Medium     | D-M3-12 stdin convention (`--signal -`) + shell tools cover scripted workflows; protocol doc at Batch G; full skills in M4 |
| ~~R-M3-9~~  | ~~Stale flag confusing~~                                                                                            | —          | Dropped — TTL deferred to Phase 2a                                                                                         |
| **R-M3-10** | Handler-collapse loses set-once reducer invariant thread-through                                                    | Medium     | Batch C pre-check matrix tests all 7 signal types against nodeId/kind/brief set-once                                       |
| **R-M3-11** | MCP tool rename breaks any pre-release docs referencing `foreman__initialize`/`foreman__plan`                       | Medium     | ADR 0003 clearly supersedes; M2 unpublished (version 0.0.0); low real consumers                                            |
| R-M3-12     | CLI UX regression — operators used to `foreman resume --action` face `foreman step --signal '{...}'`                | Low        | Stdin convention covers scripted; shell tools ($(cat), heredocs) cover file-driven; M4 sugar subcommands planned           |
| R-M3-13     | Grandchild lifecycle ambiguity                                                                                      | Low        | ADR 0004 §Grandchild-ownership: wrapper-driven at every tree level; documented in `docs/dogfooding-protocol.md`            |
| R-M3-14     | LangGraph `Command` import leaks into `handlers/` boundary violating D-21                                           | Low        | Proxy `Command` construction through `graph/` barrel export; dep-cruiser enforces                                          |

## 8. GAN Review Plan

- **Round 9 — Mid-WORK (planning-level, completed 2026-04-18):** v0.1 → v0.2 (round 9 findings merged).
- **v0.3 YAGNI pass (completed 2026-04-18; this doc):** Codex + Copilot parallel YAGNI review on v0.2 + M2 retrospective. 13 items resolved (drop/simplify/defer/keep). Both agents converged. No further planning-level GAN round required.
- **Round 9.5 — Mid-WORK code spike + GAN (after Batch D; inside Batch E):** Unified spike (interrupt cleanup + resume goto + composite retry). Dispatch Codex + Copilot on actual C + D code.
- **Round 10 — Post-WORK (after Batch G; Codex + Copilot in parallel):** End-to-end consistency, error path coverage, dogfooding-protocol-doc quality, dep-cruiser boundary compliance, ADR 0003/0004 trace-back, round-9.5 remediation.

## 9. Evaluation Protocol (post-WORK)

- PASS: all seven write variants accepted through `foreman__step` (`token_request` rejected with structured error envelope verified by test).
- PASS: `foreman__status` read-only.
- PASS: tests cover variants × happy + error paths, interrupt/resume, retry (incl. composite non-cascading), composite aggregation, mutex (incl. throw-during-invoke), MCP two-tool, CLI step + stdin (`--signal -`), M2 regression (incl. walking-slice migrated).
- PASS: branch coverage for `graph/` + `handlers/` stays ≥80% (vitest thresholds); no skipped tests.
- PASS (G7 net-zero): `error-codes.test.ts` asserts exactly 8 codes; `BRIEF_KIND_MISSING` is absent; `COMPOSITE_CHILD_NOT_READY` is present.
- PASS: `pnpm typecheck` / `lint` / `test` / `format:check` / `boundary-check` / `licenses:check` / `demo` all green.
- PASS: `.foreman/` dogfooding initialized in repo; `docs/dogfooding-protocol.md` exists with concrete operator walkthrough.
- PASS: round-10 GAN clean.
- If clean → tag `m3-alpha`; open M4 proposal.

## 10. Duration

- **Estimate:** 10–14 session-work days (v0.3 trimmed from v0.2's 12–16).
- **Confidence:** Medium-high.
- **Basis:** M2 took ~7 actual days for ~5 scope items; M3 has 13 active decisions + 1 unified spike + 2 M2-retrospective cleanups. Batch F+G merged; TTL deferred; Batch E combined saves ~8h.
- **Lower bound:** ~9 days (spike passes clean; mutex straightforward).
- **Upper bound:** ~17 days (Batch E spike surfaces LangGraph `goto` misbehavior requiring D-M3-8 amendment).
- **Compression risk:** Medium. Batch G (regression + dogfooding protocol doc) should not be split ahead of Batch F.

## 11. Open tensions (remaining after v0.3)

Resolved in v0.2 + v0.3 (removed from this list): error code naming + count (§3.10 locked); dogfooding protocol doc (Batch G task D-M3-13); TTL default (deferred to Phase 2a per v0.3); composite retry semantics (ADR 0004 amendment + natural default, no decision needed); grandchild ownership (ADR 0004 §Grandchild-ownership); retry α-only (locked as design per D-M3-7); `--signal-file` (dropped; stdin convention instead); contract test scope (transport-only at M3, graph-shape at M5).

Still open for round-10 scrutiny:

1. **Mutex shape final check** (D-M3-1). In-handler map chosen; Batch E spike verifies cleanup under `GraphInterrupt` throws. Only remaining open item.

## 12. Next steps

- v0.3 YAGNI, v0.4 integrity, v0.5 verification, v0.6 hexagonal-lens, v0.7 LangGraph-docs + Option B + code-shipping amendments **all completed** (this doc).
- ✅ Batches A-G **shipped** (see front-matter `commits:` chain). Tests 275→271 across G-consolidation; all gates + `pnpm demo` green in CI.
- ✅ Batch E Round 9.5 GAN: Codex 2 HIGH + 2 LOW + Copilot gaps merged (`20c836c`).
- ⏳ **Batch H in progress:** doc drift cleanup (this commit), Round 10 post-WORK GAN on integrated M3, §9 evaluation-protocol verify, tag `m3-alpha`.
- **Known follow-ups (Scout):** (a) `vitest.config.ts` has no coverage `thresholds` block despite proposal G5 claim "≥80% branch coverage on graph/+handlers/" — coverage is reported but not enforced. Either add `thresholds: { branches: 80, ... }` or strike the G5 claim before v1. (b) M4 proposal authoring (skills + PromptAssemblerPort + TokenVendorPort).
- If round 10 clean: tag `m3-alpha`; open M4 proposal for ports/adapters + prompt assembly.
