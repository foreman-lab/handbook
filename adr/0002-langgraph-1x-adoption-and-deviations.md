---
status: Approved v2
approved_by: user (kidtsui@outlook.com)
approved_date: 2026-04-18
date: 2026-04-18
supersedes: none
amends:
  - docs/phase-1-solo/spec.md §9 (checkpoint_ns semantics)
  - docs/phase-1-solo/architecture.md §2.3, §4.4 Scenario 3, Appendix A, §3 R-1 (thread_id/ns model + LangGraph version)
  - docs/phase-1-solo/m2-development-proposal.md v7 → v8 (D-M2-3 checkpoint config mapping, §3.1 dep pins, §3.2 router return type, §3.6 test wording, R-M2-5 risk text, §10 duration text)
companion: docs/foundations.md (D-21 harness-identity + protocol-integrity invariant; engine-neutral; landed at foundations.md:440)
---

# ADR 0002: LangGraph 1.x adoption — checkpoint namespace deviation + 1.x API hazards

## Status and scope

Approved v2 (2026-04-18). GAN round 1 (Codex REVISE + Copilot 10 findings) applied in v2. GAN round 2 (Codex + Copilot debate during proposal v8 patch pass, 2026-04-18) surfaced two further fixes incorporated here: allowlist regex typo (`.*` → `._-`) and M2 proposal amendment scope expanded to cover §3.1 pins + §3.2 router return type + §10 duration risk text. No open findings.

This ADR captures **LangGraph-1.x-specific** decisions. Engine-neutral rules (harness identity, deterministic semantics, etc.) are companion amendments to `foundations.md` and are NOT restated here.

## Context

The M2 proposal (v7, approved after 6 GAN rounds) targeted LangGraph 0.2.x. During M2 Batch A (`pnpm add`), the registry resolved to 1.x because the 0.2.x line was released/EOL'd while the proposal was under review. Resolved stack:

| Library                                  | Proposal (v7) | Installed  | Delta                                                           |
| ---------------------------------------- | ------------- | ---------- | --------------------------------------------------------------- |
| `@langchain/langgraph`                   | 0.2.x         | **1.2.9**  | 0.x → 1.x (initial stable, not a "true major" in strict semver) |
| `@langchain/core`                        | 0.3.x         | **1.1.40** | 0.x → 1.x                                                       |
| `@langchain/langgraph-checkpoint-sqlite` | 0.1.x         | **1.0.1**  | 0.x → 1.x                                                       |
| `@modelcontextprotocol/sdk`              | 1.x           | 1.29.0     | within major                                                    |
| `commander`                              | 12.x          | **14.0.3** | 12 → 14 (true major)                                            |
| `better-sqlite3`                         | 11.x          | **12.9.0** | 11 → 12 (true major)                                            |

**Note on 0.2.x EOL:** the ADR draft previously asserted LangChain had EOL'd 0.2.x. This is **unverified by upstream source as of the ADR date**. Reclassified as operational risk: 0.2.x continues to receive no new feature work from LangChain; pinning creates a growing backport burden. See "Alt 1" rejection below.

### Issues surfaced during Batch C (verified with spike)

1. **Node-name / channel-name collision.** LangGraph 1.x rejects node names that collide with channel names at compile time. Fixed locally: node ids prefixed (`n_init`, `n_plan`, `n_work`, `n_evaluate`). Channel names (`plan`, `work`, `evaluation`, `journal`, etc.) remain per spec §9.

2. **`checkpoint_ns = nodeId` non-functional at top level (scope-narrowed claim).** Spike `packages/solo/spike.mjs` confirmed: invoking `graph.invoke(input, { configurable: { thread_id, checkpoint_ns: 'n1' } })` against a top-level `StateGraph` writes to the checkpointer with `checkpoint_ns = ''`. The SqliteSaver's SQL WHERE clause is exact-match on `checkpoint_ns`; `getState({..., checkpoint_ns: 'n1'})` therefore returns empty values.

   **What is proven:** top-level invocations do NOT propagate a caller-provided non-empty `checkpoint_ns` to the checkpointer.

   **What is NOT yet proven:** whether non-empty `checkpoint_ns` is intentionally reserved for LangGraph subgraph routing (the plausible architectural reading), or is an undocumented behavior/bug in 1.2.9. A **composite/Send spike is required at M3 planning** to verify Scenario 3 compatibility before the composite model lands.

3. **SqliteSaver setup semantics.** `SqliteSaver.setup()` is auto-called lazily by `getTuple()` / `list()` / `put()` (guarded by an `isSetup` flag). Not a user-visible issue; no explicit call required. Implementation note for `foreman init`: priming the DB file via a short-lived `better-sqlite3` connection + `PRAGMA journal_mode = WAL` is still useful because SqliteSaver only writes schema on first use.

## Decision

