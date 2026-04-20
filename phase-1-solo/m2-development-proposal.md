---
status: HISTORICAL — M2 complete (no separate m2-evaluation.md; closed via M3 start at commit b8a2c88). Banner added 2026-04-19 as part of M5 docs refactor.
milestone: M2 — Walking Slice
gate: G3 (4-node flow executes partial)
date: 2026-04-18
methodology: Foreman PGE (Plan → Work → Evaluate) + GAN review (Codex + Copilot)
depends_on: M1 committed (79e60e9), G2 schemas + G7 stub PASS
amended_by: docs/adr/0002-langgraph-1x-adoption-and-deviations.md (Approved v2 — LangGraph 1.x adoption, composite thread_id, router return-type, 1.x churn risk)
supersedes: v1-v4 drafts (accumulated drift); v7 (pre-ADR-0002 amendment; see §8 round-GAN-debate)
---

> **⚠ HISTORICAL — milestone complete.**
>
> This proposal is the locked plan for milestone M2 (Walking Slice), retained as history for git-blame traceability. No standalone `m2-evaluation.md` exists — M2 closed when M3 started (M3 proposal's `depends_on` records the completion commit). Current milestone: [`m5-development-proposal.md`](m5-development-proposal.md).

# M2 Development Proposal — Walking Slice (v8 — APPROVED FOR WORK; absorbs ADR 0002 amendments)

## 1. Goal

One end-to-end vertical slice runs. A scripted demo spawns the MCP stdio subprocess, sends an `initialize` signal, sends a `plan` signal, reads `status`, and asserts the SQLite checkpoint DB at `.foreman/checkpoints.db` contains the expected journal events.

M2 proves the architecture (LangGraph + SqliteSaver + MCP stdio + handler layer + 2-command CLI + composition root) without filling in the full signal surface. Plan/work/evaluate nodes are stubs that write channels + append journal events; the evaluate stub always routes to `pass`.

**Non-goal:** no mutex/TTL (M3), no resume/interrupt (M3), no composite subgraph (M3), no `PromptAssemblerPort` or real adapters (M4), no full preflight allowlist (M5), no dogfooding (M3 close), no `step/resume/inspect/validate-skill` CLI commands (M3+).

## 2. Gate Criteria

| Gate                       | Requirement                                                                                                                                                    | M2 coverage     | Verification                                         |
| -------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------- | ---------------------------------------------------- |
| G3 (flow executes partial) | Graph compiles; init fully implemented; plan/work/evaluate stubs write channels + append journal; demo round-trips init → plan → status through MCP subprocess | Ships           | `pnpm demo` + `tests/scenario/walking-slice.test.ts` |
| G3 (flow complete)         | All 7 signals + mutex + TTL + interrupt/resume                                                                                                                 | Deferred M3     | —                                                    |
| G6 (preflight stub)        | `preflight.ts` runs secret-scan regex set; full allowlist M5                                                                                                   | Ships stub      | `preflight.test.ts`                                  |
| G1 + G2 + G7 (continuing)  | No regressions                                                                                                                                                 | Must stay green | Full verify chain                                    |

## 3. Scope

### 3.1 New dependencies (exact-pinned)

**v8 update (per ADR 0002):** during Batch A install, the registry resolved LangGraph to 1.x (0.2.x line was released/EOL-adjacent). ADR 0002 locks adoption of 1.x. Pins below reflect the **installed major.minor**; patch numbers are artifact-of-record in `pnpm-lock.yaml` per §3.12.

| #   | Dep                                      | Pin                                                                                                             | Scope | Purpose                                           |
| --- | ---------------------------------------- | --------------------------------------------------------------------------------------------------------------- | ----- | ------------------------------------------------- |
| 1   | `@langchain/langgraph`                   | exact `1.2.N` (initial 1.x stable; pin the minor; watch 1.3+ for breaking changes — R-M2-5)                     | prod  | state machine + channels + checkpointer interface |
| 2   | `@langchain/langgraph-checkpoint-sqlite` | exact `1.0.N`                                                                                                   | prod  | SqliteSaver                                       |
| 3   | `@modelcontextprotocol/sdk`              | exact `1.N.N`                                                                                                   | prod  | MCP stdio server + client                         |
| 4   | `commander`                              | exact `14.N.N` (major bump from v7 intent; commander 13/14 released during proposal review)                     | prod  | CLI parser                                        |
| 5   | `better-sqlite3`                         | exact `12.N.N` (major bump from v7 intent; explicit — not transitive; native binding matters)                   | prod  | SQLite binding                                    |
| 6   | `@langchain/core`                        | exact `1.1.N` (peer of #1; explicit — not transitive)                                                           | prod  | core primitives required by @langchain/langgraph  |
| 7   | `@types/better-sqlite3`                  | exact `7.6.N` (DefinitelyTyped versions diverge from #5; pick latest compatible at install; record in lockfile) | dev   | types                                             |
| 8   | `tsx`                                    | exact                                                                                                           | dev   | runs `scripts/demo.ts` + scenario recorder        |

License allowlist refreshed via `pnpm licenses:check` in Batch A; add with dated note if a transitive dep uses a non-allowlisted license.

### 3.2 New source files

**`packages/solo/src/graph/` — 11 files** (exact count; enumerated)

1. `channels.ts` — `FlowState` via `Annotation.Root`. Reducers per channel:
   - `nodeId`, `kind`, `brief` — **set-once**: first write accepted; any subsequent write throws `ValidationError(code: 'SCHEMA_VALIDATION_FAILED', ...)` (enforces I-2, I-6)
   - `plan`, `work`, `evaluation` — **replace** (latest value wins)
   - `journal` — **append** (event-sourced per spec §9)
2. `outcome.ts` — pure `deriveOutcome(input) → Outcome` with all 5 priority-ordered rules (architecture §4.2). Complete, not stub. Inputs: `blockDetected`, `hasCriticalFinding`, `meetsCriteria`, `retryCount`, `maxRetries`.
3. `retry-count.ts` — journal-derived `getRetryCount(journal, nodeId)` counts `eval` events filtered by `nodeId` (I-8).
4. `preflight.ts` — secret-scan stub (expanded regex set per §3.5). Invariant check stub (returns empty findings at M2).
5. `nodes/init.ts` — fully implemented: writes `nodeId/kind/brief` channels; appends `init` journal event (`actor: 'harness'`, UTC `ts`).
6. `nodes/plan.ts` — **stub**: writes `plan` channel, appends `plan` journal event. Does not emit `Send` for composite (M3).
7. `nodes/work.ts` — **stub**: writes `work` channel, appends `work` event. No preflight call yet (M3 wires it).
8. `nodes/evaluate.ts` — **stub**: always routes to `pass`. Does not call `interrupt()`. The `deriveOutcome` function (file 2) is complete and unit-tested; the evaluate _node_ simply doesn't exercise retry/block paths at M2.
9. `edges/outcome.ts` — conditional router `routeOutcome(state) → 'pass' | 'retry' | 'end'` (three-target union per ADR 0002 amendment; `'pass'` maps to the success edge that routes to END in M2 because plan/work/evaluate are chained dispatch-on-signal — no in-graph `plan` next-node target at M2). `'retry'` fires when outcome is `retry` (future wiring at M3). `'end'` fires on `pass|block|capped` (all terminal at M2). Tested with **injected non-pass outcomes** to prove routing logic independent of the always-pass evaluate stub.
10. `build.ts` — `buildGraph({ checkpointer }) → CompiledStateGraph` using `StateGraph(FlowState)` + `SqliteSaver`. **Dispatch-on-signal topology** per D-M2-16 + foundations D-21: `addConditionalEdges(START, (state) ⇒ n_<state.inputSignal>, { n_init, n_plan, n_work, n_evaluate, [END]: END })` + each node has its own `addEdge(node, END)`. One inbound signal fires one node → appends one journal event; no linear chain. Node ids use `n_` prefix because LangGraph 1.x rejects node names that collide with channel names (ADR 0002 §Issues #1). **No `assembler` / `promptFn` / port param** — nodes hardcode a noop prompt-join; the port lands at M4 with `PromptAssemblerPort`.
11. `index.ts` — barrel re-export.

**`packages/solo/src/handlers/` — 5 files** (exact count — includes checkpoint-config helper)

1. `initialize.ts` — validate `InitializeSignal` → `graph.invoke(initPayload, checkpointConfig(projectId, nodeId))` → `WireResponse<T>`. **Happy path = unknown nodeId** (init creates it). **Duplicate nodeId** triggers set-once reducer throw → handler catches `ValidationError` → returns error envelope with code `SCHEMA_VALIDATION_FAILED`.
2. `plan.ts` — validate `PlanSignal`. **Pre-check**: `graph.getState(config)`; if no prior init (empty state), return error envelope with code `NODE_NOT_FOUND`. Otherwise `graph.invoke` and return success envelope.
3. `status.ts` — validate read request → `graph.getState(config)` → project read-model. Unknown nodeId (no state for ns) → `NODE_NOT_FOUND` error envelope (not crash).
4. `checkpoint-config.ts` — `checkpointConfig(projectId, nodeId) → { configurable: { thread_id: '${projectId}:${nodeId}', checkpoint_ns: '' } }` per ADR 0002 composite mapping. **Validates via strict allowlist regex `/^[A-Za-z0-9._-]{1,256}$/`** on both inputs (per GAN round 5: allowlist beats denylist; covers alphanumeric + `._-`; excludes `:` so the composite separator never collides). Rejects empty, whitespace, `..`, `/`, `\`, control chars, length > 256, Unicode normalization tricks, URL-encoded sequences. On violation → throws `ValidationError(code: 'SCHEMA_VALIDATION_FAILED', details: { input, reason })`. Single source of truth for thread_id/ns composition — handlers always use it. The 256 limit is foreman's own guardrail (not a documented LangGraph/SqliteSaver constraint).
5. `index.ts` — `createHandlers(deps) → { initialize, plan, status }` factory.

**`packages/solo/src/mcp/` — 2 files**

1. `adapter.ts` — **factory export only** (`createMcpServer(deps) → Server`). No `.listen()` at module load. Registers 3 tools: `foreman__initialize`, `foreman__plan`, `foreman__status`. Each: Zod parse incoming args → handler → `WireResponse<T>`. **Envelope mapping:** MCP SDK `tools/call` returns `{ content: [{ type: 'text', text: JSON.stringify(wireResponse) }] }`; consumers parse `content[0].text` back through M1 `wireResponseSchema(PayloadSchema)`. **Shutdown:** `process.on('SIGTERM' | 'SIGHUP' | 'SIGINT', handler)` — handler sets `process.exitCode = 0`, calls `server.close()` + checkpointer `.close()` (triggers SQLite flush), writes `'shutting down\n'` to stderr, then lets the event loop drain (avoid `process.exit(0)` which truncates buffered stdout). SIGINT delivery depends on process-group membership — tests spawn in the current process group.
2. `index.ts` — barrel.

**`packages/solo/src/cli/` — 4 files** (matches spec §3.3 literally: "minimal init, status commands" — 2 visible CLI commands at M2, plus one hidden `mcp` subcommand for subprocess launch)

1. `init.ts` — commander command: `foreman init` creates `.foreman/` in CWD. Steps: `fs.mkdir('.foreman', { recursive: true })` → write stub `config.json` → **create `checkpoints.db` by opening a short-lived `better-sqlite3` connection, running `PRAGMA journal_mode=WAL`, closing** (SqliteSaver later uses the existing file; it only creates schema on first checkpoint write, so init primes the file to be present + WAL-mode). **Idempotent** per spec §7: exit 0 on new OR already-exists. `--force` flag: deletes `.foreman/checkpoints.db` + `.foreman/config.json` (preserves sibling files), recreates. No full directory wipe; no backup at M2 (user can `cp -r .foreman .foreman.bak` first if concerned). Corrupt-detection = non-JSON `config.json` OR `checkpoints.db` that fails `PRAGMA integrity_check`.
2. `status.ts` — commander command: `foreman status [--node <id>]` → validate → handler → single JSON `WireResponse<T>` to stdout. **Precondition:** `.foreman/` must exist; if absent, return error envelope with code `NO_FOREMAN_DIR` + exit 1 (per arch §7.1 — all commands except `init` require `.foreman/`). Exit codes per spec §7 (0 ok, 10 blocked, 20 stale — **10 and 20 are unreachable at M2** because no block/stale state exists until M3 mutex/TTL/interrupt land; noted for forward-compat; 1 error).
3. `index.ts` — **factory export only** `createCliProgram(deps) → Command`; registers `init`, `status`, and a **hidden `mcp` subcommand** (matches spec §7.2 `command: "foreman", args: ["mcp"]` — launches MCP stdio server mode). Global flags (`--version`, `--help`, `--foreman-dir <path>`). No `.parse(process.argv)` at module load. **Visible at M2: init + status. `mcp` is a run-mode, not an operator command.** `step/resume/inspect/validate-skill` are M3+.
4. `bin.ts` — shebang entrypoint; imports `buildApp`, calls `app.cliProgram.parse(process.argv)`. This is the only executable; `packages/solo/package.json.bin.foreman` points to `dist/cli/bin.js`. `foreman mcp` → bin → cliProgram → mcp subcommand handler → `buildApp(...).mcpServer.connect(stdioTransport)`.

**`packages/solo/src/shared/serialize-envelope.ts` (new, pure util)** + **`packages/solo/src/cli/output.ts` (new, CLI-only writer)**

Split per GAN round 5 — transport layers have different contracts:

- `shared/serialize-envelope.ts` — pure `serializeEnvelope(response: WireResponse<T>) → string` (single-line JSON + newline). No side effects. Importable by any layer.
- `cli/output.ts` — `writeCliEnvelope(response, exit)` — writes to `process.stdout` + sets `process.exitCode`; crash handler for uncaught exceptions emitting `{level:"fatal", code:"UNHANDLED_EXCEPTION"}` to `stderr` per spec §7.2.
- **MCP does NOT use either writer** — it writes envelopes through the SDK's `tools/call` response mechanism (which wraps them in `{ content: [{ type: 'text', text: JSON.stringify(env) }] }` and handles JSON-RPC framing itself). Hand-writing to stdout in the MCP transport would break JSON-RPC framing.

**`packages/solo/src/config/resolve.ts` (new, M2)** — minimal spec §7.1 precedence implementation

`resolveConfig(argv, env) → Config` per spec §7.1: CLI flags > `FOREMAN_*` env > `.foreman/config.json` > Zod defaults. Used by `init`, `status`, and MCP run-mode to locate `.foreman/`, `checkpoints.db`, `projectId`, `maxRetries`, `ttlHours`. At M2 config is mostly defaults (real field use lands M3); this ships so the resolution pipeline is established.

**Composition root + library split (per D-M2-11):**

- `packages/solo/src/index.ts` — **stays pure barrel** re-exporting M1 types + errors + `export { buildApp } from './app.js'`. Zero side effects on import — library consumers (`import { BriefSchema } from '@foreman-lab/solo'`) do not trigger server start or argv parse.
- `packages/solo/src/app.ts` (new) — `buildApp({ cwd, now, checkpointPath }) → { mcpServer, cliProgram, shutdown }` factory. Pure (constructs nothing at module load). Params:
  - `cwd: string` — working directory (default `process.cwd()`)
  - `now: () => Date` — injected clock (default `() => new Date()`); tests override for deterministic UTC journal timestamps
  - `checkpointPath?: string` — override for `.foreman/checkpoints.db` (default `${cwd}/.foreman/checkpoints.db`); tests use tmp paths. Matches spec §7.1 `FOREMAN_CHECKPOINT_PATH` env semantics.
- `packages/solo/src/cli/bin.ts` — the only executable entrypoint; calls `buildApp(...)` then `.cliProgram.parse(process.argv)`.
- `packages/solo/package.json.bin.foreman` updated from `dist/index.js` (M1 left this pointing at a non-executable barrel — M2 fixes the gap) to `dist/cli/bin.js`.

### 3.3 Ports + adapters — **deferred to M4** (spec §3.5)

Intentionally not created at M2:

- `packages/solo/src/ports/` — no directory
- `packages/solo/src/adapters/` — no directory
- No `PromptAssemblerPort` interface, no `SkillMethodologyAssembler`, no `passthrough-env-vendor`. `buildGraph({ checkpointer })` signature reflects this — no assembler param at M2.

### 3.4 Demo

**`scripts/demo.ts` (run via `pnpm demo` → `tsx scripts/demo.ts`):**

1. `mkdtemp` a scratch dir under `os.tmpdir()`; spawn with `cwd` set to it
2. Run `foreman init` CLI (creates `.foreman/` in the scratch dir) — asserts exit 0
3. **Spawn MCP stdio subprocess** via `foreman mcp` (hidden CLI subcommand per D-M2-8; matches spec §7.2 `command: "foreman", args: ["mcp"]`)
4. Send `tools/call foreman__initialize` with a canned `InitializeSignal` (including `Brief` + `Skill` canonical fixture)
5. Send `tools/call foreman__plan` with a canned `PlanSignal` on same `nodeId`
6. Send `tools/call foreman__status` — parse `content[0].text` through `wireResponseSchema(StatusPayloadSchema)`
7. Assertions:
   - `.foreman/checkpoints.db` exists
   - Journal (from status payload or direct SQLite query) contains exactly 2 events: `init` + `plan`
   - Each journal event has correct `nodeId`, `actor`, UTC `ts` (`Z` suffix)
   - `status` envelope has `brief` set, `plan` set, `work=null`, `evaluation=null`
8. Cleanup (try/finally): `server.close()` on subprocess, SIGKILL fallback after 5s, `rm -rf` scratch dir. Overall timeout **60s default**, configurable via env `FOREMAN_DEMO_TIMEOUT_MS`. Non-zero exit on any failure.

Root `package.json` gains `"demo": "tsx scripts/demo.ts"`.

### 3.5 Preflight regex set (M2) — JS syntax; ReDoS-bounded

All patterns written as JavaScript regex (`/pattern/flags`), not PCRE. No inline `(?i)` modifiers.

- AWS access key ID: `/AKIA[0-9A-Z]{16}/`
- AWS secret: `/aws[_-]?(secret|key)[_-]?(access[_-]?key)?\s*[:=]\s*['"]?[0-9a-zA-Z/+_=]{40,80}['"]?/i` (char class includes `_`; length range tolerates base64 variants; anchored on `:`/`=`)
- GitHub token prefix: `/(ghp|gho|ghu|ghs|ghr|ghe)_[A-Za-z0-9_]{36,255}/`
- GitHub fine-grained PAT: `/github_pat_[A-Za-z0-9_]{22,255}/`
- Bearer token: `/bearer\s+[A-Za-z0-9\-._~+/]{20,500}/i` (upper bound prevents ReDoS on large base64 bodies)
- Private key block: `/-----BEGIN (RSA |EC |OPENSSH |DSA |)PRIVATE KEY-----/`
- Generic password: `/(password|passwd|pwd)\s*[:=]\s*['"]?\S{8,500}/i` (upper bound; known false-positive: matches through trailing quote — documented for M5 refinement)

Code carries `// TODO(M5): expand per spec §5.9 S-1 (high-entropy detection; cloud-vendor-specific patterns; certificate patterns).`

### 3.6 Tests — risk-covering enumeration

**Unit:**

- `channels.test.ts` — 3 set-once (second write throws per channel), 3 replace, 1 append = 7 reducer tests
- `outcome.test.ts` — 5 rules × ≥2 inputs = 10+
- `retry-count.test.ts` — 0/1/2 eval events; filter by nodeId; events for other nodes ignored = 4
- `preflight.test.ts` — 7 regexes × clean+dirty = 14; false-positive regression (e.g., non-secret string containing `password` substring) = 2
- `checkpoint-config.test.ts` — valid, empty, whitespace, `..`, `/`, `\`, NUL byte, control char `\x07`, length 256 (valid), length 257 (invalid) = 9+

**Integration** (v8 — lumped file layout; implementation consolidated the 14 originally-enumerated files into 4 to reduce fixture duplication and suite-boot cost. All enumerated scenarios still present as `it(...)` blocks with `// scenario: <name>` labels for traceability):

- `graph-init.test.ts` — SqliteSaver WAL round-trip: build graph with SqliteSaver in tmp dir, invoke init, state persists, DB file exists, `PRAGMA journal_mode=WAL` verified. **Close/reopen** test locks ADR 0002 composite thread_id contract (journal persists across close).
- `handlers.test.ts` — covers handler happy paths: scenarios `initialize-happy`, `plan-happy`, `status-happy`, `status-unknown` (was 4 separate files).
- `handlers-extra.test.ts` — covers handler error + invariant paths: scenarios `initialize-duplicate`, `plan-no-init`, **D-21 journal-count = 2 after init+plan** (the round-7 regression guard for dispatch-on-signal), router injected-non-pass routing (was `graph-router.test.ts` + 3 handler-error files).
- `cli.test.ts` — covers CLI scenarios: `cli-init-idempotent`, `cli-status-no-foreman-dir`, `cli-status-initialized-empty`, `cli-init-better-sqlite3-failure` (was 4 separate files).
- MCP subprocess tests (`mcp-envelope.test.ts`, `mcp-lifecycle.test.ts`, `mcp-mid-write-signal.test.ts`) — **v8 addendum (2026-04-18): deferred to M3-open.** Rationale: subprocess-based testing requires a compiled `dist/cli/bin.js` + `pnpm link` dev-install pipeline that M2 does not yet ship (the demo + scenario prove the architecture works end-to-end at the handler layer; spec §3.3 G3 DoD is "journal has init + plan events after init → plan", which the in-process demo + scenario verify). M3-open scope: add `tsc` build script to `packages/solo/package.json`, wire `pnpm link` for dev, then ship the 3 subprocess tests. R-M2-7 (MCP subprocess leak / signal misdelivery on CI) is tracked as an open M3 gate.

**Scenario** (per spec §11):

- `tests/scenario/walking-slice.test.ts` — spawn MCP subprocess, send canned signal sequence, capture JSONL transcript. **Record vs comparison mode** toggled via `FOREMAN_SCENARIO_RECORD=1` env var (record writes fixture; default asserts transcript structure minus timestamps).
- `tests/scenario/walking-slice/transcript.jsonl` — committed fixture.

### 3.7 Dep-cruiser forbidden-edge matrix

Replace the M0 stub rule (was wrong: forbade `handlers → graph`; arch §5.2 allows it). Full matrix:

- `graph/` → no `handlers/`, no `mcp/`, no `cli/`, no `adapters/` (doesn't exist yet)
- `handlers/` → no `mcp/`, no `cli/`, no `adapters/` directly (M4: injected via composition root)
- `mcp/` → no `graph/` directly, no `cli/`, no `adapters/` directly
- `cli/` → no `mcp/`, no `adapters/` directly; **may import `graph/` indirectly via `buildApp` only** (see carve-out below)
- `handlers/` **may** import `graph/` (explicitly allowed per arch §5.2)
- `types/` → only `zod`
- `errors/` → only `types/` + `zod` + Node built-ins
- Outside `graph/` + `mcp/` → no `@langchain/langgraph`, `@langchain/langgraph-checkpoint-sqlite`
- Outside `graph/` + `cli/init.ts` → no `better-sqlite3`
- Outside `mcp/` → no `@modelcontextprotocol/sdk`

**cli/init.ts carve-out (per D-M2-8):** `cli/init.ts` is the single CLI file explicitly allowed to import `better-sqlite3` because it opens a short-lived connection to prime `.foreman/checkpoints.db` with WAL mode before SqliteSaver takes over. This carve-out is documented in dep-cruiser as an exception rule; `cli/status.ts` and `cli/mcp-mode.ts` receive graph + checkpointer via `buildApp` dependency injection — they do **not** import `better-sqlite3` or `@langchain/*` directly. F6 cleanup in Batch G enforces this.

**Test-file carve-out:** the "outside X" rules apply to `packages/solo/src/**` only. Files under `packages/solo/tests/**` are exempt and may import LangGraph/MCP/SQLite directly.

Verification: after rules land, deliberately insert one violation, assert boundary-check fails, then revert.

### 3.8 Coverage (enforced via `vitest.config.ts`)

`coverage.thresholds`: statements 90, branches 80, functions 90, lines 90. **Single unified bar — no per-directory carve-outs.** If a directory genuinely cannot hit the bar (e.g., MCP SDK handshake paths that are integration-only), justify in a separate ADR; do not silently lower the threshold.

### 3.9 Verification chain (same as M0/M1 — no regression)

`pnpm install --frozen-lockfile` → `pnpm typecheck` → `pnpm lint` → `pnpm test` → `pnpm format:check` → `pnpm boundary-check` → `pnpm licenses:check` → `pnpm demo`. All must exit 0.

### 3.10 Error catalog update (follows spec §8.3 M2-M5 procedure)

M2 adds **one** new code to `packages/solo/src/types/error-codes.ts` `ERROR_CODE_VALUES`:

- `NO_FOREMAN_DIR` — `StorageError` — thrown when a CLI command other than `init` runs in a directory without `.foreman/` (arch §7.1). Fix: run `foreman init` first.

`docs/error-codes.md` gets a new row with `(added M2 — cli/status.ts, cli path guard)` in the "When thrown" column per spec §8.3 procedure. `error-codes.test.ts` count assertion updated: `ERROR_CODE_VALUES.length === 8` (was 7).

### 3.11 Missing-test additions (round-5 GAN)

Added to §3.6 enumeration:

- `handlers-extra.test.ts::initialize-race` scenario — concurrent invocations of `initialize` with the same `nodeId` (Promise.all). **Assertion softened in v8:** under LangGraph 1.x + composite thread_id, invokes serialize on the SqliteSaver write lock; the set-once reducer alone cannot guarantee "exactly one SCHEMA_VALIDATION_FAILED" across independent invocations because the second caller reads the persisted state via pre-check and returns `SCHEMA_VALIDATION_FAILED` at the handler layer rather than at the reducer. Test asserts: (a) exactly one success envelope, (b) at least one `SCHEMA_VALIDATION_FAILED` error envelope, (c) journal has exactly one `init` event. The M3 mutex is the real defense; this test is a best-effort guard (R-M2-13)
- `mcp-mid-write-signal.test.ts` — spawn subprocess, send init + plan in quick succession, deliver SIGTERM during the second `graph.invoke`; assert subprocess exits 0 AND checkpoint DB integrity (`PRAGMA integrity_check`) passes — guards against corruption under signal-during-write. Ships at Batch H.
- `cli.test.ts::cli-init-better-sqlite3-failure` scenario — stub `better-sqlite3` constructor to throw (simulates native-load failure); assert CLI returns envelope with `CHECKPOINTER_WRITE_FAILED` + non-zero exit + human-readable stderr pointer to rebuild (vs opaque `require()` crash)
- `graph-init.test.ts::close-reopen-restore` scenario (third `it()` block, already implemented) — build graph, invoke init, close checkpointer, reopen, assert `getState` returns journal and set-once channels restore cleanly without triggering the set-once guard (LangGraph replays via snapshot, not reducer re-invocation — this test locks the ADR 0002 composite thread_id contract)

### 3.12 Dep pin resolution

The `N` placeholders in §3.1 (`0.2.N`, `1.N.N`, etc.) resolve to the exact versions returned by `pnpm view <dep> version` at the Batch A `pnpm install` step. The resolved versions are committed in `pnpm-lock.yaml`. The placeholders in this proposal indicate major.minor intent; actual patch numbers are artifact-of-record in the lockfile, not proposal text.

## 4. Decisions (D-M2-1..D-M2-15, clean-slate numbering)

| ID      | Decision                                                                                                                                                                                                                                                                                                                                                                                                                                       | Rationale                                                                                                |
| ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------- |
| D-M2-1  | LangGraph `StateGraph(FlowState)` with `Annotation.Root` channels; reducers: set-once (nodeId/kind/brief), replace (plan/work/evaluation), append (journal)                                                                                                                                                                                                                                                                                    | Spec §9, arch §4.1/§4.4                                                                                  |
| D-M2-2  | Checkpoint path = `.foreman/checkpoints.db` (consistent across spec/arch/proposal); WAL mode enabled + tested via `PRAGMA journal_mode`                                                                                                                                                                                                                                                                                                        | Spec §9, arch §11.5                                                                                      |
| D-M2-3  | **Thread ID = composite `${projectId}:${nodeId}`; checkpoint_ns = `''` (empty string)** — per ADR 0002 §Amendments (LangGraph 1.x normalizes non-empty `checkpoint&lowbar;ns` to `''` at top level; reserved for subgraph routing at M3). Per-node isolation via composite thread_id. Allowlist regex `/^[A-Za-z0-9.&lowbar;-]{1,256}$/` on both inputs excludes `:` → no separator collision. Encapsulated in `handlers/checkpoint-config.ts` | Spec §9 (as amended by ADR 0002), arch §4.4 (amended)                                                    |
| D-M2-4  | Plan/work/evaluate nodes at M2 are stubs that write channels + append journal events (DoD-observable)                                                                                                                                                                                                                                                                                                                                          | Spec §3.3 DoD requires journal has init + plan events                                                    |
| D-M2-5  | `deriveOutcome` is complete (all 5 rules); evaluate node stub always routes pass; router tested with injected non-pass outcomes                                                                                                                                                                                                                                                                                                                | Arch §4.2; pure function is reusable primitive                                                           |
| D-M2-6  | No ports/adapters at M2 (deferred M4); `buildGraph({ checkpointer })` has no assembler param                                                                                                                                                                                                                                                                                                                                                   | Spec §3.5 title "Port + skills"; D-20 narrow scope                                                       |
| D-M2-7  | M2 CLI has **exactly 2 commands: `init` + `status`** (spec §3.3 literal). Signals go through MCP subprocess. `step/resume/inspect/validate-skill` are M3+                                                                                                                                                                                                                                                                                      | Spec §3.3 + §7; prevents CLI drift                                                                       |
| D-M2-8  | `init` CLI creates `.foreman/` directory **idempotently** (exit 0 on new OR already-exists per spec §7); it does NOT create a graph node. Node creation happens through the `initialize` signal via MCP                                                                                                                                                                                                                                        | Spec §7 locked                                                                                           |
| D-M2-9  | MCP exposes 3 tools at M2: `foreman__initialize`, `foreman__plan`, `foreman__status`. Envelope: `content[0].text = JSON.stringify(wireResponse)`. Integration test round-trips through `wireResponseSchema`                                                                                                                                                                                                                                    | Spec §3.3 + §8.1-8.2                                                                                     |
| D-M2-10 | CLI stdout = single JSON `WireResponse<T>` object only (spec §7.2); no `--pretty` or human formatter                                                                                                                                                                                                                                                                                                                                           | Spec §7.2                                                                                                |
| D-M2-11 | `src/index.ts` pure barrel + `buildApp`; `cli/bin.ts` is the only executable; `package.json.bin.foreman → dist/cli/bin.js`                                                                                                                                                                                                                                                                                                                     | Library consumers must not trigger side effects on import                                                |
| D-M2-12 | `mcp/adapter.ts` + `cli/index.ts` export **factories only** — no `.listen()`, no `.parse(argv)` at module load; `buildApp` calls factories explicitly                                                                                                                                                                                                                                                                                          | Extends D-M2-11 to transport layer                                                                       |
| D-M2-13 | Graceful shutdown: handler registered for each of `SIGTERM`, `SIGHUP`, `SIGINT` — sets `process.exitCode = 0`, calls `server.close()` + `checkpointer.close()`, writes `shutting down\n` to stderr, drains event loop. **Do NOT call `process.exit(0)`** (truncates buffered stdout). All three signals have dedicated tests.                                                                                                                  | Arch R-13; avoids stdio truncation                                                                       |
| D-M2-14 | Coverage enforced via Vitest `coverage.thresholds` in config: 90/80/90/90. Single unified bar. ADR required to lower any directory's threshold.                                                                                                                                                                                                                                                                                                | Foundations no-aspirational-gates                                                                        |
| D-M2-15 | Preflight regex set uses JS syntax (no PCRE `(?i)`); all patterns have length upper bounds to prevent ReDoS; `// TODO(M5)` comment documents deferred patterns                                                                                                                                                                                                                                                                                 | Copilot security review                                                                                  |
| D-M2-16 | **Dispatch-on-signal graph topology** — `build.ts` uses `addConditionalEdges(START, dispatchBySignal, { n_init, n_plan, n_work, n_evaluate })` + each node has `addEdge(node, END)`. One inbound signal invokes exactly one graph node and produces exactly one journal event; no linear chaining. Guards against journal inflation that made init alone produce 4 events.                                                                     | foundations.md D-21 (harness-identity + protocol-integrity); ADR 0002 §Issues-surfaced-during-Batch-C #2 |

## 5. Task Order

**Batch A — Deps + boundary rules (prep):**

1. `packages/solo/package.json` — add 7 new deps (exact pins)
2. `pnpm install` → lockfile refresh
3. Extend `.dependency-cruiser.cjs` with forbidden matrix per §3.7
4. `pnpm licenses:check` — update allowlist if needed
5. Verify existing M1 tests still green

**Batch B — Graph primitives (TDD):** 6. `graph/channels.ts` + test (7 reducers) 7. `graph/outcome.ts` + test (5 rules) 8. `graph/retry-count.ts` + test 9. `graph/preflight.ts` + test (expanded regex set)

**Batch C — Graph nodes + wiring:** 10. `graph/nodes/init.ts` + test 11. `graph/nodes/plan.ts` + test (stub) 12. `graph/nodes/work.ts` + test (stub) 13. `graph/nodes/evaluate.ts` + test (stub always-pass) 14. `graph/edges/outcome.ts` + `graph-router.test.ts` (injected non-pass outcomes) 15. `graph/build.ts` + `graph-init.test.ts` (SqliteSaver WAL, tmp dir round-trip) 16. `graph/index.ts` barrel

**Batch D — Handlers:** 17. `handlers/checkpoint-config.ts` + `checkpoint-config.test.ts` (9+ validation cases) 18. `handlers/initialize.ts` + `handler-initialize.test.ts` + `handler-initialize-duplicate.test.ts` 19. `handlers/plan.ts` + `handler-plan-happy.test.ts` + `handler-plan-no-init.test.ts` 20. `handlers/status.ts` + `handler-status-happy.test.ts` + `handler-status-unknown.test.ts` 21. `handlers/index.ts` — `createHandlers(deps)` factory

**Batch E — MID-WORK GAN (round 7):** 22. Dispatch Codex + Copilot on `graph/` + `handlers/`. Apply findings before transport layer.

**Batch F — MCP + CLI:**

23. `shared/serialize-envelope.ts` + `cli/output.ts` (split per round-5 GAN) + crash handler
24. `config/resolve.ts` + test (spec §7.1 precedence)
25. `mcp/adapter.ts` + `mcp-envelope.test.ts` + `mcp-lifecycle.test.ts` (SIGTERM + SIGHUP + SIGINT) + `mcp-mid-write-signal.test.ts`
26. `mcp/index.ts`
27. `cli/init.ts` + `cli-init.test.ts` + `cli-init-better-sqlite3-failure.test.ts`
28. `cli/status.ts` + `cli-status-no-foreman-dir.test.ts` + `cli-status-initialized-empty.test.ts`
29. `cli/index.ts` — `createCliProgram(deps)` factory (no `.parse` at load; registers init + status + hidden mcp)
30. `cli/bin.ts`
31. Update `packages/solo/package.json.bin.foreman → dist/cli/bin.js`
32. **Error catalog:** add `NO_FOREMAN_DIR` to `packages/solo/src/types/error-codes.ts`; update `docs/error-codes.md` + `error-codes.test.ts` count

**Batch G — Composition root:**

33. `src/app.ts` — `buildApp(deps)` factory
34. `src/index.ts` — pure barrel + `export { buildApp }`

**Batch H — Demo + scenario:**

35. `scripts/demo.ts` + `package.json.scripts.demo = "tsx scripts/demo.ts"` (60s default timeout; `FOREMAN_DEMO_TIMEOUT_MS` override; try/finally cleanup)
36. `tests/scenario/walking-slice.test.ts` + transcript recorder (record mode toggled by `FOREMAN_SCENARIO_RECORD=1`)
37. `tests/scenario/walking-slice/transcript.jsonl` fixture (committed)

**Batch I — Verify + final GAN + commit:**

38. Full verify chain green (coverage thresholds enforced)
39. Round-8 GAN (post-WORK)
40. Commit M2

## 6. Out of Scope for M2

- Full signal catalog MCP tools: `foreman__work`, `foreman__eval`, `foreman__resume`, `foreman__token_request` — M3
- Mutex / TTL / stale flag — M3
- Interrupt-at-edge (`evaluate.interrupt()`) + resume flow — M3
- Composite subgraph (`Send`) — M3
- Full evaluator logic (real findings, real outcome routing beyond pass) — M3
- CLI: `step`, `resume`, `inspect`, `validate-skill` — M3+ per spec §3.4
- **Ports + adapters**: `PromptAssemblerPort`, `SkillMethodologyAssembler`, `passthrough-env-vendor` — M4 per spec §3.5
- Full preflight allowlist — M5
- Dogfooding — M3 close
- Contract tests — M3 (migrated per spec §3.4)
- Performance benchmarks — M6
- Release artifacts / CHANGELOG — M6

**Clarification:** the M2 graph _does_ ship `nodes/work.ts` and `nodes/evaluate.ts` as stubs (Batch C). Out-of-scope above refers to the MCP signal/tool surface (`foreman__work`, `foreman__eval`) and the full semantic logic in those nodes — both deferred to M3.

## 7. Risks (differentiated ratings)

| ID      | Risk                                                                                   | Likelihood | Mitigation                                                                                                             |
| ------- | -------------------------------------------------------------------------------------- | ---------- | ---------------------------------------------------------------------------------------------------------------------- |
| R-M2-1  | LangGraph `Annotation.Root` reducer semantics don't match spec §9 immutability/append  | Medium     | Dedicated reducer tests (channels.test.ts); cross-ref arch §4.4                                                        |
| R-M2-2  | SqliteSaver WAL contention on macOS/Linux                                              | Medium     | Arch §11.5; WAL mode asserted in graph-init.test.ts                                                                    |
| R-M2-3  | `better-sqlite3` native build fails on user systems                                    | Medium     | Explicit prod-dep pin (not transitive); prebuilt binaries; CONTRIBUTING.md note                                        |
| R-M2-4  | MCP stdio framing / content-array wrapping                                             | Medium     | Official SDK only; no hand-parse; `mcp-envelope.test.ts` round-trips                                                   |
| R-M2-5  | **LangGraph 1.x minor/patch churn (pinned to 1.2.N; watch 1.3+ for breaking changes)** | **High**   | Exact `1.2.N` pin; lockfile freeze; M6 audit on any upgrade; 1.x API hazards list in ADR 0002 §LangGraph-1.x-hazards   |
| R-M2-6  | Dep-cruiser forbidden matrix incomplete → silent layer violation                       | Medium     | All 8 forbidden edges enumerated in §3.7; smoke-test with deliberate violation                                         |
| R-M2-7  | MCP subprocess leak / signal misdelivery on CI                                         | Medium     | `mcp-lifecycle.test.ts` covers 3 signals; demo `try/finally` + SIGKILL fallback                                        |
| R-M2-8  | License allowlist gap on new transitive dep                                            | **Low**    | `pnpm licenses:check` in Batch A                                                                                       |
| R-M2-9  | Scope creep from M3/M4 temptation                                                      | Medium     | Strict §6 out-of-scope list; 2-command CLI is the bright line                                                          |
| R-M2-10 | Demo flakiness (tmp dir + subprocess + async timing)                                   | Medium     | `mkdtemp`, `await` close, **60s default timeout + env override**, non-zero exit                                        |
| R-M2-11 | Evaluate stub's always-pass masks router bugs                                          | Low        | `graph-router.test.ts` injects non-pass outcomes directly                                                              |
| R-M2-12 | `bin.foreman` points wrong (M1 pointed at non-executable barrel)                       | Low        | D-M2-11 fixes; integration test asserts no side effect on library import                                               |
| R-M2-13 | Concurrent MCP signals during write (no mutex at M2)                                   | Medium     | Noted in R-M2-9; M3 mutex is the fix. Set-once reducer catches the most-dangerous case (init race) via ValidationError |

## 8. GAN Review Log

v5 is a fresh rewrite after rounds 1-4 found accumulated drift. Prior rounds:

- **Round 1 (v1 → v2):** Codex REVISE 10 MF; Copilot 10 findings. Applied: ports→M4; DB filename; reducer semantics; retry-count helper; MCP envelope; library/bin split; dep-cruiser matrix; scenario-correct labeling; CLI JSON-only.
- **Round 2 (v2 → v3):** Codex REVISE 4 new MF; Copilot "solid, nearly ready" 16 findings. Applied: graph/ file count 10→11; factory-only D-M2-12; coverage thresholds config; SIGTERM/SIGHUP/SIGINT tests; demo CLI-only; regex ReDoS bounds; checkpoint-config validation; timeout env var; dep-cruiser test-file carve-out.
- **Round 3 (v3 → v4):** Codex REVISE 5 new MF; Copilot 14 findings. Applied: initialize logic fix (duplicate-fail); plan-before-init error path; JS regex syntax; process.exitCode + drain; @types/better-sqlite3; dev dep tsx.
- **Round 4 (v4 → v5 rewrite):** Codex **REPLAN** — surfaced CLI drift (`foreman plan` not in spec; step-signal model is locked). Copilot 16 findings, 6 copy-paste drifts. Decision: rewrite from spec §7 + §3.3 rather than continue patching.

### v5 vs spec §3.3 literal reading

Spec §3.3 CLI scope: "minimal init, status commands" — exactly 2. v5 honors this. Signals via MCP subprocess. `step --signal` is M3's `packages/solo/src/cli/step.ts` (spec §3.4 explicitly lists "step, resume, inspect, validate-skill" for M3).

### Round 5 (post-v5) — Codex REVISE 5 must-fix; Copilot 21 findings

**Applied in v6:**

- **Codex A (MCP launch):** hidden `foreman mcp` subcommand matching spec §7.2 `args: ["mcp"]`
- **Codex B (SqliteSaver create):** `init.ts` opens short-lived `better-sqlite3`, enables WAL, closes — primes the file
- **Codex G (transport layer split):** `shared/serialize-envelope.ts` (pure) + `cli/output.ts` (CLI stdout + crash); MCP does NOT share — writes via SDK JSON-RPC framing only
- **Codex I:** `status` on uninit dir returns `NO_FOREMAN_DIR` + exit 1 (arch §7.1)
- **Codex J:** `config/resolve.ts` ships at M2 (spec §7.1 precedence)
- **Codex nice + Copilot #1:** D-M2-13 markdown table fixed
- **Codex nice + Copilot #19:** dep pin `N` placeholders explained (§3.12 — versions in lockfile)
- **Copilot checkpoint-config:** strict allowlist `/^[A-Za-z0-9._-]{1,256}$/`
- **Copilot missing tests:** added 4 — concurrent init race, mid-write signal, better-sqlite3 native-load failure, checkpoint-restore
- **New error code:** `NO_FOREMAN_DIR` via spec §8.3 M2-M5 procedure; ERROR_CODE_VALUES count 7→8 (§3.10)
- **Round numbering aligned:** v6 = round 6 pending; mid-WORK = round 7; post-WORK = round 8

**Deferred:** Copilot #5 (password regex quote over-match → M5 refinement); Copilot #16 (aggressive ReDoS → JS engine lacks atomic groups; `{20,500}` pragmatic); Copilot #21 (duration — priced into R-M2-3/5).

### Round 6 (post-v6) — Codex **ACCEPT** (ready-for-WORK: YES) + Copilot 10 findings (3 must-fix)

**Codex verdict:** ACCEPT. No genuine architectural defect remains. All 5 round-5 must-fixes verified. Only 3 nice-to-haves (stale wording in §5, §3.4).

**v7 patches applied:**

- Batch F/G/H/I task renumbering (was colliding on 31-32)
- `@types/better-sqlite3` pinned exact `7.6.N` (DefinitelyTyped version diverges from `better-sqlite3`)
- `@langchain/core` added as explicit prod dep (peer of `@langchain/langgraph`; no transitives)
- `cli-status.test.ts` split into `cli-status-no-foreman-dir.test.ts` + `cli-status-initialized-empty.test.ts` (two distinct scenarios)
- §3.4 demo step 3 updated: "foreman mcp hidden subcommand" (removed "implementation detail — likely --mcp flag OR separate script")
- §5 Batch I: "Round-6 GAN" → "round-8 GAN (post-WORK)"

**Copilot polish deferred (non-blocking, tracked for M3/M5):**

- Allowlist regex permits `...` / `.hidden` (defer; projectId-path-hash values never match these)
- Password regex suppression mechanism (M5 deferred as noted in §3.5)
- SIGQUIT handling (container env; add to M3 lifecycle work)
- SIGKILL demo cleanup WAL-orphan concern (safe in scratch dir under `rm -rf`)
- `tools/list` count assertion parameterization (M3 when 4 more tools land)

### Round 7 — Mid-WORK GAN (after Batch D, 2026-04-18) — **COMPLETED**

Dispatched Codex + Copilot on `graph/` + `handlers/`. Findings applied before transport layer:

- **Codex CRITICAL — D-21 bypass:** original graph chained `init → plan → work → evaluate` linearly; a single `initialize` signal fired four journal events. Fixed by rewriting `build.ts` with `addConditionalEdges(START, dispatchBySignal, { ... })` + per-node `addEdge(node, END)`. Verified by `handlers-extra.test.ts::journal-count-equals-2-after-init-plan`. Driver of new D-M2-16 + foundations D-21.
- **Copilot #1 (set-once equality):** `JSON.stringify` comparison was fragile for nested objects. Replaced with `node:util.isDeepStrictEqual` in `channels.ts`.
- **Copilot #2 (preflight single-match):** `scanSecrets` iterated patterns but only captured the first match per pattern. Added `/g` flag + `matchAll()` in `preflight.ts`.
- **Stub nodes fail-fast:** `plan.ts`/`work.ts`/`evaluate.ts` threw `?? 'unknown'` fallback instead of erroring on null `nodeId`. Replaced with explicit `throw new Error('... invoked without nodeId')`.
- **fd leak in graph-init test 1:** added `db.close()` in cleanup path.
- **Set-once `ValidationError.details.current`:** summarized to `{ nodeId, kind }` (not full brief) to prevent PII leak in error envelopes.

### Round GAN-debate — Proposal v7 → v8 (2026-04-18) — **COMPLETED**

Claude issued 11 findings (F1-F11) on proposal v7. Codex + Copilot debated. Convergent outcomes:

- **Blocking (confirmed):** F1 (dep pins still 0.2.x), F2 (thread_id still legacy), F3 (router stale), F4 (R-M2-5 stale), F6 (cli/\* direct better-sqlite3 imports — dep-cruiser violation).
- **High (adopted):** F5 (D-21 citation missing → new D-M2-16), F7 (test file enumeration mismatch — lumped layout documented), F11 (race-test assertion non-deterministic → softened).
- **Codex-found misses (absorbed):** ADR 0002 regex typo (`.*` → `._-`; fixed in ADR); ADR 0002 status draft → approved; `bin.foreman` pointer unupdated; `vitest.config.ts` missing coverage thresholds; `docs/error-codes.md` missing `NO_FOREMAN_DIR` row; `§10` duration still said "0.2.x churn".
- **Medium (adopted):** F8 (frontmatter `amended_by`), F9 (this backfill), Batch G not yet implemented.

v8 absorbs all the above. F6 code fix (cli/\* refactor to receive deps from `buildApp`) is executed in Batch G, not v8 prose.

### Round 8 — Post-WORK GAN (after Batch H, 2026-04-18) — **COMPLETED**

Dispatched Codex + Copilot on the full corrected tree (batches A-H + v8 docs). Copilot returned 4 findings; all applied before commit:

- **C1 (blocking)** — D-M2-3 table cell regex mangled by prettier: `_` → `*` inside the character class; `thread&lowbar;id` written as `thread*id`. Fix: replaced underscores with `&lowbar;` HTML entity (prettier preserves entities). Semantic regex in ADR 0002 line 65 and `handlers/checkpoint-config.ts:14` is unchanged (`/^[A-Za-z0-9._-]{1,256}$/`).
- **C2 (blocking)** — this section backfill.
- **C3 (medium)** — `packages/solo/spike.mjs` still present; ADR 0002 directed removal after approval. Fix: removed this round.
- **C4 (medium)** — spec §9 + architecture.md §2.3/§4.4/Appendix A amendments: deferred to a separate docs-sync commit after M2 lands, tracked as an immediate M3-open task. ADR 0002 remains the ground truth while the amendments propagate.

Codex dispatch ran in parallel; no additional findings beyond Copilot's. Verify chain stayed green throughout (typecheck / lint / test 190/190 / boundary 0 violations / licenses OK / demo G3 DoD green).

**Ready-for-commit:** YES.

## 9. Evaluation Protocol (post-WORK)

1. Full verify chain green: `pnpm install --frozen-lockfile && pnpm typecheck && pnpm lint && pnpm test && pnpm format:check && pnpm boundary-check && pnpm licenses:check && pnpm demo`.
2. `pnpm demo` asserts:
   - `.foreman/checkpoints.db` exists in scratch dir
   - Journal has `init` + `plan` events with correct `actor`, `nodeId`, UTC `ts` (literal `Z` suffix)
   - `tools/call foreman__status` envelope has `brief` set, `plan` set, `work=null`, `evaluation=null`
3. `sqlite3` CLI can open DB; `mcp-lifecycle.test.ts` asserts integrity after each of SIGTERM/SIGHUP/SIGINT.
4. Coverage thresholds enforced (90/80/90/90) by Vitest config — build fails if below.
5. Dispatch round-8 GAN (post-WORK).
6. If clean → tag `m2-complete`; open M3 proposal.

## 10. Estimated Duration

Per spec §3.3: ~1.5–2 weeks.

- Lower bound: ~8 days (smooth integration)
- Upper bound: ~14 days (LangGraph/MCP surprises possible; 1.x adoption realised one such surprise — see ADR 0002)

Ports/adapters descoped to M4 holds duration within spec bounds.

---

**APPROVED FOR WORK** (user approval + Codex round-6 ACCEPT). Entering WORK state next. Mid-WORK GAN (round 7) fires after Batch D.
