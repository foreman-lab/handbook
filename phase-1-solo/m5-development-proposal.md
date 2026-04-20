---
status: vN Locked 2026-04-19 by operator sign-off — authored 2026-04-19 post-M4-pivot; 9 Codex GAN rounds applied (R1-R6: early drift + schema/naming convergence; R7 deep adversarial + signal-matrix + journal schema + ADR 0008 min-content; R8 testability lens with exact test filenames per bullet + MANUAL markers; R9 factual audit — I-2/I-6 → I-1 invariant number + error-count math +1=11 + ADR 0008 forward-reference). Batch A starts next.
milestone: M5 — Domain-blind ship milestone (external rendering + 2 playbooks + scenario suite + preflight + traceability)
gate: closes G4 (scenario suite) + G9 (2-playbook target); maintains G2 (2-tool MCP); partial G6 (S-2..S-5); G7 audit defers to M6
date: 2026-04-19
methodology: Foreman PGE (Plan → Work → Evaluate) + GAN review (Codex + Copilot) through hexagonal-architecture lens + domain-blind-core lens
depends_on: M4-pivot close (working tree has 14 staged deletions reverting PromptAssemblerPort; M4-alpha must tag against the post-pivot baseline before M5 Batch A starts)
supersedes: (no predecessor skeleton at this tier); absorbs un-shipped M4 remainder (inspect/validate-playbook/example-playbook/agent-setup/dogfooding-log) and sprint-8 intent (wrappers verification + traceability + remediation) under current M-naming
grounding: spec §3.5 M5/M6 split; foundations D-17/D-18/D-20/D-21 + S-1..S-5; architecture.md §3 (post-pivot); ADR 0002/0003/0004/0005; roadmap Gates 4, 6, 9; M4 proposal §10 non-goals items 1/2/7
---

# M5 Development Proposal — Domain-Blind Ship (v0.1 draft — 2026-04-19)