**ADAPT to LangGraph 1.x** with spec deviations documented here. Do not pin to 0.2.x.

### Amendments to spec §9 (checkpoint semantics)

Old (pre-1.x):

> Thread ID: `projectId` (one thread per `.foreman/` project).
> Checkpoint namespace: `nodeId` (one ns per node within project).

New (M2 onward):

> **Thread ID:** composite `${projectId}:${nodeId}` (one thread per node within a project).
> **Checkpoint namespace:** empty string `''` for terminal nodes. Non-empty `checkpoint_ns` is **reserved for LangGraph-managed subgraph routing** (composite children, M3+).
> **Per-node isolation** is achieved via the composite thread\*id, not via namespace.
> **Input validation** (already in place via `checkpoint-config.ts`): both `projectId` and `nodeId` match `/^[A-Za-z0-9._-]{1,256}$/` (literal dot, underscore, hyphen inside the character class; no wildcard). The allowlist excludes `:`, `/`, `\`, control characters, `..`, and path-traversal sequences, so the composite thread_id has no separator-collision vulnerability.

### Amendments to architecture.md (required by Codex round-1 review)

- **§2.3 (thread / namespace model):** update "one `thread_id` per project" and "`checkpoint_ns = nodeId`" to the composite-thread_id model above.
- **§4.4 Scenario 3 (parent + 2 children composite):** rewrite parent queries from `getState({thread_id: projectId, checkpoint_ns: 'c1'})` to `getState({thread_id: '${projectId}:${parentId}', checkpoint_ns: 'c1'})`. **Mark the exact subgraph-query shape as TO-VERIFY at M3 composite planning** (contingent on the composite/Send spike).
- **Appendix A glossary:** update "Thread ID" and "Checkpoint namespace" definitions.
- **§3 R-1:** update dependency baselines to installed 1.x versions.

### Amendments to M2 proposal (v7)

- **D-M2-3:** new thread_id / checkpoint_ns mapping per above.
- **§3.1 dep pins:** reflect installed major versions (§3.12's "`N` placeholder → lockfile authority" clause still applies for patch numbers).
- **§3.2 graph/ file 9 (`edges/outcome.ts`):** router returns `'pass' | 'retry' | 'end'` (mapped to LangGraph edge target). Proposal text `'END' | 'plan'` was stale.
- **§3.6 `handler-checkpoint-restore.test.ts`:** wording updated — test the new thread_id shape; journal persists across close/reopen.
- **R-M2-5 risk text:** update from "LangGraph 0.2.x churn" to "LangGraph 1.x minor/patch churn (pinned)".

### LangGraph 1.x API hazards (NOT to be adopted at Phase 1)

These are engine-specific prohibitions. The corresponding timeless invariants live in `foundations.md` (companion amendment); this list exists for M2-M6 reviewer quick-reference:

- **`createReactAgent`, `ToolNode`, `ToolExecutor`** — prebuilt agent/tool-runtime primitives. foreman is a harness, not an agent runtime.
- **`RetryPolicy` on nodes** — retries node EXECUTION, not checkpointer writes. Would produce duplicate journal events and violate journal-derived determinism (arch I-7/I-8).
- **`CachePolicy` on nodes** — caches computed outputs; opaque to journal reconstruction.
- **`withLangGraph` / `schemaMetaRegistry` inside `types/`** — couples Zod schemas to LangGraph runtime metadata. Allowed inside `graph/` only.
- **`task` / `entrypoint` functional API** — blurs the locked 4-node FSM (arch §4). Phase 1 uses explicit `StateGraph` only.
- **`HumanInterrupt` / `HumanResponse` as a public contract** — vocabulary (`accept/ignore/response/edit`) mismatches foreman's `approve/override/fail`. The types may be used internally behind our resume adapter at M3, but MUST NOT leak through MCP/CLI.
- **`Store` / `BaseStore` as a skill registry** — contradicts D-18 (harness runtime) signal-embedded model. Phase 2a hosted service may maintain a catalog at its service edge (roadmap / Gate 6), not in the harness.
- **LangGraph streaming writers / token streams as default telemetry** — P-7 keeps telemetry opt-in. Any `--stream` CLI must be explicit operator action.

### LangGraph 1.x adoptions (WITH caveats)

- **Node-id prefix `n_`** — convention to avoid channel-name collision (implemented).
- **`Command({ resume, goto })`** — adopt at M3 for resume-to-originNode routing (arch §4.3). Zero M2 cost — design lock only.
- **`Send`** — required at M3 for composite children (arch Scenario 3). Spike before M3 locks the subgraph-query shape.

### Batch A infrastructure note

- Root `package.json` gains `pnpm.onlyBuiltDependencies: ["better-sqlite3"]` to allow the native `gyp` build under the strict `.npmrc` `ignore-scripts=true` policy. **Already applied** in root `/Users/xuzhijie/Desktop/ai/foreman/package.json`.

## Planned code changes (this ADR locks the design; code fixes land in Batch C cleanup)

| Change                                                                                                        | File                                                     | Rule grounding              |
| ------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | --------------------------- |
| Thread-id composition `${projectId}:${nodeId}`, `checkpoint_ns = ''`                                          | `handlers/checkpoint-config.ts` (to be created, Batch D) | ADR 0002 §9 amendment       |
| Set-once reducer uses `node:util.isDeepStrictEqual` (replace `JSON.stringify`)                                | `graph/channels.ts`                                      | Code hardening (Copilot #1) |
| `scanSecrets` uses `matchAll` + `g` flag per pattern                                                          | `graph/preflight.ts`                                     | Code hardening (Copilot #2) |
| Stub nodes `plan.ts`/`work.ts`/`evaluate.ts` throw on null `nodeId`                                           | same                                                     | P-5 fail-fast               |
| Router returns explicit `'block'`/`'capped'` paths with inline comment                                        | `graph/edges/outcome.ts`                                 | Completeness guard          |
| `graph-init.test.ts` test 1 adds `db.close()` in cleanup                                                      | same                                                     | fd-leak fix                 |
| `ValidationError` from set-once reducer carries **summarized** `current` (nodeId + kind only, not full brief) | `graph/channels.ts`                                      | Avoid PII leak              |

## Consequences

**Immediate (M2):** unskip `graph-init` close/reopen test; it passes with new thread_id.

**M3:** run composite/`Send` spike to verify Scenario 3 query shape. If subgraph checkpoints are queryable as `getState({ thread_id: '${projectId}:${parentId}', checkpoint_ns: 'c1' })`, we adopt that pattern. If not, a second ADR amends §4.4.

**Status (2026-04-18): spike ran; hoped query shape FAILED. SUPERSEDED by ADR 0004** (`docs/adr/0004-composite-independent-threads.md`). LangGraph 1.2.9 auto-assigns subgraph namespaces as `<nodeName>:<task-uuid>` (ephemeral), not at operator-chosen handles. Composite children are modeled as independent top-level threads (one thread per node) per ADR 0004; no subgraphs, no `Send`. Architecture §4.4 Scenario 3 is being amended to match the per-thread model. See ADR 0004 Appendix A for spike verdict and `packages/solo/scripts/composite-send-spike.ts` for the reproducible artifact.

**Phase 2a (hosted service):** multi-tenant thread_id shape is `${tenantId}:${projectId}:${nodeId}` — a three-segment composite preserves isolation across tenants. Codex round-1 catch: two-segment `${projectId}:${nodeId}` is NOT sufficient if `projectId` is not globally unique across tenants. Phase 2a ADR will pin the schema.

**Phase 2a skill catalog:** lives at service edge, NOT in harness. roadmap.md / Gate 6 tracks the design.

**Phase 3+ engine switch (Temporal / Inngest):** foundations stay engine-neutral, so the harness-identity + deterministic-semantics invariants transfer. ADR 0002's specific LangGraph 1.x API-hazards list becomes obsolete; a successor ADR supersedes.

## Alternatives Considered

### Alt 1: Pin to LangGraph 0.2.x

**Rejected.** Operational risk (not EOL-confirmed, but: feature work has moved to 1.x; ecosystem agents target 1.x; every patch becomes a manual backport). Verification of LangChain's deprecation policy would be required to upgrade this to a hard claim.

### Alt 2: Fork SqliteSaver to bypass `checkpoint_ns` normalization

**Rejected.** Maintaining a fork is unsustainable; couples foreman to LangGraph internals.

### Alt 3: Single thread per project; encode nodeId in metadata

**Rejected.** Loses LangGraph's natural checkpoint isolation; every `getState` scans-and-filters. Worse complexity, same semantic result.

### Alt 4 (chosen): Composite thread_id `${projectId}:${nodeId}`

**Adopted.** Clean, idiomatic for LangGraph 1.x, preserves per-node isolation, forward-compatible with M3 composite via subgraph `checkpoint_ns` (pending spike).

## References

- Installed source: `node_modules/.pnpm/@langchain+langgraph-checkpoint-sqlite@1.0.1/.../dist/index.js` (ephemeral; see GitHub ref below)
- Upstream GitHub (pinned-by-version): `https://github.com/langchain-ai/langgraphjs/tree/@langchain/langgraph-checkpoint-sqlite@1.0.1` — SqliteSaver source under `libs/checkpoint-sqlite/src/`
- Spike artifact: `packages/solo/spike.mjs` (to be removed after ADR approval; verification script for future reviewers)
- Companion foundations amendment: `docs/foundations.md` §D-21 (harness-identity + protocol-integrity invariant; engine-neutral; landed at line 440)
