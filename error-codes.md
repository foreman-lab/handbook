---
status: M3 Batch A — 8 codes (net-zero vs M2: −BRIEF_KIND_MISSING +COMPOSITE_CHILD_NOT_READY; catalog locks at M6 per G7)
baselined: 2026-04-17
last_updated: 2026-04-18 (M3 Batch A: BRIEF_KIND_MISSING removed — never thrown; COMPOSITE_CHILD_NOT_READY added per ADR 0004 + M3 proposal §3.10)
next_sweep: M6 (G7 full-catalog enforcement)
---

# foreman Error Code Catalog

This document ships with M1 (Contracts) and carries the 7 starter codes declared in `packages/solo/src/types/error-codes.ts`. During M2–M5 WORK, new codes introduced to the codebase are added here with an ADR-lite note stating which milestone introduced them and which `BaseError` subclass throws them. M6 sweep (gate G7) audits that every thrown `BaseError` uses a code listed here and locks the catalog for v0.1.

## Runtime shape

All codes live in `ErrorCodeSchema = z.enum(ERROR_CODE_VALUES)` (runtime-validated Zod enum). The TypeScript type `ErrorCode` is inferred via `z.infer<typeof ErrorCodeSchema>`. Consumers should narrow at adapter boundaries and rely on inference internally.

## D-5 — 4-class error taxonomy

Per foundations decision **D-5** (see `docs/foundations.md`), every `BaseError` maps to exactly one of these subclasses:

- `ValidationError` — input/schema violations.
- `SecurityError` — policy/approval/secret-scan failures.
- `StorageError` — persistence/checkpointer faults.
- `BaseError` — only when no subclass fits (should be rare; review in M6 sweep).

No other subclass may be introduced without an ADR updating D-5.

## Starter codes (M1)

| Code                          | Class                 | When thrown                                                                                                                                                                                                           | Fix                                                                                                                                  |
| ----------------------------- | --------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| `SCHEMA_VALIDATION_FAILED`    | `ValidationError`     | Signal did not match Zod schema (top-level or payload)                                                                                                                                                                | Check signal shape against `packages/solo/src/types/signal.ts`                                                                       |
| ~~`BRIEF_KIND_MISSING`~~      | ~~`ValidationError`~~ | **Removed M3** (Batch A, proposal §3.10): never thrown by production code — Zod rejection surfaces as `SCHEMA_VALIDATION_FAILED` with `details.issues`. Kept as a strike-through row for history; do not reintroduce. | —                                                                                                                                    |
| `NODE_NOT_FOUND`              | `ValidationError`     | Signal targeted an unknown `nodeId`                                                                                                                                                                                   | Verify node was initialized via a prior `step(type:'initialize')` signal                                                             |
| `OPERATOR_APPROVAL_REQUIRED`  | `SecurityError`       | Irreversible ToolCall submitted without prior `ResumeSignal { action: "approve" }` (enforcement lands M2+)                                                                                                            | Send resume with `approve` or `override`, with justification                                                                         |
| `PREFLIGHT_SECRET_DETECTED`   | `SecurityError`       | Work report contains a credential pattern matched by preflight scan (enforcement lands M2+)                                                                                                                           | Remove secret from report; use `foreman step --signal '{"type":"token_request",...}'` for scoped creds (available at M4; M3 rejects) |
| `CHECKPOINTER_WRITE_FAILED`   | `StorageError`        | SQLite/LangGraph checkpointer write error (disk full, FS unavailable) (enforcement lands M2+)                                                                                                                         | Check disk; foreman auto-retries once before surfacing                                                                               |
| `JOURNAL_CORRUPTION_DETECTED` | `StorageError`        | Journal event deserialization failed (enforcement lands M2+)                                                                                                                                                          | Inspect via `foreman inspect`; restore from prior checkpoint or `foreman init --force`                                               |

**Note on "enforcement lands M2+":** the 7 codes listed above are defined in M1 as part of the type contract. The runtime paths that throw them are implemented in M2 onward (graph nodes, preflight, checkpointer adapters). M1 ships only the vocabulary; M2+ ships the behavior.

## M2 additions

| Code             | Class          | When thrown                                                                                                                 | Fix                                                                       |
| ---------------- | -------------- | --------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| `NO_FOREMAN_DIR` | `StorageError` | (added M2 — cli/status.ts, cli path guard) CLI command other than `init` ran in a directory without `.foreman/` (arch §7.1) | Run `foreman init` in the target directory first, then re-run the command |

## M3 additions

| Code                        | Class             | When thrown                                                                                                                                                                                                                                        | Fix                                                                                                     |
| --------------------------- | ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------- |
| `COMPOSITE_CHILD_NOT_READY` | `ValidationError` | (added M3 Batch A per proposal §3.10 + ADR 0004 §Composite-retry-non-cascading) Composite parent's `step(type:'evaluate')` arrives before all children reach terminal state; enforced by sequence-validity pre-check in `handlers/step.ts` at Batch C. | Wait for all children to reach an evaluated/completed state, then re-submit the composite parent `evaluate` |

## Adding a code (M2–M5 procedure)

1. Add the new literal to `ERROR_CODE_VALUES` in `packages/solo/src/types/error-codes.ts`.
2. Update `error-codes.test.ts` count assertion.
3. Append a row to the table above with the milestone tag in the "When thrown" column (e.g., `(added M3 — handlers/mutex)`).
4. Throw the new code from exactly one `BaseError` subclass per D-5 (see taxonomy above).
5. Tests for the thrown path must assert the code value, not just `instanceof BaseError`.

## M6 sweep (G7)

The M6 sweep audits:

- Every `throw new BaseError(...)` or subclass uses a code listed in this file.
- Every code in this file is thrown from at least one place in the codebase.
- No orphan codes (added but never thrown) and no ad-hoc string literals.
- Error-code doc and `error-codes.ts` stay in sync (CI check).

Once clean, the catalog is locked for v0.1 release.