**Authoring context (read before critiquing):** this is the first milestone proposal written entirely under the **domain-blind state-machine** model (operator reality-check 2026-04-18: "foreman is infrastructure; skills are external; rendering happens outside foreman"). M4 locked its plan on 2026-04-18 with `PromptAssemblerPort` as central; mid-execution the operator pivoted. The 14-file `PromptAssembler*` revert is staged but not yet committed, and the M4-alpha tag is not cut. **M5 assumes M4 ships in its post-pivot form** (no runtime rendering adapter; `inspect`/`validate-playbook` CLI + `agent-setup.md` + `tdd-feature` example playbook all deferred to M5 since they were M4 secondary scope that didn't land before the pivot). If M4-alpha tags with any of those still in scope, M5 Batch A1 shrinks accordingly.

## 1. Goal

M5 is the **domain-blind ship milestone** before the M6 v0.1 release. It delivers:

1. **Externalized prompt rendering validated** — an external agent (Claude Code, Copilot CLI, Cursor, or equivalent) reads raw `brief.playbook.phases` from the step envelope, renders prompts in the agent layer, and submits signals back to foreman. Foreman's role is storage + dispatch; no runtime interpolation. (`copilot-plugin/` in the worktree is unrelated workspace tooling — see §11 Q-M5-1 resolution.)
2. **Two example playbooks shipped** — `tdd-feature` (terminal) + `refactor-extract` (composite) under `examples/`, proving G9 and validating the kind-blind playbook contract across two genuinely different methodologies.
3. **Scenario test suite (G4)** — end-to-end scenarios exercising realistic multi-playbook flows: terminal plan→work→evaluate, composite with children, block/unblock/fail paths, recovery from dirty shutdown.
4. **Preflight allowlist materialized** — `playbook.scope.paths` + `playbook.scope.readOnly` (M3-shipped casing — boolean flag, not per-path list) enforced at signal-validation time, replacing the no-op M3-era stub.
5. **Architecture traceability doc** — every active invariant mapped to at least one automated test. Active set: docs/architecture.md §1.6 I-1..I-9 + I-12 + I-13 + M5-introduced I-18 (agent-sole-renderer) + I-19 (allowlist pure) + I-20 (traceability self-check) + disciplines D-5/D-17/D-18/D-20/D-21 + security S-1..S-5. Post-2026-04-18 pivot retired I-10/I-11/I-14/I-15/I-16/I-17 (all assembler-scoped — see ADR 0007). Handler-level composite kind branching is authorized by ADR 0004, not by an invariant.
6. **CLI inspect + validate-playbook** (carried from M4) — operator debugging + playbook authoring tools.
7. **Integrity sweep** — grep-based stale-concept detection (`blockedFrom`, `PromptAssembler*`, `prompt-view`, `nextPrompt`, `view-mapping`) returning zero unexplained hits post-pivot.

**Non-goal (locked):** see §10. Explicitly deferred: README / v0.1 tarball packaging / changelog / perf benchmarks / daemon / composite playbook authoring guide — all M6.

## 2. Gates

| Gate | Closes fully? | Partial? | Notes |
|------|---------------|----------|-------|
| G2   | Maintained    | —        | 2-tool MCP surface preserved; new `contract/mcp-tool-count.test.ts` from M4 carries forward |
| G4   | Yes           | —        | Scenario suite lands: ≥8 scenarios covering terminal, composite, block/unblock/fail, dirty-shutdown recovery, composite two-playbook isolation |
| G6   | No            | Yes      | S-2..S-4 materialized (evaluator readOnly, completion-requires-evaluation, composition evaluation); S-5 partial (production obligation asserted at work-signal validation); S-1 "no ambient credentials" stays at M3/M4 stub level (dropped per M4 pivot `R-M4-11` / Batch C1 deferral); full S-1 materialization waits for Phase 2a Vault/HSM |
| G7   | No            | Audit    | Full error-catalog audit with `details` schema per code defers to M6; M5 limits itself to the existing M4 codes minus `PROMPT_ASSEMBLY_FAILED` (pivot-reverted) |
| G9   | Yes           | —        | 2-playbook target met: `tdd-feature` + `refactor-extract` + `agent-setup.md` + one full playbook-driven dogfooding run per playbook |

**Test-count delta:** M4-post-pivot baseline projected ~280 passing (M3 270 + M4-non-pivot batches ~10). M5 adds ~55 (2-playbook fixtures + scenario suite 8 scenarios + preflight allowlist unit/integration + traceability snapshot + inspect/validate-playbook CLI exit-code matrix + integrity-sweep runner test). Target: ~335 passing at M5-alpha.

## 3. Scope

### 3.1 Dependency changes

- **Default:** no new npm deps. YAML parser (`yaml@2.x`) pulled in by `validate-playbook` (carried from M4 Batch F).
- **Test-time only:** none. Vitest 2.1.x already in place.

### 3.2 Runtime / authoring / rendering split (post-pivot lock)

Three distinct layers, each with one responsibility:

| Layer     | Responsibility                           | What it sees                    | What it never does                         |
|-----------|------------------------------------------|---------------------------------|--------------------------------------------|
| Foreman runtime | dispatch signals; maintain state machine | parsed `Playbook` object in state  | render prompts; interpret methodology text |
| Wrapper / CLI   | validate + parse YAML → `Playbook`       | raw YAML / TS modules           | touch runtime; render prompts              |
| Agent layer     | render prompts from playbook phases    | `brief.playbook` in step envelope  | mutate state directly; bypass MCP tools    |

**Key invariant (new at M5, codified in ADR 0007):** the step envelope carries playbook content verbatim; the agent is solely responsible for turning `phases.{phase}` + state context into a prompt string. Foreman emits no `nextPrompt`, no `assembledPrompt`, no projection — it emits the raw playbook and the current state.

### 3.3 File enumeration

**New files (M5):**

- `examples/tdd-feature/playbook.yaml` — terminal playbook (kind-blind phase content; no `{{var}}` tokens required since foreman no longer interpolates; authors may use `${...}` or Mustache-style placeholders that the agent layer resolves)
- `examples/tdd-feature/README.md` — 1-page playbook description + sample operator workflow
- `examples/refactor-extract/playbook.yaml` — composite playbook (phases.work is a consolidation prompt)
- `examples/refactor-extract/README.md` — same shape as tdd-feature
- `packages/solo/src/cli/inspect.ts` — `foreman inspect [--node <id>] [--pretty]` (M4 Batch E deliverable, carried forward)
- `packages/solo/src/cli/validate-playbook.ts` — `foreman validate-playbook <path>` (M4 Batch F deliverable, carried forward; no `PromptAssemblerPort` dependency — unchanged post-pivot)
- `packages/solo/src/allowlist/check.ts` — scope-path + readonly enforcement; pure function `checkAllowlist(signal, scope | undefined): AllowlistResult` (directory named `allowlist/` not `preflight/` to avoid collision with existing `packages/solo/src/graph/preflight.ts` secret-scanner — per Codex GAN Round 7 F6)
- `packages/solo/src/allowlist/index.ts` — barrel
- `docs/agent-setup.md` — MCP config snippets for Claude Code / Copilot CLI / Cursor + **an explicit section on externalized rendering**: how the agent reads `methodology` from `foreman__status` output and constructs prompts
- `docs/phase-1-solo/architecture-traceability.md` — single-table mapping: `Invariant/Discipline → Test(s) → Module(s) → Status`. Every I-*, D-*, S-* row; gaps flagged
- `docs/adr/0007-domain-blind-core.md` — ADR codifying the 2026-04-18 pivot; invalidates the not-yet-written ADR 0006 (prompt-assembler-port); references the 14-file revert as its artifact
- `docs/adr/0008-scope-allowlist.md` — scope-path enforcement contract. **Minimum content (locked at v0.7 after Codex GAN Round 7 F4):** (1) severity table — `out_of_scope` = hard-block; `readonly_violation` + write intent = hard-block; (2) signal → (path, intent) extraction matrix (per §3.7); (3) missing/optional scope semantics (`scope === undefined` = permissive skip; `scope.paths === []` = deny-all); (4) pathless signal skip policy (`initialize`/`plan`/`eval`/`resume`/`retry`/`token_request`/`status` skipped; only `work` extracts paths); (5) journal event shape for category `'preflight'`; (6) recovery path (agent must resubmit with in-scope path OR operator must update `playbook.scope` + re-initialize a new node — `brief` is immutable-after-init per I-1)
- `packages/solo/tests/e2e/` (new directory) — scenario test suite:
  - `scenario-01-terminal-tdd.test.ts` — full tdd-feature playbook terminal flow
  - `scenario-02-composite-refactor.test.ts` — composite refactor-extract with ≥2 children
  - `scenario-03-block-unblock-approve.test.ts` — evaluator block → operator approve
  - `scenario-04-block-unblock-override.test.ts` — evaluator block → operator override
  - `scenario-05-block-fail.test.ts` — evaluator block → operator fail → END routing
  - `scenario-06-recovery-dirty-shutdown.test.ts` — simulate dirty shutdown → rehydrate from file-backed checkpointer → resume. **NOT `:memory:` — the recovery claim requires a persistent DB that survives process termination; uses a scratch tmp-dir SQLite file instead, accepted as slower per Batch D timing.**
  - `scenario-07-composite-two-playbook.test.ts` — composite parent with 2 children, each child initialized with a distinct `brief.playbook` (set-once per I-1 of docs/architecture.md §1.6 ("Brief is immutable after init")). Proves each child's phase content is carried through its own thread without cross-contamination. **Not brief mutation** — changing an initialized node's playbook is an ADR-scope change (requires amending the set-once invariant).
  - `scenario-08-preflight-rejection.test.ts` — signal referencing out-of-scope path → `PREFLIGHT_BLOCKED` error
- `packages/solo/tests/unit/allowlist-check.test.ts` — scope-path matcher + readonly enforcement + edge cases (glob, parent-traversal, empty `scope.paths`, missing `scope`, pathless-signal skip) + N=100 purity harness per I-19. Symlink handling is out-of-scope and explicitly tested-as-skipped.
- `packages/solo/tests/integration/cli-inspect.test.ts` — JSON + `--pretty` modes + `--foreman-dir` honor
- `packages/solo/tests/integration/cli-validate-playbook.test.ts` — valid / malformed / missing-file exit-code matrix + `WireResponse<Playbook>` stdout contract
- `packages/solo/tests/contract/traceability-snapshot.test.ts` — asserts `docs/phase-1-solo/architecture-traceability.md` has a row for every I-*, D-*, S-* declared in `docs/foundations.md` (grep-and-match; fails if a new invariant lands without a traceability row)
- `scripts/integrity-sweep.sh` — shell runner for the stale-concept grep set; exit 0 iff all patterns either (a) return zero hits or (b) match a documented carve-out in `docs/phase-1-solo/integrity-sweep-carveouts.md`
- `docs/phase-1-solo/integrity-sweep-carveouts.md` — whitelist of intentional hits with rationale (e.g., ADR references to retired concepts)

**Updated files (M5):**

- `packages/solo/src/types/error-codes.ts` — **+1**: add `PREFLIGHT_BLOCKED` for scope violations. Rewrite leading JSDoc count 10 → 11 (M5 net +1 on top of post-M4 baseline of 10; `PROMPT_ASSEMBLY_FAILED` was removed at M4-alpha and does not return).
- `packages/solo/src/handlers/step.ts` — call `checkAllowlist(signal, state.brief?.playbook?.scope)` once at dispatch top (after Zod parse, before any write); return `PREFLIGHT_BLOCKED` `WireResponse` on `{ ok: false }`, emit journal event on the same path
- `packages/solo/src/types/journal.ts` — add `'preflight'` to the block-event category enum so `PREFLIGHT_BLOCKED` violations can be journaled (per Codex GAN Round 7 F3 — event schema update was missing from prior file-list)
- `packages/solo/src/graph/nodes/{plan,work,evaluate}.ts` — **no change from post-pivot baseline**; nodes stay pure state reducers
- `packages/solo/src/cli/index.ts` — register `inspect` + `validate-playbook` subcommands (carried from M4)
- `docs/phase-1-solo/architecture.md` — §3 rewritten for post-pivot: drop all `PromptAssemblerPort` / `PromptView` / `view-mapping` references; add §3.5 "Agent-rendered prompts (externalized)" subsection
- `docs/phase-1-solo/spec.md` — §3.5 updated: split M4 (pivot-shrunk) vs M5 (this proposal) vs M6 (release) scope tables; `token_request` row stays at M3 stub level
- `docs/dogfooding-protocol.md` — §12 updated: new deferrals for M6; `inspect` + `validate-playbook` promoted from "deferred" to "shipped at M5"
- `docs/dogfooding-log.md` — append 2 playbook-driven runs (one per example playbook) as Batch G deliverable
- `docs/foundations.md` — no change if S-* invariants unchanged; review required in Batch F
- `docs/phase-1-solo/m4-development-proposal.md` — status header updated to reflect pivot (Batch I of M5, not M4 — M4 docs stay frozen as history; a pointer line added at the top: "Superseded in part by M5 proposal after 2026-04-18 domain-blind pivot")
- `.dependency-cruiser.cjs` — lift the time-boxed `no-orphans` exclusion for `ports/**` (no ports directory any more); add a `no-prompt-assembler-resurrection` sentinel rule banning any file path matching `prompt-assembler|view-mapping|prompt-view` under `packages/solo/src/`

### 3.4 Error-code deltas

**`PREFLIGHT_BLOCKED`** (new)
- **Location:** thrown (returned as `WireResponse.error`) by `handlers/step.ts` when `checkPreflight` rejects
- **Trigger:** signal references a path not permitted by `playbook.scope.paths`, or attempts a write intent while `playbook.scope.readOnly === true`. **No schema change to `ScopeSchema` — uses the M3-shipped `{ readOnly: boolean, paths: string[] }` shape at `packages/solo/src/types/playbook.ts:13-18`.** If future work wants per-path readonly granularity, that's a schema evolution via ADR.
- **`details` schema:** `{ path: string, reason: 'out_of_scope' | 'readonly_violation', allowed: readonly string[] }`
- **Remediation hint:** "Signal rejected by preflight: path `<path>` violates playbook scope (`<reason>`). Update `playbook.scope.paths` / `playbook.scope.readOnly` or use a different path."

**Explicit removals:** none at M5 (`PROMPT_ASSEMBLY_FAILED` was already removed at M4-alpha commit `64dd599` — baseline at M5 start is 10 codes).

**Net delta at M5:** +1 (PREFLIGHT_BLOCKED). Final count at M5-alpha: **11** (was 10 at M4-alpha baseline). Prior Codex GAN Round 9 F1 caught the earlier "+1 −1 = net-zero = 10" claim as wrong math — the −1 already happened at M4, not here.

### 3.5 Dep-cruiser rule updates

The 6 M4 hexagonal rules are reduced to the ones that still make sense post-pivot:

| M4 rule | M5 disposition | Rationale |
|---------|----------------|-----------|
| `ports-only-types-and-zod` | **Retired** | No ports/ directory post-pivot |
| `adapters-no-transport-or-handlers` | **Retired** | No adapters/ directory post-pivot |
| `handlers-no-adapters` | **Retired** | No adapters/ |
| `handlers-no-ports` | **Retired** | No ports/ |
| `graph-no-adapters` | **Retired** | No adapters/ |
| `composition-root-isolation` | **Relaxed** | `app.ts` no longer crosses ports↔adapters; rule narrows to "app.ts is the sole site that instantiates `openSqliteCheckpointer` + `buildGraph` + `createHandlers` + `createMcpServer` simultaneously" |

**New rules at M5:**

- `allowlist-pure-imports` — `packages/solo/src/allowlist/**` may only depend on `types/`, `errors/`, and `zod` (matches the §3.6 layer table). Enforced by dep-cruiser path rule.
- `allowlist-no-node-builtins` — `packages/solo/src/allowlist/**` must NOT import `node:fs`, `node:path`, `node:os`, `node:crypto`, or `node:process`. Enforced by dep-cruiser. Complementary ESLint `no-restricted-imports` rule covers the legacy un-prefixed spellings (`fs`, `path`, etc.). **Limit:** neither tool can ban direct use of globals (`Date.now()`, `Math.random()`, `process.env` if imported elsewhere); those are caught only by the purity unit-test harness that runs `checkAllowlist` with a determinism probe (identical input → identical output across N=100 invocations with no side-effects observed by a sentinel spy).
- `no-prompt-assembler-resurrection` — sentinel: any file matching `prompt-assembler|view-mapping|prompt-view` regex fails the build.
- `no-renderer-function-resurrection` — **new** per Codex GAN Round 7 F7: grep-based contract test (`tests/contract/agent-rendering-boundary.test.ts`) scans all `packages/solo/src/` `.ts` files for function/method names matching `/render.*[Pp]rompt|assemble.*[Pp]rompt|project.*[Vv]iew|buildPrompt|composePrompt/` — any match fails. Catches function-level resurrection that the filename sentinel misses (e.g. a `renderPrompt` helper snuck into `handlers/step.ts`).

**I-13 enforcement post-pivot:** the composition-root contract test migrates from "sole ports↔adapters crosser" to "sole instantiator of framework-bound entities". The test file is renamed `tests/contract/composition-root.test.ts` (same path) but its assertion changes.

### 3.6 Layer table (post-pivot)

| Layer | Role | May import | May NOT import |
|---|---|---|---|
| `types/` | Zod schemas + branded types | `zod` | everything else |
| `errors/` | typed error classes | `types/` | everything else |
| `allowlist/` | pure scope-path enforcement (checkAllowlist) | `types/`, `errors/`, `zod` | `graph/`, `handlers/`, `mcp/`, `cli/`, filesystem, `node:*` builtins |
| `graph/` | LangGraph nodes + channels + build + M2 secret preflight (`graph/preflight.ts`) | `types/`, `errors/`, `@langchain/langgraph` | `handlers/`, `mcp/`, `cli/`, `allowlist/` (allowlist runs in handlers) |
| `handlers/` | MCP request handlers; call allowlist + graph | `types/`, `errors/`, `graph/` (compiled), `allowlist/` | `mcp/`, `cli/` |
| `mcp/`, `cli/` | transport delivery | `handlers/`, `types/`, `errors/` | `graph/`, `allowlist/` directly |
| `app.ts` | composition root | all | — |

### 3.7 Allowlist preflight contract

**Directory naming:** `packages/solo/src/allowlist/` (NOT `preflight/` — avoids collision with existing `packages/solo/src/graph/preflight.ts` which handles M2-shipped secret scanning). The invariant formerly proposed as "I-19 preflight pure" is renamed to **"I-19 allowlist pure"** for the same reason (see §8).

```typescript
// allowlist/check.ts
import { z } from 'zod';
import type { SkillScope, StepSignal } from '../types/index.js';

export const AllowlistResultSchema = z.discriminatedUnion('ok', [
  z.object({ ok: z.literal(true), skipped: z.boolean().optional() }),
  z.object({
    ok: z.literal(false),
    reason: z.enum(['out_of_scope', 'readonly_violation']),
    path: z.string(),
    allowed: z.array(z.string()).readonly(),
  }),
]);

export type AllowlistResult = z.infer<typeof AllowlistResultSchema>;

/**
 * Signal → (path, intent) extraction matrix — locked at v0.7 after Codex GAN
 * Round 7 F1 flagged the §3.3 vs §3.7 signature mismatch:
 *
 *   initialize   → skipped: true (no paths referenced; brief sets the scope itself)
 *   plan         → skipped: true (plan is a proposal, no file-touching)
 *   work         → for each ToolCall in payload.toolCalls: extract (path, intent)
 *                  from the tool's args; intent = 'write' for writes/edits/shell
 *                  mutations, 'read' otherwise. Reject if ANY extraction fails.
 *   eval         → skipped: true (evaluation reads state, not files)
 *   resume       → skipped: true (operator action, no file access)
 *   retry        → skipped: true (control signal)
 *   token_request → skipped: true (no paths; see M4 Batch C1 drop — Phase 2a)
 *   status       → skipped: true (read-only envelope)
 *
 * Empty / missing scope semantics (locked at v0.7):
 *   scope === undefined          → skipped: true (permissive default; operator
 *                                  is responsible for adding scope if needed)
 *   scope.paths.length === 0     → all paths rejected as out_of_scope
 *                                  (explicit empty allowlist = deny-all)
 *   scope.readOnly === true AND
 *     intent === 'write'         → readonly_violation
 *
 * Per-path readonly (e.g. `readonly: string[]`) is deferred; requires an ADR
 * that evolves ScopeSchema + migrates example playbooks + fixtures.
 */
export function checkAllowlist(
  signal: StepSignal,
  scope: SkillScope | undefined,
): AllowlistResult {
  // 1. If scope is undefined → { ok: true, skipped: true }
  // 2. Per the matrix above, decide whether this signal type has paths to check
  // 3. For signals with paths, match each requestedPath against scope.paths
  //    globs; if no match → out_of_scope
  // 4. If any path is a write-intent and scope.readOnly === true → readonly_violation
  // 5. Otherwise → { ok: true }
  //
  // NO filesystem access — foreman does not stat / readdir. That is the agent's job.
  // Returns structured result; caller translates to WireResponse.
}
```

**Critical contract points:**

- `checkAllowlist` does no I/O. It matches the requested path string against the allowlist patterns. Whether the path actually exists on disk is the agent's / work executor's concern, not foreman's. This preserves the domain-blind posture (ADR 0007).
- The signature takes the full `StepSignal` (Zod-parsed) + optional `scope`, NOT a raw `(path, intent, scope)` triple. The function itself runs the extraction matrix. This keeps the extraction logic centralized and auditable (the M3-era mistake of "validation scattered across handlers" does not repeat).
- `handlers/step.ts` calls `checkAllowlist(signal, state.brief?.playbook?.scope)` once at the top of the dispatch (after Zod parse, before any write). On `{ ok: false, ... }` it returns a `PREFLIGHT_BLOCKED` `WireResponse` and appends a journal event with category `'preflight'` (see §3.3 journal.ts update).

## 4. Dogfooding integration

M4 planned playbook-driven dogfooding where foreman assembled prompts. M5 replaces that with **agent-rendered dogfooding**:

1. Operator authors `brief.playbook` as inline `Playbook` object (YAML → `validate-playbook` → JSON → signal payload), OR points to one of `examples/{tdd-feature,refactor-extract}/playbook.yaml`
2. `foreman__step type=initialize` seeds state with the playbook
3. Agent calls `foreman__status` → receives current state including `brief.playbook.phases.plan` (raw string)
4. **Agent** resolves any `${var}` / `{{var}}` placeholders locally using state context (plan, prior work, etc.) and constructs its planning prompt
5. Agent plans, submits `foreman__step type=plan` with plan payload
6. Repeat for work and evaluate phases — foreman never sees the rendered prompt
7. Operator logs friction into `docs/dogfooding-log.md` (2 runs minimum: one terminal, one composite)

**Batch G acceptance:** one `tdd-feature` run + one `refactor-extract` run, both logged with ≥1 actionable friction item or an explicit "no friction" line + second-reviewer sign-off (anti-vanity gate carried from M4).

## 5. Risk register

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| R-M5-1: M4-pivot revert doesn't cleanly commit before M5 Batch A starts (uncommitted 14-file diff sits in working tree) | HIGH | HIGH | M5 blocks on a verified `m4-alpha` tag cut against a green post-pivot tree; Batch A pre-flight asserts `git status --porcelain` is clean |
| R-M5-2: Agent-rendered prompts drift between Claude Code / Copilot CLI / Cursor — playbook author has no way to verify consistency | MEDIUM | HIGH | `docs/agent-setup.md` documents a canonical placeholder syntax (`${var}` recommended); each example playbook README includes a sample rendered prompt for operator sanity-check; dogfooding-log captures drift-in-practice |
| R-M5-3: Allowlist scope creep — operators want filesystem awareness (symlink / case-insensitive FS / etc.) | MEDIUM | MEDIUM | ADR 0008 (to be authored in Batch F — see §7.F + §3.3 file list) locks "pure string match, no I/O" as a required min-content item; symlink/case handling moves to a Phase-2a enhancement with a separate ADR |
| R-M5-4: Scenario test suite is slow (8 scenarios × graph invocation × checkpointer I/O) and blocks CI | MEDIUM | MEDIUM | Run scenarios 1-5 + 7-8 against in-memory `SqliteSaver` (`:memory:`); scenario-06 uses file-backed tmp DB (dirty-shutdown recovery requires persistence — §7.D). Parallelize via vitest threads. Budget: soft target locked at Batch D mid-work once measurements exist; hard fail at >60s aggregate per §7.D. |
| R-M5-5: `refactor-extract` composite playbook exposes a bug in the 2-tool composite flow that M4 batches never hit (because M4 shipped only tdd-feature) | MEDIUM | MEDIUM | Scenario-02 + scenario-07 are the probe; if a bug surfaces, hotfix lands in M5 before G4 can close |
| R-M5-6: `agent-setup.md` walkthrough doesn't actually work on a clean Copilot CLI install (version skew, auth flow change, etc.) | LOW | MEDIUM | Batch G acceptance requires a clean-VM reproducer; failures push to Batch I remediation before m5-alpha tag |
| R-M5-7: Traceability doc becomes a stale artifact — every invariant change requires manual row update | MEDIUM | LOW | `traceability-snapshot.test.ts` grep-and-matches the doc against `foundations.md`; CI fails on drift |
| R-M5-8: Integrity sweep carve-out file grows into a junk drawer | LOW | LOW | Each entry requires a linked ADR or commit SHA; review gate at Batch F |
| R-M5-9: S-2..S-5 materialization reveals gap in the current S-* test coverage not flagged by M3/M4 | LOW | MEDIUM | Batch E maps every S-* to a test; if a gap is found, Batch E writes the test (scope creep risk accepted because foundation invariants trump ship-speed) |
| R-M5-10: 8-batch scope blows the ~2-week window | MEDIUM | MEDIUM | Descoping rule: push scenario suite (G4) to M6 before cutting playbooks (G9) — 2-playbook target is the proposal-level differentiator |

## 6. Timing table

| Batch | Scope | Est. hours | Risk |
|-------|-------|-----------|------|
| A | `inspect` + `validate-playbook` CLI (new work — M4 Batch E/F never started per m4-evaluation.md) + YAML parser dep audit + exit-code matrix tests | 8-12 | MEDIUM |
| B | `examples/tdd-feature/` + `examples/refactor-extract/` YAML + README | 3-4 | LOW |
| C | Preflight allowlist impl + unit tests + `PREFLIGHT_BLOCKED` error code + handler integration | 5-7 | MEDIUM |
| D | Scenario suite (8 scenarios) + test harness + `:memory:` for most, file-backed tmp DB for scenario-06 (dirty-shutdown recovery) | 8-12 | MEDIUM |
| E | Traceability doc + snapshot test (with fs-existence check per F6) + S-2..S-5 coverage audit (gap-fill if needed) | 4-6 | LOW |
| F | Integrity sweep runner + carve-out doc + ADR 0007 (already landed at M4 close) + ADR 0008 | 3-4 | LOW |
| G | `agent-setup.md` + mechanical CI gates (see §7.G) + manual release-evidence bundle (separate from CI) | 5-7 | MEDIUM |
| H | Dep-cruiser rule cleanup + post-WORK GAN + `m5-alpha` tag | 3-4 | LOW |
| **Total** | | **39-56 hours** | |

Fits ~2 weeks of focused work. Looser than M4's 45-65h because M5 introduces no new port / adapter / LangGraph channel plumbing; it ships existing seams + documentation + tests. Re-baselined 2026-04-19 after GAN Round 1 caught Batch A as new-work (not carryover) + Batch D scenario-06 needing file-backed DB + Batch F ADR 0007 already-landed.

## 7. Batch layout with acceptance criteria

### Batch A — CLI: inspect + validate-playbook (NEW WORK)

**Important:** despite M4 proposal framing, no `inspect` / `validate-playbook` code landed at M4 — Batches E/F never started (see `m4-evaluation.md` landed-vs-proposed table). This is fresh work, not carryover. Sizing in §6 reflects that (8-12h, MEDIUM risk — not 5-7h LOW).

**Ships:** `cli/inspect.ts`, `cli/validate-playbook.ts`, CLI registration in `cli/index.ts`, integration tests under `tests/integration/`.

**Acceptance (all mechanical):**
- `foreman inspect` outputs `WireResponse<{ state, journal }>` JSON — `tests/integration/cli-inspect.test.ts` asserts shape via Zod parse
- `foreman inspect --pretty` wraps in section headers without changing JSON schema — same test asserts pretty mode contains `===` section markers AND that stripping them yields valid JSON
- `foreman inspect --foreman-dir /tmp/xyz` honors the override — same test uses a mkdtemp-ed dir and asserts inspect reads from it
- `foreman validate-playbook <valid.yaml>` exits 0, stdout = `WireResponse<Playbook>` parsable — `tests/integration/cli-validate-playbook.test.ts` exit-code matrix
- `foreman validate-playbook <malformed.yaml>` exits 1, stdout = `WireResponse<never>` with `PLAYBOOK_VALIDATION_FAILED` — same test
- `foreman validate-playbook <missing-file>` exits 1, stderr contains path — same test
- YAML parser present (either already installed or `yaml@2.x` pinned); verified by `pnpm list yaml` exit 0 in a CI step

### Batch B — example playbooks

**Ships:** `examples/tdd-feature/playbook.yaml` + README; `examples/refactor-extract/playbook.yaml` + README.

**Acceptance (all mechanical per Codex GAN Round 8):**
- `pnpm foreman validate-playbook examples/tdd-feature/playbook.yaml` exits 0
- `pnpm foreman validate-playbook examples/refactor-extract/playbook.yaml` exits 0
- Each README passes `tests/contract/example-readme-template.test.ts` which asserts exact heading presence (`## Description`, `## When to Use`, `## Operator Commands`, `## Sample Rendered Prompt`) + that the code-fenced blocks under `## Operator Commands` and `## Sample Rendered Prompt` contain at least one non-empty line each (prevents vacuous-pass per R8 finding)
- `examples/` explicitly NOT in the npm tarball `files` array — grep-assertion in `tests/contract/package-json-files.test.ts` reads `packages/solo/package.json` and asserts `files` array does NOT contain `examples`
- `refactor-extract` playbook is composite: `tests/contract/example-skills-shape.test.ts` asserts `phases.work` contains the literal word `consolidate` (or `consolidation`) and that `scope.paths.length >= tdd-feature.scope.paths.length`

### Batch C — preflight allowlist

**Ships:** `allowlist/check.ts`, `allowlist/index.ts`, unit tests (including N=100 purity harness), `handlers/step.ts` integration, `types/journal.ts` `'preflight'` category, `PREFLIGHT_BLOCKED` error code.

**Acceptance:**
- `checkPreflight` is pure (no `fs`, no `process`, no timers) — ESLint `no-restricted-imports` rule
- Unit test matrix covers: exact match, glob match, negation, parent-traversal (`..`), readonly violation on write intent, readonly allow on read intent
- Handler returns `PREFLIGHT_BLOCKED` when signal violates scope
- Journal records the preflight block with category `'preflight'` (distinct from `'integrity'` and `'cascade'`)
- No existing test regresses

### Batch D — scenario suite

**Ships:** 8 scenarios under `tests/e2e/`, shared harness helpers, dual checkpointer strategy: `:memory:` for scenarios 1-5 + 7-8 (fast), file-backed scratch tmp-dir SQLite for scenario-06 only (survives process-termination simulation).

**Acceptance:**
- All 8 scenarios green on CI. **Timing budget locked at Batch D mid-work once the first scenario runs produce measurements.** Hard fail at aggregate over 60s (rule: scenario suite must stay within 2× typical integration-test budget); intermediate "acceptable" soft target is a Batch D deliverable alongside the first timing measurement.
- Each scenario's journal produces a stable snapshot (timestamps stripped)
- Coverage delta (per Codex GAN Round 8): baseline = M4-alpha commit (`64dd599`) lcov.info; metric = `statements` coverage over `packages/solo/src/handlers/**` + `packages/solo/src/graph/**`; target = +5 percentage points absolute at M5 Batch D close, verified by `scripts/coverage-delta.sh` which diffs the two lcov reports
- **Scenario-06 (dirty-shutdown recovery):** uses file-backed `new Database(dbPath)` + `SqliteSaver`; scenario opens a session, writes plan/work events, closes DB handle without explicit graceful shutdown, re-opens with same `dbPath`, asserts state rehydrates. `:memory:` is incompatible with this claim and must NOT be used here.
- **Scenario-07 (composite 2-playbook):** proves a composite parent initializes 2 children, each with a distinct `brief.playbook` (immutable-after-init per I-1). Each child's `foreman__status` returns its own phase content verbatim; cross-child reads never leak the sibling's playbook. **Not brief mutation** — any test that changes a node's playbook post-init is out of scope here and requires an ADR amending set-once semantics.

### Batch E — traceability + S-* audit

**Ships:** `docs/phase-1-solo/architecture-traceability.md`, `tests/contract/traceability-snapshot.test.ts`, any gap-filling tests.

**Acceptance:**
- Every invariant declared in `foundations.md` has a row with at least one test path
- Snapshot test **fs-verifies** each cited test path exists under `packages/solo/tests/` via `fs.statSync()` — a row with a typoed or fabricated path fails the build (closes Codex GAN Round 1 F6 — grep-only check can pass on fake paths)
- Snapshot test fails if a new invariant lands in `foundations.md` without a row
- S-2 (evaluator readOnly) has an explicit contract test
- S-3, S-4, S-5 each have at least one e2e or integration test referenced (test file existence checked by the snapshot)
- Any gap discovered during audit is closed in this batch, tracked as a line in `docs/phase-1-solo/architecture-traceability.md` (§Audit findings) with the invariant → new test path → commit SHA. `tests/contract/traceability-snapshot.test.ts` fails if any §Audit findings row cites a test path that `fs.statSync()` cannot resolve.

### Batch F — integrity sweep + ADRs

**Ships:** `scripts/integrity-sweep.sh`, `docs/phase-1-solo/integrity-sweep-carveouts.md`, `docs/adr/0008-scope-allowlist.md` (ADR 0007 already landed at M4 close — §6 timing accordingly re-baselined).

**Acceptance (mechanical per Codex GAN Round 8):**
- `bash scripts/integrity-sweep.sh` exits 0 against the clean tree
- `rg -c` against the sweep script confirms patterns include ALL of: `PromptAssembler`, `prompt-view`, `view-mapping`, `nextPrompt`, `blockedFrom`, `state\\.spec`, `ArtifactChecker` (`tests/contract/integrity-sweep-coverage.test.ts`)
- ADR 0007 **already landed at M4-alpha** (commit `64dd599`) — this Batch F ships only ADR 0008
- ADR 0008 content verified by `tests/contract/adr-0008-content.test.ts`: must contain H2 headings matching all 6 min-content items from §3.3 — `## Severity Table`, `## Signal Extraction Matrix`, `## Missing Scope Semantics`, `## Pathless Signal Policy`, `## Journal Event Shape`, `## Recovery Path`

### Batch G — agent-setup + dogfooding

**Ships:** `docs/agent-setup.md`, 2 dogfooding-log entries, manual release-evidence bundle (not CI-gated — see split below).

**CI-gated acceptance (mechanical):**
- `agent-setup.md` has sections for Claude Code, Copilot CLI, Cursor + "Externalized rendering" section — grep-checkable headers in `tests/contract/agent-setup-structure.test.ts`
- Each example playbook has a companion README with sections matching a documented template — grep-checkable
- `rg` search for forbidden instructions in `agent-setup.md` (e.g., "run `foreman render`") returns zero hits

**Manual release evidence (operator-gated, NOT CI):**
- **(MANUAL) Clean-VM walkthrough:** on a fresh VM with only `node` + `pnpm` + chosen agent installed, the quick-start command sequence produces a green `foreman inspect`. This is agent-side behavior foreman cannot verify; result recorded as a dated log entry signed by the reviewer at `docs/release-evidence/m5-clean-vm-log.md` (new file).
- **(MANUAL) Dogfooding log:** one entry per example playbook, `### YYYY-MM-DD — <title>` template, with ≥1 actionable friction item or "no friction" + second-reviewer sign-off (anti-vanity gate). Reviewer's judgment — CI may grep the file for the marker lines but not assess quality.

Split rationale per Codex GAN Round 1 F7: foreman milestone gates must be deterministic (reproducible from git state). Agent-side walkthroughs + subjective friction assessment don't fit that bar and are better tracked as release-evidence artifacts than CI gates.

### Batch H — cleanup + GAN + tag

**Ships:** dep-cruiser rule adjustments, post-WORK GAN remediations, `m5-alpha` tag.

**Acceptance (mechanical where possible per Codex GAN Round 8):**
- Dep-cruiser rules reflect §3.5 (retired M4 rules; new `allowlist-pure-imports` + `allowlist-no-node-builtins` + `no-prompt-assembler-resurrection` + `no-renderer-function-resurrection`) — verified by `tests/contract/depcruise-rules-snapshot.test.ts`
- **(MANUAL)** Codex GAN archived review at `docs/release-evidence/m5-gan-codex.md` + Copilot sign-off at `docs/release-evidence/m5-gan-copilot.md`; both must record zero BLOCKER/HIGH findings at close (operator confirms before tag)
- `m5-alpha` annotated tag contains the exact strings "I-18", "I-19", "I-20", "S-2", "S-3", "S-4" in its message — verified by `git tag -n99 m5-alpha | rg -c '<each pattern>'`
- **(MANUAL)** Operator confirms no open GitHub issues labeled `m5-blocker` or `m5-high` at tag time (query: `gh issue list --label m5-blocker --label m5-high --state open`)

## 8. Invariants introduced at M5

- **I-18** "Agent is the sole prompt renderer" — foreman emits raw playbook phases verbatim; no `nextPrompt` / `assembledPrompt` / projection field appears in `StatusSuccessSchema`, `StepSuccessSchema`, any graph channel, any MCP tool envelope, or any `cli/inspect` / `cli/validate-playbook` output. **Four-layer enforcement** (M4 lesson — filename grep alone is NOT enough; Codex GAN Round 7 F7 added layer 4):
  - `no-prompt-assembler-resurrection` dep-cruiser rule (filename sentinel)
  - `no-renderer-function-resurrection` contract test — greps all `src/**/*.ts` for function names matching `render.*[Pp]rompt|assemble.*[Pp]rompt|project.*[Vv]iew|buildPrompt|composePrompt` (catches function-level resurrection that filename sentinel misses)
  - `tests/contract/brief-exposure.test.ts` already guards `foreman__status` envelope; M5 adds analogous guards for `foreman__step` envelope (extend `transport-handler.test.ts`) + `cli/inspect` JSON + `cli/inspect --pretty` text output (new `tests/integration/cli-inspect.test.ts` assertion block)
  - Schema-level: each new Zod schema that ships in M5 includes a `.strict()` call to reject unknown keys, so a stray `nextPrompt` field cannot silently pass Zod parse
- **I-19** "Allowlist is pure" — `allowlist/**` imports only `types/`, `errors/`, and `zod` (matches §3.5 `allowlist-pure-imports` rule + §3.6 layer table — reconciled against Codex GAN Round 1 F5; renamed from "preflight" to avoid collision with existing `graph/preflight.ts` per Codex GAN Round 7 F6). Enforcement layered:
  - Dep-cruiser `allowlist-pure-imports` (import graph) + `allowlist-no-node-builtins` (no `node:*`)
  - ESLint `no-restricted-imports` for legacy un-prefixed builtins (`fs`, `path`, `os`, `crypto`, `process`)
  - **Purity harness in `tests/unit/allowlist-check.test.ts`:** N=100 invocations of `checkAllowlist` with identical inputs must produce byte-identical outputs with zero observable side effects (sentinel spy on `Date`, `Math.random`, `process.env`). This catches global-access that static rules cannot.
- **I-20** "Traceability is self-checking" — every `foundations.md` invariant has a traceability row with a test path that **fs-resolves** to a real file under `packages/solo/tests/`. Enforced by `traceability-snapshot.test.ts` which calls `fs.statSync(testPath)` on each cited path (closes Codex GAN Round 1 F6 — grep-only check can pass on fabricated paths).

(Note: M4 invariants I-10..I-17 were tied to the now-reverted `PromptAssemblerPort`; all six assembler-scoped invariants (I-10/I-11/I-14/I-15/I-16/I-17) are retired in ADR 0007. I-12 "no `TokenVendingPort` before Phase 2a" + I-13 "composition root" retained with narrower framing. Original M4 proposal used `TokenVendorPort` — canonical post-pivot spelling is `TokenVendingPort` per ADR 0007 L128 + ADR 0003 amendment + foundations.md D-20. Handler-level `kind === 'composite'` branching in `handlers/step.ts` is explicitly authorized by ADR 0004 §"Composite parent work phase = consolidation" — no invariant governs it.)

## 9. Pre-v0.1-lock actions

Before this proposal promotes to vN locked:

1. **Commit the M4-pivot revert** (14 files, currently staged-deleted) with a clear message; tag `m4-alpha` against that commit. M5 Batch A does not start until the tag exists.
2. **Reconcile M4 proposal status header** — add a "superseded in part" pointer so readers don't mistake it for the active plan.
3. **Confirm `yaml@2.x` present or pinned** — `validate-playbook` dep audit.
4. **`copilot-plugin/` disposition — RESOLVED 2026-04-19:** unrelated workspace tooling, not a foreman deliverable (see §11 Q-M5-1). `agent-setup.md` Batch G writes generic MCP-stdio integration for Claude Code / Copilot CLI / Cursor without anchoring on `copilot-plugin/`. No further action in M5.
5. **ADR 0007 first pass** reviewed before Batch F codifies it.
6. **Two GAN rounds** against this v0.1 (Codex + Copilot, hexagonal + domain-blind lenses) before promotion to vN.

## 10. Non-goals (locked)

1. **README / v0.1 tarball / changelog** — M6
2. **Perf benchmarks** (`Perf-Q-a`/`Perf-Q-b`) — M6
3. **Daemon / HTTP / WebSocket** — Phase 2a
4. **`TokenVendingPort`** — Phase 2a (M4 pivot Batch C1 drop stands; canonical post-pivot spelling is `TokenVendingPort` — original M4 proposal used `TokenVendorPort`, renamed for consistency with ADR 0003 amendment + foundations.md D-20 + ADR 0007)
5. **Prompt rendering inside foreman** — ADR 0007 locks externalization; any revival requires a new ADR that supersedes 0007
6. **3rd example playbook** — 2-playbook target meets G9; 3rd defers to M6+
7. **Composite-of-composite** — scenarios test composite-of-terminals; deeper nesting deferred
8. **Cross-project playbook registry** — Phase 2+
9. **Playbook hot-reload** — `brief.playbook` is embedded in the initialize signal and fixed set-once (I-1 of docs/architecture.md §1.6 ("Brief is immutable after init")); no reload concept, no mid-flow playbook swap. Mutation attempts are rejected by the set-once reducer (already tested in `tests/unit/channels.test.ts`).
10. **Symlink / case-insensitive / canonicalization in allowlist** — ADR 0008 (to be authored in Batch F) will lock pure-string match; any filesystem awareness requires a separate ADR
11. **Error-catalog full audit** — M6
12. **Dashboard / UI** — Phase 2b
13. **Windows-specific agent-setup** — best-effort only; WSL recommended

## 11. Open questions (for v0.1 GAN rounds)

**Q-M5-1 RESOLVED (operator 2026-04-19 M4 commit review):** `copilot-plugin/` is an unrelated workspace plugin (Codex-orchestration harness used by Claude Code, not a foreman deliverable) and stays OUT of the foreman repo's tracked surface. `agent-setup.md` at Batch G documents generic MCP-stdio integration for Claude Code / Copilot CLI / Cursor without anchoring on `copilot-plugin/`. If a reference agent is shipped at M5+, it lives under `examples/reference-agent/` as a separate deliverable with its own ADR.

**Q-M5-2:** preflight severity — is `PREFLIGHT_BLOCKED` always a hard-block, or does `playbook.scope.readOnly === true` + write-intent violation warn-and-proceed in some cases? ADR 0008 must lock this.

**Q-M5-3:** traceability doc format — single-table or per-invariant section? Codex/Copilot often prefer tables for greppability; architect prefers sections for context. Default: single table.

**Q-M5-4 RESOLVED (Round 1 GAN):** scenario-07 is composite-two-playbook **isolation**, not brief mutation. `brief.playbook` is immutable-after-init per I-1 (phase-1-solo/architecture.md §1.6, enforced by `setOnceReducer` at `packages/solo/src/graph/channels.ts`). A conflicting second write throws `SCHEMA_VALIDATION_FAILED` — this is tested in `tests/unit/channels.test.ts`. Mid-flow playbook swap would require an ADR amending set-once semantics and is explicitly out of scope (§10 non-goal #9).

**Q-M5-5:** integrity-sweep runner — shell script vs Node script? Shell (`rg` / `grep`) is fastest; Node gives cross-platform portability. Default: shell + a Windows note.

**Q-M5-6:** should `agent-setup.md` include a Codex / Cursor example, or strictly Claude Code + Copilot CLI? Scope decision: more examples = more clean-VM walkthrough surface to maintain.

**Q-M5-7:** dogfooding anti-vanity gate — does "second-reviewer sign-off" require a human or can Codex count? Previous M4 language was ambiguous; lock this for M5.

## 12. Next steps

1. **Pre-v0.1-lock checklist** (§9 items 1-6) — complete before proposal locks
2. **Proposal Round 1 GAN** (Codex + Copilot, hexagonal + domain-blind lenses) — fold findings → v0.2
3. **Proposal Round 2 GAN** (if needed based on R1 severity) → v0.3
4. **Operator sign-off** — lock to vN
5. **Batch A starts** — per-batch mid-WORK + post-WORK GAN (M3/M4 pattern)
6. **Batch-H post-WORK sign-off** → tag `m5-alpha`

## 13. Lineage

- **v0.1 (this file)** — authored 2026-04-19 post-M4-pivot; absorbs un-shipped M4 remainder + sprint-8 intent under current M-naming
- **Prior lineage not applicable** — unlike M4, M5 has no skeleton tier; drafted directly against the post-pivot architecture with the intent of 2 GAN rounds before promotion

### Expected GAN round taxonomy

- **R1 Codex** — adversarial: scope creep, untestable gates, hidden assumptions about M4 close
- **R1 Copilot** — cross-file: doc drift, grep-assertable claims, stale references
- **R1 architect (optional)** — hexagonal + domain-blind lens: ADR 0007 coherence; no backdoor port revivals
- **R2 Codex + Copilot** — re-review after R1 remediation; blocker check
- **Operator sign-off** — R2 sign-off + simultaneous ADR 0007 + 0008 acceptance flip

Batch A starts post operator sign-off on vN.
