# Phase 1 Solo — Specification

**Status:** Locked — Phase 1 production baseline (2026-04-17; ~20 rounds of Codex + Copilot review; final pre-launch audit pass applied)
**Last updated:** 2026-04-17
**Scope:** Phase 1 Solo ship plan — methodology, milestones, ship criteria, testing, wire contracts
**Related:** [`foundations.md`](../foundations.md), [`roadmap.md`](../roadmap.md), [`architecture.md`](../architecture.md), [`phase-1-solo/architecture.md`](architecture.md)

This document is the **operational spec** that materializes `phase-1-solo/architecture.md` (the design) into an implementable plan: what gets built, in what order, to what quality bar, with what tests, under what methodology.

---

## 1. Purpose and scope

### 1.1 What this spec is

- The **ship plan** for `@foreman-lab/solo@1.0.0`
- Locks: methodology choice, 7-milestone plan, 10 ship gates, testing discipline, MCP/CLI wire contracts, signal schemas, checkpoint schema, example playbook contracts, agent-transcript format, dep inventory, CI pipeline, reference machine, dogfooding protocol

### 1.2 What this spec is not

- Not the architecture — see `phase-1-solo/architecture.md`
- Not the principles — see `foundations.md`
- Not the tier roadmap — see `roadmap.md`
- Not a tutorial or onboarding guide

### 1.3 Exit criteria (when this spec is done)

All 10 ship gates in §4 pass, all 8 testing-methodology subsections in §5 hold on the real code, and the M6 release checklist is complete.

---

## 2. Methodology

### 2.1 Pick: Kanban + XP-lite hybrid

Locked after Codex + Copilot debate. Rationale:

- **Solo dev** eliminates Scrum ceremony (sprints, standups, retros, planning poker all assume a team)
- **Kanban** = pull-based flow + WIP limit — one person does one thing at a time
- **XP-lite** = TDD + CI + small releases + "simplest thing that works" — cherry-picked practices that survive the solo constraint (pair programming does not apply)
- **Shape Up's 6-week cycles** are too coarse; our milestones are 1–2 weeks each
- **Waterfall's design phase** is already done (4 locked docs); implementation is iterative, not waterfall
- **Strict Scrum / Shape Up / Pure Waterfall** rejected

### 2.2 Working agreements

| Rule                               | Value                                                                                                         |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------- |
| WIP limit (milestones)             | **1** active milestone — no parallel milestone work                                                           |
| WIP limit (tasks within milestone) | **2** — one coding, one waiting on CI/test                                                                    |
| Task list                          | **Locked per milestone** in this spec (§3); drift goes to `BACKLOG.md`, reviewed only at milestone boundaries |
| Weekly Friday self-review          | 15 min written in local `DEVLOG.md` (not committed) — what shipped, what blocked, what's next                 |
| Release                            | **When-it's-ready** (P-9); hard 1.5× cutoff per milestone prevents drift                                      |

### 2.3 Definition of done per milestone

1. All milestone tasks complete per §3 task list
2. All milestone tests pass in CI (no skips, no `xit`/`xdescribe`)
3. No regressions in prior milestone tests
4. `DEVLOG.md` entry for the milestone close

### 2.4 Risk management

- **Scope creep**: strictly locked task lists. Addition mid-milestone is rejected; goes to `BACKLOG.md` or triggers an ADR if structural.
- **Timebox**: soft target + **1.5× hard cutoff per milestone**. If a 2-week milestone hits day 21 not done, stop and triage. Cut polish/docs first. **Never cut** S-1..S-5 security tests, scenario tests, checkpoint integrity, or MCP validation.
- **Per-milestone cutlines** (ordered least to most painful):
  - M0: fewer CI jobs (keep install/typecheck/lint/test; defer audit + gitleaks to M1)
  - M1: stub `docs/error-codes.md` to 7 known codes only; full sweep at M6
  - M2: ship only `init` + `status` handlers in walking slice; add `plan` to M3
  - M3: **delay dogfooding to M4**; never cut MCP surface or handlers (breaks G2/G3)
  - M4: ship 1 example playbook (`tdd-feature` only); add `refactor-extract` at M5
  - M5: drop 3rd agent (Cursor); keep Claude Code + Copilot CLI only; defer perf benchmarks to M6
  - M6: non-negotiable — release is release
- **External blockers** (R-1 LangGraph API, R-2 MCP SDK, R-15 supply chain): exact-pin deps; security bumps go through the CVE fast-path per R-1 mitigation; all other bumps require ADR.
- **Weekly DEVLOG** catches drift early (Friday check — am I on track?).

### 2.5 Dogfooding transition

- **Decision (locked):** dogfooding starts **at the end of M3** (Solo Alpha), not M4 or post-v1.
- **Rationale:** friction surfaces real UX bugs while they are still cheap to fix. Waiting until v1 forfeits the most valuable dogfooding window.
- **Action at M3 close:** add `.foreman/` to the foreman repo; use `foreman init` + **raw signals** to orchestrate M4 work. Post-2026-04-18 domain-blind pivot (ADR 0007): playbook-based orchestration kicks in at M5 close when example playbooks + `agent-setup.md` ship. No runtime `PromptAssemblerPort` / `SkillMethodologyAssembler` — external agents render prompts from raw `brief.playbook.phases[phase]` carried through the `brief` channel. (Historical class names `PromptAssemblerPort` / `SkillMethodologyAssembler` retained as retired-symbol references.)
- **M3/M4 dogfooding scope:** tracking work via `foreman init` + hand-crafted `brief` objects, `foreman step` submitting raw `plan`/`work`/`evaluate` signals, `foreman inspect` reading state. Agent authors prompts manually throughout M3 and the post-pivot M4.
- **M5 dogfooding upgrade:** once `tdd-feature` / `refactor-extract` example playbooks ship (G9 closure at M5), M5/M6 tasks move to playbook-driven orchestration with the agent resolving `brief.playbook.phases[phase]` placeholders locally.
- **Boundary:** vitest remains the **independent test oracle**. foreman-in-use is a canary, not a gate. If foreman breaks during dogfooding, file a bug against foreman; do not alter the test suite to accommodate the bug.
- **Artifact:** `docs/dogfooding-log.md` captures friction, gets polished at M6.

---

## 3. Milestone plan (M0–M6)

Total target: **10–11.5 weeks** from M0 start to v1.0.0 tag.

### 3.1 M0 — Bootstrap (~1 week)

**Purpose:** empty but correct repo; CI green on a no-op commit.

**Tasks:**

- `pnpm-workspace.yaml` + `turbo.json` + root `package.json`
- `packages/solo/package.json` with `@foreman-lab/solo` name, `"bin": { "foreman": "dist/index.js" }`, `engines.node = ">=20"`, exact-pin deps placeholder
- `tsconfig.json` (ES2022, NodeNext, strict)
- `eslint.config.js` + `.prettierrc.json` + `dependency-cruiser` config enforcing D-17 layer boundaries on `packages/solo/src/*`
- `vitest.config.ts` + `vitest.coverage` config (c8/istanbul provider)
- `.npmrc` with `ignore-scripts=true` for `@langchain/*` + `@modelcontextprotocol/*` (per R-15)
- `.pnpmauditrc.json` (empty `{ "ignore": [] }` — policy format documented in §13)
- `scripts/check-licenses.mjs` (allowlist: `Apache-2.0`, `MIT`, `BSD-2-Clause`, `BSD-3-Clause`, `ISC`; blocks: `GPL-*`, `AGPL-*`, `LGPL-*`, proprietary)
- `.github/workflows/ci.yml` — install, typecheck, lint, test, boundary-check on push + PR
- `LICENSE` (Apache-2.0), `.gitignore`, **reading-order README.md** (one-page: "What is foreman / 2-hour doc-chain reading list / project status" with links in order: `foundations.md` → `roadmap.md` → `architecture.md` → `phase-1-solo/architecture.md` → `phase-1-solo/spec.md` → `adr/0001-...`)
- **`CONTRIBUTING.md` stub** (one paragraph: "This project is under Phase 1 active development by a single maintainer; external contributions are welcomed as issues/discussion on GitHub but PRs are paused until v1.0.0 tag. For design context, start with `docs/foundations.md`. Full contributor guide lands at M6.")

**Definition of done:** `pnpm install && pnpm typecheck && pnpm lint && pnpm test` pass on a fresh clone; CI green on main.

**Gate satisfied:** G1 (monorepo + import-boundary lint).

### 3.2 M1 — Contracts (~1 week)

**Purpose:** lock the wire protocol and error shapes before any logic exists.

**Tasks:**

- `packages/solo/src/types/` — Zod schemas for: `Brief`, `Skill`, `SignalType` union, all 7 signal envelopes, `ToolCall` (with `reversibility`), `WireResponse<T>`, `JournalEvent`, `Finding`, state channels
- `packages/solo/src/types/errors.ts` — `ErrorCode` union (SCREAMING_SNAKE strings)
- `packages/solo/src/errors/` — `BaseError` + `ValidationError` + `SecurityError` + `StorageError`
- `docs/error-codes.md` — stub with the 7 starter codes from spec §8.3 (authoritative M1 list)
- TDD: first failing test = "InitializeSignal with missing `kind` fails Zod parse"

**Definition of done:** every Zod schema has at least two tests (accept + reject); `ErrorCode` is a TS union, not an enum; error classes extend `Error` with `code` + `message` + `details` fields.

**Gate progress:** G2 (schemas side), G7 (error-codes.md stub).

### 3.3 M2 — Walking slice (~1.5–2 weeks)

**Purpose:** one end-to-end vertical slice executes. Does not cover all signals, but proves the architecture.

**Tasks:**

- `packages/solo/src/graph/channels.ts` — `FlowState` via `Annotation.Root` with 7 channels
- `packages/solo/src/graph/outcome.ts` — pure truth table (5 rules per architecture.md §4.2)
- `packages/solo/src/graph/preflight.ts` — secret scan stub (S-1) + integrity check stub
- `packages/solo/src/graph/nodes/{init,plan,work,evaluate}.ts` — 4 nodes (plan + work + evaluate are stubs at M2; only `init` fully implemented)
- `packages/solo/src/graph/edges/outcome.ts` — conditional router (pass/retry/block/exhausted)
- `packages/solo/src/graph/build.ts` — `buildGraph({ checkpointer, assembler })` using `SqliteSaver`
- `packages/solo/src/handlers/{initialize,plan,status}.ts` — minimum handlers to exercise init → plan → status
- `packages/solo/src/mcp/adapter.ts` — minimal MCP stdio server exposing 3 tools at M2 (`foreman__initialize`, `foreman__plan`, `foreman__status`); collapsed to 2 tools at M3 per ADR 0003 (`foreman__status` + polymorphic `foreman__step`)
- `packages/solo/src/cli/` — minimal `init`, `status` commands
- `packages/solo/src/index.ts` — composition root wiring checkpointer + graph + handlers + transport
- **Demo:** `pnpm run demo` invokes foreman end-to-end, initializes a node, emits an init event, returns status

**Definition of done:** a scripted demo completes init → plan stub → status without errors; journal has `init` + `plan` events; checkpoint DB exists and is readable via `sqlite3`.

**Gate progress:** G3 (4-node flow executes partial).

### 3.4 M3 — Solo Alpha (~2 weeks)

**Purpose:** complete the MCP + CLI surface so foreman is self-usable. **Dogfooding starts at M3 close.**

**Tasks:**

- Remaining handlers: `work`, `eval`, `resume`, `token_request`. **Note:** M3/M4 stubs `token_request` with `SCHEMA_VALIDATION_FAILED`. M4 Batch C1 (unstub + pass-through vending) was dropped 2026-04-18 to Phase 2a per operator + GAN (foreman as infrastructure does not call external APIs; pass-through vending provides zero security value vs the agent reading env directly — see ADR 0007). `TokenVendingPort` materializes at Phase 2a when Vault/HSM provides real scoping + revocation + audit.
- `handlers/mutex.ts` — per-nodeId async mutex (architecture.md §11.5)
- `handlers/ttl.ts` — 24h TTL check (architecture.md §11.6)
- `handlers/status.ts` — full CQRS read with `stale` flag
- Remaining CLI commands: `step`, `resume`, `inspect`, `validate-playbook`
- Full `foreman__*` MCP tool surface (7 tools)
- Error envelope wiring in transport layer
- `BaseError` → `WireResponse<T>` converter at MCP/CLI boundaries

**Added to M3 (migrated from M5 for deliverability — see B4 in the integrity pass):**

- `test/contract/handler-graph.test.ts` — handler↔graph boundary tests (`graph.invoke` input shape; returned state shape; `graph.getState` channel shape)
- `test/contract/transport-handler.test.ts` — transport↔handler boundary tests (Zod-parsed signal → handler → `WireResponse<T>` envelope)

**Definition of done:** all 7 signals accepted + dispatched; mutex serializes same-nodeId invocations (test: Q-6a); TTL marks nodes stale; `foreman inspect` dumps state; self-invoke smoke test passes; both contract test suites green.

**M3 close = dogfooding begins.** Operator runs `foreman init` in the foreman repo itself; subsequent M4 tasks tracked as foreman nodes using raw signals. Post-2026-04-18 domain-blind pivot (ADR 0007): playbook-based orchestration slipped from M4 to **M5 close** (when example playbooks + `agent-setup.md` ship) — see §2.5.

**Gate progress:** G3 (flow complete), G2 (MCP wiring).

### 3.5 M4 — Port + playbooks (~1.5 weeks)

**Purpose (post-2026-04-18 pivot):** materialize D-18 (playbook injection) + two example playbooks + external-rendering agent-setup. D-20 (port discipline) is re-interpreted — Phase 1 ships zero ports after the PromptAssemblerPort revert (ADR 0007); the discipline itself stands unchanged for Phase 2a `TokenVendingPort` promotion.

**Tasks (deferred to M5 per the 2026-04-18 domain-blind pivot — M4 itself ships only the revert + carryover remediation + ADR 0007 + the supersession banner):**

- `examples/tdd-feature/playbook.yaml` — example terminal playbook phases (M5)
- `examples/refactor-extract/playbook.yaml` — example composite playbook phases (M5)
- `docs/agent-setup.md` — copy-paste MCP config snippets for Claude Code, Copilot CLI, Cursor + "Externalized rendering" section showing how the agent reads `brief.playbook.phases[phase]` from `foreman__status` and constructs prompts (with npx warning per architecture.md §7.1) (M5)
- `packages/solo/src/cli/validate-playbook.ts` — `foreman validate-playbook <path>` CLI (transport-only; Zod-validates playbook YAML; no runtime coupling to any port) (M5)
- `packages/solo/src/cli/inspect.ts` — `foreman inspect` operator debugging command (M5)

**Definition of done (M5 deliverable):** `foreman validate-playbook examples/tdd-feature/playbook.yaml` returns `{ ok: true, data: Playbook }`; an external agent consuming `foreman__status` renders the three-phase prompts correctly from raw `phases` strings; `agent-setup.md` walkthrough verified against a clean-VM install of Claude Code + Copilot CLI. No runtime `PromptAssemblerPort` surface; no `nextPrompt` field in step envelope.

**Gate progress:** G9 (playbooks + docs).

### 3.6 M5 — Scenarios + coverage (~2 weeks)

**Purpose:** prove end-to-end correctness + performance + security under a real MCP agent.

**Tasks:**

- `packages/solo/tests/scenario/` — M3 walking-slice smoke (1 scenario; shipped at M3). `packages/solo/tests/e2e/` — M5 Batch D adds ≥8 end-to-end scenarios (per `docs/phase-1-solo/m5-development-proposal.md` §7.D): terminal, composite, block/unblock/fail, dirty-shutdown recovery, composite two-playbook isolation, preflight rejection. Total scenario coverage at M5-alpha: ≥9 scenarios across `tests/scenario/` + `tests/e2e/`.
- `test/load/concurrent-signals.test.ts` — Q-6a + Q-6b tests with `--pool=forks`, 60s timeout, 1 retry
- `test/security/` — one integration test per S-1..S-5 (Q-8) (see §5.9 for per-requirement test plan)
- `test/integration/mcp-clients/` — smoke tests against Claude Code, Copilot CLI, Cursor (G8)
- Coverage gate: `pnpm test --coverage` reports ≥ 80% branch on `graph/` + `handlers/`

**Note:** Contract tests migrated to M3 (handler↔graph + transport↔handler are natural at M3 when handlers + transport land); perf benchmarks (Q-11a/Q-11b) migrated to M6 (post-scenario, after flow stabilizes).

**Definition of done:** all 7 scenarios green in CI; coverage ≥ 80%; all 5 S-requirement integration tests green; 3-agent MCP smoke tests green.

**Gate progress:** G4 (scenarios), G5 (coverage), G6 (S-1..S-5), G8 (3 agents).

### 3.7 M6 — Release (~1 week)

**Purpose:** polish + v1.0.0 tag + npm publish.

**Tasks:**

- `docs/error-codes.md` — completeness sweep (every `ErrorCode` enumerated with cause + fix, per Q-7)
- `README.md` — foreman-in-30-seconds intro + install + first-run + MCP setup link
- `CHANGELOG.md` — v1.0.0 release notes
- `LICENSE` verified (Apache-2.0)
- `CONTRIBUTING.md` — minimal; points to foundations.md and spec.md
- `docs/dogfooding-log.md` — capture M3–M6 friction
- `package.json` publish metadata: `files`, `exports`, `repository`, `keywords`, `bin`
- **Perf benchmarks (migrated from M5 per B4):** `test/perf/handshake.test.ts` (Q-11a gate: p95 < 800 ms) + `test/perf/domain.test.ts` (Q-11b characterization only, no assertion)
- `pnpm pack` → inspect tarball, manual audit
- Fresh-VM install test (Q-9): `docker run -it node:20 bash -c "npm i -g /tmp/foreman-lab-solo-1.0.0.tgz && foreman --version"`
- CI green on all gates; no skipped tests
- Tag `v1.0.0`; `pnpm publish --access public`

**M6 additional deliverable — `docs/semver-policy.md`:** per foundations.md deferred decisions, a versioning/semver policy document **must be committed before `pnpm publish`**. Covers: what triggers a major bump (breaking change to signal schema, MCP tool shape, CLI command surface, or foundational decision), pre-1.0 behavior (none — foreman ships 1.0.0, not 0.x), monorepo package coordination (each `@foreman-lab/*` package versions independently). Drafted in M6, reviewed by Copilot+Codex, locked before tag.

**Definition of done:** `npm install -g @foreman-lab/solo@1.0.0` on a fresh Ubuntu 22.04 VM produces a working `foreman status` within 30 seconds. All 10 gates pass. `docs/semver-policy.md` committed and linked from README.

**Gate progress:** G7, G9 (README), G10 (install test).

---

## 4. Ship criteria — 10 gates → milestone matrix

| #   | Gate                                                            | Milestone(s)                                        | Evidence                                                                                                                                      |
| --- | --------------------------------------------------------------- | --------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------- |
| G1  | Monorepo + import-boundary lint green                           | M0                                                  | CI green on empty commit; `dependency-cruiser` green                                                                                          |
| G2  | 7 MCP tools Zod-validated (`foreman__*`)                        | M1 (schemas) + M3 (wiring)                          | Schema tests (M1) + MCP adapter integration test (M3)                                                                                         |
| G3  | 4-node LangGraph flow executes end-to-end                       | M2 (walking slice) + M3 (complete + contract tests) | Integration test: init → plan → work → eval → pass; `test/contract/*` green (migrated to M3 per B4)                                           |
| G4  | 7 scenario tests pass (architecture §6)                         | M5                                                  | `test/scenario/*.test.ts` all green                                                                                                           |
| G5  | ≥ 80% branch coverage on `graph/` + `handlers/` (Q-3)           | M5                                                  | `vitest --coverage` CI gate                                                                                                                   |
| G6  | S-1..S-5 each have ≥ 1 integration test (Q-8)                   | M5                                                  | `test/security/` suite                                                                                                                        |
| G7  | `error-codes.md` 100% complete (Q-7)                            | M1 stub + M6 sweep                                  | Enum-sweep test asserts every code has a row                                                                                                  |
| G8  | 3+ agents complete a scenario via MCP                           | M5                                                  | Smoke tests: Claude Code, Copilot CLI, Cursor                                                                                                 |
| G9  | `agent-setup.md` + 2 example playbooks + README                 | M4 (playbooks/docs) + M6 (README)                   | Checklist; snippets verified                                                                                                                  |
| G10 | `npm i -g @foreman-lab/solo` works < 30s on clean machine (Q-9) | M6                                                  | Fresh-VM test (Docker ubuntu-22.04); also gated on Q-11a perf: MCP handshake p95 < 800 ms on `ubuntu-latest` 2-core (migrated from M5 per B4) |

**All 10 gates must pass for v1.0.0 tag.** No partial ship.

---

## 5. Testing methodology

### 5.1 TDD policy (by module)

| Scope                                       | Discipline                              | Rationale                                               |
| ------------------------------------------- | --------------------------------------- | ------------------------------------------------------- |
| `graph/` (nodes, edges, outcome, preflight) | **Test-first (Red-Green-Refactor)**     | Behavior-critical; pure functions with truth tables     |
| `handlers/` (signal dispatch, mutex, TTL)   | **Test-first**                          | Concurrency + correctness; hard to verify by inspection |
| `types/` (Zod schemas)                      | **Test-first**                          | Contract stability; parse/reject round-trip tests       |
| `ports/`, `adapters/` (when they exist)     | **Test-first**                          | Post-2026-04-18 pivot: no `ports/` / `adapters/` at Phase 1 (ADR 0007); row applies if/when Phase 2a promotes `TokenVendingPort`. |
| `mcp/`, `cli/`                              | **Test-after** (integration-covered)    | Thin dispatch layers; tested via scenario + integration |
| Composition root (`app.ts`)                 | **Exempt**                              | Covered implicitly by integration/scenario tests; runtime smoke at `tests/contract/composition-root.test.ts` |
| Config files, type aliases, comments        | **Exempt**                              | Validated by tooling or usage                           |
| Spikes (`spikes/` directory)                | **Exempt — quarantined, never shipped** | Enter production through TDD                            |

### 5.2 Test shape — Diamond, not Pyramid

| Tier        | Target | Scope                                                            | Local time budget |
| ----------- | ------ | ---------------------------------------------------------------- | ----------------- |
| Unit        | ~40%   | Pure functions: outcome, preflight, mutex, TTL, Zod parse/reject | < 5 s             |
| Integration | ~40%   | Handler + graph + checkpointer; transport → handler dispatch     | < 30 s            |
| Scenario    | ~20%   | 7 end-to-end flows (architecture §6)                             | < 2 min total     |

Diamond (not pyramid) because foreman is a thin orchestration harness — most value lives at layer boundaries, not in leaf functions.

**Outcome truth-table coverage:** rules 1–4 (pass, block, retry, exhausted edge cases) are each exercised by at least one of the 7 scenarios in architecture §6. **Rule 5 (exhausted — retries exhausted) is covered by an exhaustive unit test** in `test/unit/outcome.test.ts` enumerating all 16 `(meetsCriteria × hasCriticalFinding × blockDetected × retriesRemaining)` input combinations — not by a dedicated scenario. The unit test matrix is the rule-5 witness; no 8th scenario is added at Phase 1.

### 5.3 Graph node testing strategy

- **Pure functions** (outcome, preflight, reducers): unit-tested directly, no LangGraph runtime involved
- **Node integration**: tested via handler → `graph.invoke` with real `SqliteSaver` backed by in-memory SQLite (`":memory:"`)
- **Transitions**: tested by scenario tests exercising multi-node flows (init → plan → work → evaluate)

### 5.4 Contract tests (2 boundaries at Phase 1)

| Boundary             | What's tested                                                                                            | Gate |
| -------------------- | -------------------------------------------------------------------------------------------------------- | ---- |
| Handler ↔ Graph     | `graph.invoke` input shape + returned state shape; `graph.getState` returns expected channels            | CI   |
| Transport ↔ Handler | Zod-parsed signal → handler call → `WireResponse<T>` envelope; error path returns `{ ok: false, error }` | CI   |

Port contract tests retired at Phase 1 post-2026-04-18 domain-blind pivot (ADR 0007); no runtime ports exist. Will re-add when Phase 2a introduces `TokenVendingPort` with a Vault/HSM adapter + pass-through adapter.

### 5.5 Coverage policy

- **Floor:** 80% branch coverage on `graph/` + `handlers/` (Q-3, CI gate)
- **Philosophy:** coverage is a **symptom of TDD**, not a cause. Well-disciplined TDD naturally yields 85–92%. The 80% floor catches omissions; do not pad to hit a number.
- **Exclusions (documented in coverage config):** generated types, composition root wiring, comments, `spikes/`, test fixtures

### 5.6 First failing test per milestone

| Milestone          | First red test                                                                                  |
| ------------------ | ----------------------------------------------------------------------------------------------- |
| M1 — Contracts     | `InitializeSignal with missing kind field fails Zod parse with BRIEF_KIND_MISSING`              |
| M2 — Walking slice | `init node accepts valid brief and emits init event to journal`                                 |
| M3 — Handlers      | `two concurrent signals to same nodeId serialize via mutex; second waits`                       |
| M4 — Pivot close   | `step envelope does not contain any nextPrompt / assembledPrompt / prompt-view field` (post-2026-04-18 domain-blind pivot; see ADR 0007 + tests/unit/error-codes.test.ts rejection test for PROMPT_ASSEMBLY_FAILED) |
| M5 — Scenarios     | `Scenario 1 happy path completes with outcome=pass`                                             |

### 5.7 Deferred testing tools

| Tool                          | Decision                                                             | Revisit                               |
| ----------------------------- | -------------------------------------------------------------------- | ------------------------------------- |
| `fast-check` (property-based) | YAGNI at Phase 1 — outcome truth table is 16 enumerable combinations | M5 if edge cases surface in scenarios |
| `stryker-js` (mutation)       | Phase 2+ — too expensive for solo dev                                | When contributor count > 1            |

### 5.8 Dogfooding test-authority boundary

- **vitest is the independent test oracle.** Scenario/unit/integration tests are the acceptance gates.
- **foreman-in-use during dogfooding is a canary**, not a test gate. If foreman breaks during dogfooding, file a bug against foreman; do not alter tests to accommodate.
- Agent output (plans, work reports, evaluations) is **untrusted** until tests pass independently. Dogfooding does not validate foreman; tests do.

### 5.9 Security test strategy (G6)

Gate G6 requires **one integration test per S-1..S-5**, each with concrete assertions. Test file: `test/security/<requirement>.test.ts`.

| Requirement                                  | Test name                           | Key assertion                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| -------------------------------------------- | ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **S-1** (no ambient credentials)             | `s1-no-ambient-credentials.test.ts` | Handler rejects signals containing ambient credential patterns (`AWS_*`, `OPENAI_*`, bearer tokens) in payload; token vending hook invocation is isolated (does not leak env to handler return value)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **S-2** (operator approval for irreversible) | `s2-operator-approval.test.ts`      | Work report with `ToolCall.reversibility === "irreversible"` and no prior `ResumeSignal { action: "approve" }` triggers `block` outcome with category `security`; `interrupt()` is called; graph pauses                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| **S-3** (action classification)              | `s3-reversibility-required.test.ts` | `ToolCall` without `reversibility` field defaults to `"unknown"` → treated as irreversible by S-2 logic; explicit `"reversible"` passes through                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| **S-4** (MCP input validation)               | `s4-transport-validation.test.ts`   | Malformed signals (unknown type, missing required fields, extra properties) return `{ ok: false, error: { code: "SCHEMA_VALIDATION_FAILED", ... } }` with no side effects; no journal event written; no graph.invoke. **`.strict()` scope:** applied to top-level `Signal` envelope + `Brief` + `Skill` schemas. `payload` is a discriminated union which enforces known fields per signal type via its discriminator (no `.strict()` needed on unions — discriminator handles it). Nested `ToolCall[]`, `Finding[]`, etc. do NOT use `.strict()` — forward-compatibility requires ignoring unknown fields in nested arrays (allows agents to add metadata without breaking foreman). |
| **S-5** (checkpointer-only state access)     | `s5-no-raw-sql.test.ts`             | Import-boundary lint rejects any `import` of `better-sqlite3` or raw SQLite client outside `packages/solo/src/graph/`; handlers, transport, adapters never touch DB directly (verified by dep-cruiser graph scan + grep CI check)                                                                                                                                                                                                                                                                                                                                                                                                                                                     |

All 5 tests run in `pnpm test:security` (part of `test:integration` in CI). Failures block merge.

### 5.10 Agent smoke test strategy (G8)

Gate G8 requires 3 agents to complete at least one scenario via MCP. Test file: `test/integration/mcp-clients/<agent>.test.ts`.

| Agent                  | Version pinned in CI                               | Scenario used                              | Pass criteria                                                                        |
| ---------------------- | -------------------------------------------------- | ------------------------------------------ | ------------------------------------------------------------------------------------ |
| **Claude Code**        | Pinned to a specific released version in CI matrix | Scenario 1 (happy path, tdd-feature playbook) | Scenario completes with `outcome: "pass"`; transcript diff-matches committed fixture |
| **GitHub Copilot CLI** | Pinned version                                     | Scenario 1                                 | Same                                                                                 |
| **Cursor**             | Pinned version (MCP stdio mode)                    | Scenario 1                                 | Same                                                                                 |

**Orchestration**:

- CI job `mcp:smoke` runs nightly on main and on tag push (not every PR — costly, agent setup is slow)
- Each agent is invoked via its CLI or stdio adapter; **foreman under test runs in a Docker container** (reproducible env, same image built from the tarball); **agents run natively** on the `ubuntu-latest` runner (installed via each agent's official install step in the GH Actions workflow — proprietary CLIs have no Docker images)
- Recorded transcripts (`test/scenario/scenario-1/<agent>-transcript.jsonl`) are per-agent (agent behavior varies; comparison is per-agent, not cross-agent)
- Environment: `ubuntu-latest` 2-core (§14)

**Gate tiers (resolves G8 vs skip conflict):**

- **Tag push (release):** ALL 3 agents must pass. No skips allowed. If any install fails, release is blocked.
- **Nightly main:** ≥ 2 of 3 must pass. Individual skip allowed WARN if version unavailable upstream.
- **PR:** advisory only (doesn't run on every PR — gated to nightly + tag).

### 5.11 Determinism test specification (Q-4 / P-5)

Q-4 (reproducibility) requires a concrete, falsifiable test. Production-grade definition:

**Scenario under test:** Scenario 1 (happy path, terminal node, `tdd-feature` playbook) — the simplest end-to-end path.

**Procedure:**

1. Spawn foreman as MCP stdio server in a Docker container with a freshly-seeded `.foreman/` (deterministic brief, deterministic scripted agent).
2. Drive foreman through `initialize` → `plan` → `work` until first `eval` is about to arrive.
3. `SIGKILL` the process (not `SIGTERM` — graceful shutdown masks races).
4. Restart foreman with the same `.foreman/` and same remaining agent script.
5. Continue to completion; outcome = `pass`.
6. Capture full final state (all 7 channels) + full `journal[]`.
7. Run the full scenario AGAIN without the kill (control run).
8. **Pass criterion (structural deep-equal):**
   - Every state channel (`nodeId`, `kind`, `brief`, `plan`, `work`, `evaluation`, `journal`) deeply equal between kill-run and control-run, **excluding journal event `ts` timestamps** (non-deterministic by definition — excluded per §11).
   - Journal event **order** identical (ordinal stability).
   - Journal event **count** identical (no duplicate or lost events from kill).
9. **Fail criterion:** any channel diverges, or journal order/count differs. Failing Q-4 blocks release (G-level, non-negotiable).

**Test file:** `test/scenario/scenario-1-determinism.test.ts`. Runs in CI on `ubuntu-latest` 2-core. Repeats 5 times per CI run (statistical hedge against rare non-determinism); all 5 must pass.

---

## 6. MCP tool surface contract

**Amended by ADR 0003** (state-machine scope + 2-tool MCP surface + wrapper-owns-auth doctrine; 2026-04-18). The original 7-tool/one-tool-per-signal-type design was drift — it encoded request-response framing and actor partitioning at the transport boundary, which conflicts with foreman's role as state-machine storage. The amended surface is:

Foreman exposes **2 MCP tools**, namespace-prefixed `foreman__*` (D-19).

| Tool name         | Kind  | Purpose                                                                                                                                                                                                                                                                                                   |
| ----------------- | ----- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `foreman__status` | Read  | Project current state to a read model. No side effects. Input: `{ nodeId }`. Output: `WireResponse<StatusPayload>` (see §8.2 + §8.4).                                                                                                                                                                     |
| `foreman__step`   | Write | Polymorphic advance — one of `{initialize, plan, work, eval, resume, retry, token_request}` payload variants (see §8.1 internal signal catalog + §8.4 per-signal schemas). Exactly one semantic transition per call. Input is a discriminated union keyed on `type`. Output: `WireResponse<StepPayload>`. |

Each tool's input is a Zod-validated payload; output is `WireResponse<T>` (see §8.2). Zod schemas are the source of truth — MCP SDK `inputSchema` is generated from Zod. Per ADR 0003, the internal signal vocabulary (7 types: initialize/plan/work/eval/resume/retry/token_request + status read) is distinct from the MCP transport granularity (2 tools).

**Actor handling:** `step` payloads carry an optional `actor: 'agent'|'operator'|'harness'` field which foreman journals as metadata. Foreman does not enforce actor identity at any phase — that is the wrapper's concern (the MCP stdio client at Phase 1; the daemon at Phase 2a; etc.). See ADR 0003 §1 for the wrapper-owns-auth doctrine.

---

## 7. CLI command specifications

**Amended by ADR 0003.** Foreman exposes **5 CLI commands** (down from 6; `foreman resume` is subsumed by `foreman step --signal '{"type":"resume",...}'` per the 2-tool MCP surface):

| Command                         | Purpose                                                                                                                                                                                               | Exit codes                                                                                                                                                                                             |
| ------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `foreman init`                  | Create `.foreman/` in CWD (checkpoints.db + config.json). Idempotent.                                                                                                                                 | **0** success (new or already-exists — aligns with architecture §7.1 "safe to re-run"); **1** error (unwritable dir, corrupt existing `.foreman/`); `--force` flag required to overwrite corrupt state |
| `foreman status [--node <id>]`  | Read current state; 0 if all nodes pass or idle, non-zero if blocked                                                                                                                                  | 0 ok; 10 has-active-block; 20 stale nodes present; 1 error                                                                                                                                             |
| `foreman step --signal <json>`  | Submit a polymorphic step signal as JSON. Accepts `type: 'initialize'\|'plan'\|'work'\|'evaluate'\|'resume'\|'retry'\|'token_request'` (internal signal catalog per §8.1). Foreman dispatches internally. | 0 success (any outcome); 1 validation error; 2 unknown nodeId                                                                                                                                    |
| `foreman inspect [--node <id>]` | Dump full state + last N journal events as JSON                                                                                                                                                       | 0 always (read-only)                                                                                                                                                                                   |
| `foreman validate-playbook <path>` | Zod-parse a playbook YAML file; report errors                                                                                                                                                      | 0 valid; 1 invalid (prints Zod issues)                                                                                                                                                                 |

Ergonomic CLI wrappers (e.g., `foreman plan --proposal <file>`) may be added as **UX sugar** on top of the single `step` subcommand. They are convenience layers — not separate MCP tools. Block resolution: `foreman step --signal '{"type":"resume","nodeId":"n1","payload":{"action":"approve","note":"..."}}'`. Capped restart: `foreman step --signal '{"type":"retry","nodeId":"n1","payload":{}}'` (α-only; no `reason` field at M3 per ADR 0003 §2.5).

**Output format:** JSON by default on stdout (`--json` is implicit). Human-readable output is a Phase 2a concern (no `--human` flag at Phase 1).

**Global flags:** `--version`, `--help`, `--foreman-dir <path>` (override `.foreman/` discovery).

### 7.1 Config resolution (D-8 materialized)

Order (CLI flags > env > project config > built-in defaults, per D-8):

1. **CLI flags** — `--foreman-dir <path>`, others per-command
2. **Environment variables** — `FOREMAN_*` prefix:
   - `FOREMAN_DIR` — absolute path to `.foreman/`, overrides walk-up discovery
   - `FOREMAN_LOG_LEVEL` — reserved for Phase 2a (ignored in Phase 1; stdio has no structured logging yet)
   - `FOREMAN_CHECKPOINT_PATH` — override `.foreman/checkpoints.db` location
3. **Project config file** — `.foreman/config.json`, Zod-validated at load:
   ```typescript
   const Config = z.object({
     projectId: z.string().optional(), // Gate 6 hedge; defaults to directory-path hash
     maxRetries: z.number().int().positive().default(3),
     ttlHours: z.number().positive().default(24),
     reserved: z.record(z.unknown()).optional(), // Phase 2a+ fields land here
   });
   ```
4. **Built-in defaults** — the `.default(...)` values in the Zod schema above

**Reference implementation sketch** (`packages/solo/src/config/resolve.ts`):

```typescript
export function resolveConfig(argv: string[], env: NodeJS.ProcessEnv): Config {
  const fileConfig = loadProjectConfig(); // reads .foreman/config.json or {}
  const envConfig = readForemanEnv(env); // picks FOREMAN_* keys
  const cliConfig = parseArgv(argv); // picks --flag values
  return Config.parse({ ...fileConfig, ...envConfig, ...cliConfig }); // precedence: later wins
}
```

Config resolution is tested (TDD per §5.1) with unit tests covering each precedence tier.

**`resolveConfig()` failure paths** — `.foreman/config.json` malformed (invalid JSON syntax) is caught by the loader and wrapped as `ValidationError { code: "SCHEMA_VALIDATION_FAILED", message: "config.json is not valid JSON", details: { file: ".foreman/config.json", parseError: <SyntaxError.message> } }`. Never surfaces as a raw `SyntaxError` stack trace to the operator.

### 7.2 Output format and unhandled error presentation

**Normal output (success + expected errors):**

- All CLI commands write to **stdout** as a single JSON object matching `WireResponse<T>` (§8.2)
- Exit code encodes outcome (per §7 table)
- Nothing else on stdout — pipes to `jq` cleanly

**Unhandled errors (unexpected crashes):**

- Any uncaught exception, OOM, or assertion failure reaches the top-level crash handler in `packages/solo/src/index.ts`
- Handler writes a **single JSON line** to **stderr** (not stdout — preserves stdout for normal output consumers):
  ```json
  {
    "ts": "2026-04-17T14:00:00Z",
    "level": "fatal",
    "code": "UNHANDLED_EXCEPTION",
    "message": "<error.message>",
    "details": {
      "stack": "<error.stack>",
      "pid": 12345,
      "nodeId": "<if-known>"
    }
  }
  ```
- Exit code: **99** (unhandled — distinct from 1 which is expected error)
- No raw stack trace dump on stderr; the JSON line's `details.stack` carries it for diagnostic tools
- `process.on('uncaughtException', ...)` + `process.on('unhandledRejection', ...)` both register this handler

**Why JSON on stderr not plain text:** operators ingest foreman's stderr in dogfooding setups + future Phase 2a daemon log pipelines. Structured format enables grep/jq without special parsers and forward-compatibility when Phase 2a adds pino logging.

---

## 8. Signal type catalog (Zod source of truth)

The sketches below are **directional, not final**. Source of truth is `packages/solo/src/types/signal.ts` once M1 ships; any divergence between this section and the M1 schema files is resolved by updating this section, not the code. During M1 drafting, implementation must satisfy the sketches' type discrimination + required fields; optional fields and Zod refinements may expand.

### 8.1 Signal envelope

**Amended by ADR 0003:** The signal enum below is the **internal vocabulary** for state transitions and reads. It is distinct from the MCP transport granularity (2 tools: `foreman__status` + `foreman__step`; see §6). The 7 write variants (initialize/plan/work/evaluate/resume/retry/token_request) are delivered via `foreman__step`; the 1 read variant (status) is delivered via `foreman__status`. MCP tool names are transport affordances; the semantic boundary per foundations D-21 is validated signal + graph channel + journaled event.

```typescript
const Signal = z.object({
  type: z.enum([
    'initialize',
    'plan',
    'work',
    'evaluate',
    'resume',
    'retry', // exhausted-state restart; α-only per ADR 0003 §2.5
    'token_request',
    'status',
  ]),
  nodeId: z.string(),
  payload: z.unknown(), // discriminated per type
});
```

### 8.2 Wire response envelope

```typescript
const WireResponse = <T>(data: z.ZodType<T>) =>
  z.discriminatedUnion('ok', [
    z.object({ ok: z.literal(true), data }),
    z.object({
      ok: z.literal(false),
      error: z.object({
        code: ErrorCode,
        message: z.string(),
        details: z.unknown().optional(),
      }),
    }),
  ]);
```

### 8.3 Error code starter catalog (M1 stub; full sweep at M6)

The M1 `docs/error-codes.md` stub ships with these 7 codes. They map 1:1 to architecture.md §8.3 table rows and are the authoritative Phase 1 starter set. M6 sweep adds any new codes introduced during M2–M5 and locks completeness per G7.

| Code                          | Class                 | When thrown                                                                                                                                                                                                                                                                                                                                                                | Fix                                                                                                                                                                                                                                          |
| ----------------------------- | --------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SCHEMA_VALIDATION_FAILED`    | `ValidationError`     | Signal did not match Zod schema (top-level or payload)                                                                                                                                                                                                                                                                                                                     | Check signal shape against `packages/solo/src/types/signal.ts`                                                                                                                                                                               |
| ~~`BRIEF_KIND_MISSING`~~      | ~~`ValidationError`~~ | **Removed M3 Batch A** (proposal §3.10): never thrown by production code — Zod rejection surfaces as `SCHEMA_VALIDATION_FAILED` with `details.issues`. Strike-through kept for history.                                                                                                                                                                                    | —                                                                                                                                                                                                                                            |
| `COMPOSITE_CHILD_NOT_READY`   | `ValidationError`     | (added M3 Batch A; trigger amended v0.7 per Option B decision 2026-04-18) Composite parent's `step(type:'work')` arrives before all children reach an evaluated state; enforced by `handlers/step.ts` work-signal sequence-validity pre-check at Batch D. The composite parent's work phase IS the consolidation of children's outputs — children must be evaluated first. | Wait for all children to have a non-null evaluation; then the agent synthesizes the consolidated `WorkReport` (using `brief.playbook.phases.work` prompt) and submits `step(type:'work', nodeId: parent, payload: {report, toolCalls:[]})` |
| `NODE_NOT_FOUND`              | `ValidationError`     | Signal targeted an unknown `nodeId`                                                                                                                                                                                                                                                                                                                                        | Verify node was initialized via a prior `step(type:'initialize')` signal (ADR 0003 2-tool surface)                                                                                                                                           |
| `OPERATOR_APPROVAL_REQUIRED`  | `SecurityError`       | **Sequence-validity check (ADR 0003 §4):** Irreversible ToolCall submitted without a prior journaled `resume` event carrying `action: 'approve'` for this nodeId. Foreman checks the event exists; it does **not** check caller identity — that is the wrapper's concern.                                                                                                  | `foreman step --signal '{"type":"resume","nodeId":"<id>","payload":{"action":"approve","note":"..."}}'` (or `"action":"override"` with `justification`)                                                                                      |
| `PREFLIGHT_SECRET_DETECTED`   | `SecurityError`       | Work report contains a credential pattern (AWS key, bearer token, etc.) matched by preflight scan                                                                                                                                                                                                                                                                          | Remove secret from report; use `foreman step --signal '{"type":"token_request",...}'` for scoped creds (available at M4; M3 rejects)                                                                                                         |
| `CHECKPOINTER_WRITE_FAILED`   | `StorageError`        | SQLite/LangGraph checkpointer write error (disk full, FS unavailable)                                                                                                                                                                                                                                                                                                      | Check disk; foreman auto-retries once before surfacing                                                                                                                                                                                       |
| `JOURNAL_CORRUPTION_DETECTED` | `StorageError`        | Journal event deserialization failed                                                                                                                                                                                                                                                                                                                                       | Inspect via `foreman inspect`; restore from prior checkpoint or `foreman init --force`                                                                                                                                                       |

Full 100% catalog enforcement is G7 (M6 sweep). M1 ships these 7 + test asserts every thrown `BaseError` uses one of them.

### 8.4 Per-signal schemas (summary)

| Signal          | Payload sketch                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| --------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `initialize`    | `{ kind: "terminal"\|"composite", brief: Brief }`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| `plan`          | `{ kind: "terminal", proposal: string }` OR `{ kind: "composite", children: Brief[] }`                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| `work`          | `{ report: string, toolCalls: ToolCall[] }` where `ToolCall = { name: string, args: unknown, reversibility: "reversible"\|"irreversible"\|"unknown" }`. **For composite parents (Option B, v0.7):** `report` is the agent's consolidation of children's outputs, synthesized using `brief.playbook.phases.work` as the prompt (D-18). Handler pre-check gates on children-readiness (COMPOSITE_CHILD_NOT_READY). Composite work is the consolidation phase — not "do the work" directly.                                                                                           |
| `evaluate`      | `{ findings: Finding[], meetsCriteria: boolean }` where `Finding = { severity: "info"\|"warning"\|"critical", detail: string }`. **Evaluate node is kind-agnostic (v0.7)** — same handling for terminal and composite; it assesses whatever `WorkReport` came in (for composite, that report is the agent's consolidation).                                                                                                                                                                                                                                                             |
| `resume`        | `{ action: "approve", note?: string } \| { action: "override", justification: string } \| { action: "fail", reason: string }`. **Valid only from `block` state (ADR 0003 §4).** Routing (v0.7 amended per LangGraph 1.x docs): handler sends `new Command({resume: payload})` to `graph.invoke` (invoke-input Commands accept only `resume`). The evaluate node owns routing on post-resume re-run — `approve`/`override` return a plain state-update + static `n_evaluate → END` edge fires; `fail` returns `new Command({update: {journal}, goto: END})` (node-return Command shape). |
| `retry`         | `{}` (empty payload). **Valid only from `exhausted` state (ADR 0003 §2.5, §4).** α-only (no target discriminator); no `reason` field at M3 (v0.3 YAGNI — forward-compat for future ADR). Control signal: handler appends a `retry_reset` journal marker (`data: { nodeId }`), clears the `evaluation` channel to `null`, returns success. Agent then sends `type: 'plan'` as the next step.                                                                                                                                                                                             |
| `token_request` | `{ scope: string, ttl: number }` (ttl in seconds, positive int). **M3/M4 behavior:** handler rejects with `SCHEMA_VALIDATION_FAILED` + `details: { reason: 'token_request deferred to Phase 2a' }`. M4 Batch C1 (unstub + pass-through vending) was dropped 2026-04-18 per operator + GAN — foreman as infrastructure does not call external APIs, and pass-through provides zero security value vs the agent reading env directly (see ADR 0007). `TokenVendingPort` materializes at Phase 2a with Vault/HSM. The signal stays in `StepSignalSchema` for forward-compat.                                                      |
| `status`        | `{}` (empty payload; no writes, just read)                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |

---

## 9. Checkpoint schema

LangGraph's `SqliteSaver` owns the SQLite schema internally. Foreman's contribution:

| Channel      | Type                        | Notes                                                                          |
| ------------ | --------------------------- | ------------------------------------------------------------------------------ |
| `nodeId`     | string                      | Set at `init`, immutable                                                       |
| `kind`       | `"terminal" \| "composite"` | Set at `init`, immutable (I-2)                                                 |
| `brief`      | `Brief` (Zod)               | Immutable; includes `projectId: string` (Gate 6 hedge) + `playbook: Playbook` (D-18) |
| `plan`       | `Plan \| null`              | Latest; replace reducer                                                        |
| `work`       | `WorkReport \| null`        | Latest; replace reducer                                                        |
| `evaluation` | `Evaluation \| null`        | Latest; replace reducer                                                        |
| `journal`    | `JournalEvent[]`            | Append reducer (event sourcing)                                                |

**Thread ID**: composite `${projectId}:${nodeId}` (one thread per node — including each composite child, per ADR 0004 Option B "independent threads"). **Checkpoint namespace**: empty string `''` at the top-level graph; non-empty `checkpoint_ns` is reserved for LangGraph-managed subgraph routing and is NOT used for composite children at M3+ (children run in independent top-level threads, not subgraphs). Per-node isolation is via the composite thread_id, not via namespace. **Amended from the legacy "thread=projectId; ns=nodeId" model by ADR 0002 (LangGraph 1.x adoption); composite independence locked by ADR 0004 Option B (2026-04-18).**

**Brief schema includes `projectId: string`** — the design hedge from Gate 6. Phase 1 sets it from `.foreman/`'s absolute path hash; Phase 2+ hosted service replaces with user-assigned project ID.

**Journal event timestamp format (MUST be UTC):**

```typescript
const JournalEvent = z.object({
  ts: z.string().datetime({ offset: false }), // ISO 8601 with explicit 'Z' suffix; reject +HH:MM offsets
  type: z.enum(['initialize', 'plan', 'work', 'evaluate', 'block', 'resume', 'retry_reset']),
  nodeId: z.string(),
  actor: z.enum(['agent', 'operator', 'harness']),
  data: z.unknown(),
});
```

All journal events are timestamped in UTC (`Z` suffix, no offset). This is non-negotiable for:

- **P-5 determinism:** cross-timezone dogfooding (developer in one TZ, CI in another) produces the same journal
- **Q-4 determinism test:** timestamp comparison would fail non-deterministically with locale-relative timestamps
- **Phase 2a hosted service:** server and clients in arbitrary timezones must agree on event order

Zod rejects strings with explicit offsets at schema validation. `new Date().toISOString()` produces `Z` suffix by default — use it. Never `Date.toString()` or custom formatting.

---

## 10. Example playbook contracts

**Canonical format: YAML.** Playbooks ship as `examples/<name>/playbook.yaml`. Agents read the file, parse YAML, Zod-validate against `Playbook` schema, embed the validated object in `brief.playbook` on the `initialize` signal.

**D-18 boundary clarification:** D-18 says "foreman never reads playbook files" — this constrains **runtime** (graph nodes, handlers, transport). The `foreman validate-playbook <path>` CLI command is a **user-invoked dev tool**, not part of the harness operating flow. It reads a file to help playbook authors; it never runs during a live agent session. This is consistent with D-18 — runtime playbook loading remains signal-injected by the agent.

### 10.1 `tdd-feature`

Drives a test-first feature implementation. `examples/tdd-feature/playbook.yaml`:

```yaml
name: tdd-feature
phases:
  plan: |
    Propose a test list for the feature: list 3–7 behaviors that must be verified.
    For each: test file path, scenario name, input/output sketch.
  work: |
    Implement the feature. Write tests from the plan first, then implementation.
    Ensure tests fail before green.
    Return: paths of test files written, paths of source files touched, summary.
  evaluate: |
    Run the test suite. Report pass/fail per planned test.
    Report any regression in existing tests.
    meetsCriteria: true iff all planned tests pass and no regressions.
scope:
  readOnly: false
  paths: ['src/**', 'test/**']
```

### 10.2 `refactor-extract`

Extracts a function/module without behavior change. `examples/refactor-extract/playbook.yaml`:

```yaml
name: refactor-extract
phases:
  plan: |
    Identify extraction target (function or module).
    Plan the new location and import path.
    Enumerate callers that will be updated.
  work: |
    Move code to new location; update all call sites.
    Do not change behavior (no logic edits; only moves + imports).
  evaluate: |
    Run full test suite + `git diff` review.
    meetsCriteria: true iff all tests pass AND diff shows moves+imports only, no logic changes.
scope:
  readOnly: false
  paths: ['src/**'] # no test/ edits — tests prove no behavior change
```

Both playbooks ship in `examples/` and are copy-pasteable as starters.

---

## 11. Agent transcript format

Scenario tests must capture the full agent ↔ foreman exchange for reproducibility (P-5). Format: **newline-delimited JSON** (`.jsonl`), one entry per signal.

```jsonl
{"ts":"2026-04-17T14:00:00Z","direction":"agent→foreman","signal":{"type":"initialize","nodeId":"n-1","payload":{...}}}
{"ts":"2026-04-17T14:00:00.120Z","direction":"foreman→agent","response":{"ok":true,"data":{...}}}
{"ts":"2026-04-17T14:00:05Z","direction":"agent→foreman","signal":{"type":"plan","nodeId":"n-1","payload":{...}}}
```

**Per-scenario artifact:** `test/scenario/<name>/transcript.jsonl` committed to the repo. Replay-deterministic given identical brief + agent prompts (P-5).

**Recording mechanism:**

- Scenario tests run via `pnpm test:scenario` (default) — compares live agent exchanges against committed `transcript.jsonl` fixtures; fails on divergence
- `pnpm test:scenario:record` — vitest custom runner mode that writes new `transcript.jsonl` files instead of comparing (use when intentionally updating scenarios)
- Capture hook: a vitest `beforeAll` wraps the MCP adapter's `send`/`receive` methods with a recorder that appends each signal + response to a per-scenario `.jsonl` stream
- Timestamps in transcripts use ISO 8601 (UTC) but are **excluded from replay comparison** (non-deterministic by design); only `direction`, `signal/response` structure, and `nodeId` are compared
- Fixture source: tests construct `Brief` + `Skill` + agent-prompt objects inline; recorder captures the MCP conversation they produce
- `.jsonl` files are version-controlled; changes require explanation in CHANGELOG

---

## 12. Dependency inventory + pinning policy

Exact-pinned (no caret, no tilde) per R-1 and D-7:

| Dep                                      | Version policy                                     | Layer               | Notes                                |
| ---------------------------------------- | -------------------------------------------------- | ------------------- | ------------------------------------ |
| `@langchain/langgraph`                   | Exact                                              | `graph/` only       | R-1; upgrades require ADR            |
| `@langchain/langgraph-checkpoint-sqlite` | Exact                                              | `graph/` only       | Same discipline as langgraph         |
| `@modelcontextprotocol/sdk`              | Exact                                              | `mcp/` only         | D-7; pre-1.0 churn                   |
| `zod`                                    | Exact                                              | `types/`, `errors/` | Schema contract; bumps require regen |
| `commander`                              | Caret OK                                           | `cli/` only         | Stable API                           |
| `better-sqlite3`                         | Exact (transitive via langgraph-checkpoint-sqlite) | N/A                 | R-15                                 |

`pnpm-lock.yaml` **committed** and integrity-hashed. CI runs `pnpm install --frozen-lockfile`. `.npmrc` disables install scripts for `@langchain/*` and `@modelcontextprotocol/*` (R-15).

Upgrade process:

- **Security patches:** PR + passing CI + changelog entry (R-1 fast-path)
- **All other bumps:** ADR required only if `graph/` or `mcp/` code changes needed; else PR + changelog

---

## 13. CI pipeline definition

Runs on: PR, push to main, tag push.

| Job                                                                                                                                                                                                           | PR                  | Push-main   | Tag          |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------- | ----------- | ------------ |
| `install` (`pnpm install --frozen-lockfile`)                                                                                                                                                                  | ✓                   | ✓           | ✓            |
| `typecheck` (`pnpm tsc --noEmit`)                                                                                                                                                                             | ✓                   | ✓           | ✓            |
| `lint` (`pnpm eslint . && pnpm dependency-cruiser src`)                                                                                                                                                       | ✓                   | ✓           | ✓            |
| `format:check` (`pnpm prettier --check`)                                                                                                                                                                      | ✓                   | ✓           | ✓            |
| `test:unit`                                                                                                                                                                                                   | ✓                   | ✓           | ✓            |
| `test:integration`                                                                                                                                                                                            | ✓                   | ✓           | ✓            |
| `test:contract`                                                                                                                                                                                               | ✓                   | ✓           | ✓            |
| `test:scenario`                                                                                                                                                                                               | ✓                   | ✓           | ✓            |
| `test:load` (`--pool=forks`, Q-6a/b)                                                                                                                                                                          | ✓ (tagged PRs only) | ✓           | ✓            |
| `test:perf:handshake` (Q-11a p95 < 800 ms on 10 cold spawns; fails CI on regression)                                                                                                                          | —                   | ✓           | ✓            |
| `test:perf:domain` (Q-11b characterization only — records p50/p95/p99 to PR summary, never fails)                                                                                                             | —                   | ✓           | ✓            |
| `coverage` (≥ 80% branch on graph/+handlers/ per Q-3)                                                                                                                                                         | ✓                   | ✓           | ✓            |
| `security:audit` (`pnpm audit --production`)                                                                                                                                                                  | ✓                   | ✓           | ✓            |
| `security:gitleaks` (pre-commit + CI)                                                                                                                                                                         | ✓                   | ✓           | ✓            |
| `security:no-network` — lint job rejects `fetch`/`axios`/`@sentry/*`/`posthog-js`/`mixpanel`/`@amplitude/*` imports in `packages/solo/src/*`; enforces P-7 (no telemetry)                                     | ✓                   | ✓           | ✓            |
| `licenses:check` — `pnpm licenses list --prod --json \| node scripts/check-licenses.mjs` rejects non-Apache-compatible deps (MIT, BSD, ISC, Apache-2.0 allowed; GPL/AGPL/proprietary rejected); enforces P-10 | ✓                   | ✓           | ✓            |
| `mcp:smoke` (3 agents: Claude Code, Copilot CLI, Cursor; **retry: 2** on network-dependent agent install steps; flake budget documented per-agent in `test/integration/mcp-clients/FLAKES.md`)                | —                   | ✓ (nightly) | ✓            |
| `publish` (`pnpm pack` + `npm publish --access public`)                                                                                                                                                       | —                   | —           | ✓ (tag only) |

All PR jobs are required checks for merge. Nightly main runs catch flakes + MCP smoke drift.

**`pnpm audit` false-positive policy:** real-world CVE scanners flag false positives regularly. Rather than skipping audit on red, the repo ships `.pnpmauditrc.json` (M0 deliverable) with an `ignore` list format:

```json
{
  "ignore": [
    {
      "id": "CVE-2025-XXXXX",
      "package": "dep-name",
      "reason": "Not exploitable in our usage — [1-line rationale]",
      "issue": "https://github.com/foreman-lab/foreman/issues/NN",
      "expires": "2026-10-17"
    }
  ]
}
```

- Every entry requires a GH issue link and an expiry date (default 6 months; rejected CVEs must be re-reviewed)
- Entries without `issue` or `expires` fail CI config-lint
- `security:audit` job reads this file; anything not in the ignore list blocks the build
- Weekly cron job checks for expired entries and opens follow-up issues

**`test:perf:handshake` script specification:**

```typescript
// test/perf/handshake.test.ts — sketch
for (let i = 0; i < 10; i++) {
  const t0 = performance.now();
  const child = spawn("node", ["dist/index.js", "mcp"], { stdio: "pipe" });
  child.stdin.write(JSON.stringify({ jsonrpc: "2.0", id: 1, method: "initialize", params: {...} }) + "\n");
  await firstJsonResponse(child.stdout);  // reads one JSON-RPC response
  samples.push(performance.now() - t0);
  child.kill();
}
const p95 = percentile(samples.sort((a,b) => a-b), 0.95);
expect(p95).toBeLessThan(800);  // Q-11a gate
```

- Runs in a dedicated CI step (not inside vitest's default pool — needs process isolation)
- Reference env: `ubuntu-latest` 2-core (§14)
- Q-11b uses the same harness but measures `tools/call foreman__step` (with `type=initialize` payload) latency after handshake; reports-only, no assertion (characterization baseline for Phase 2a promotion)

---

## 14. Reference machine

Benchmark NFRs (Q-1, Q-2, Q-11a/Q-11b) measured on:

- **Platform:** GitHub Actions `ubuntu-latest` (currently Ubuntu 22.04)
- **CPU:** 2-core x86_64 (GH Actions hosted runner spec)
- **Memory:** 7 GB (GH Actions hosted runner spec)
- **Node.js:** 20 LTS (exact version from `engines.node`)
- **Storage:** Workspace on SSD (GH Actions default)

Developer laptops will be faster — treat the reference as a floor, not a ceiling. Numbers on local dev do not unblock a failing CI perf gate.

---

## 15. Dogfooding protocol

### 15.1 Pre-alpha (M0–M2)

No dogfooding. Work tracked in git commits + local `DEVLOG.md`.

### 15.2 Solo alpha (M3 onwards)

- **Enter dogfooding:** at M3 close, run `foreman init` in the foreman repo itself (foreman project `.foreman/` lives alongside `docs/`, `packages/`, etc.)
- **M3 mode — raw signals** (per §2.5): tasks tracked via `foreman init` + hand-crafted `brief` objects (no `playbook` field yet), `foreman step` submits raw `plan` / `work` / `evaluate` signals. Agent authors prompts manually.
- **M5 close — playbook-upgrade:** post-2026-04-18 domain-blind pivot (ADR 0007) relocated prompt rendering to the agent layer, so the playbook-driven upgrade slips from M4 to M5. Once `examples/tdd-feature/` + `examples/refactor-extract/` + `agent-setup.md` + `foreman validate-playbook` CLI ship, M5/M6 tasks move to playbook-driven orchestration with `brief.playbook` pointing at the inline `Playbook` object and the agent resolving `phases[phase]` placeholders locally.
- **Capture friction:** every rough edge found while using foreman-on-foreman is logged in `docs/dogfooding-log.md` with:
  - Date
  - Symptom (one line)
  - Suspected cause
  - Fix (link to commit or "deferred to Phase 2a")
- **Boundary:** agent outputs are untrusted until vitest says green (§5.8). Dogfooding is a canary, not a test gate.

### 15.3 Backlog discipline

- `BACKLOG.md` at repo root tracks "not Phase 1" scope that surfaces during dogfooding
- Only reviewed at milestone boundaries
- Entries get one-line status: `[P1]` (Phase 1, add if time), `[P2A]` (Phase 2a), `[NO]` (rejected)

---

## Appendix A: Traceability (spec → architecture → foundations/roadmap)

| Spec section             | Architecture source                                   | Foundations/Roadmap source           |
| ------------------------ | ----------------------------------------------------- | ------------------------------------ |
| §2 Methodology           | —                                                     | — (first-time decisions)             |
| §3 Milestones            | —                                                     | Roadmap Gate 2 scope                 |
| §4 Ship criteria         | architecture §8.4                                     | Roadmap Gate 2 + Gate 6              |
| §5 Testing methodology   | architecture §8.1 (NFRs Q-1..Q-11b)                   | D-10 (vitest)                        |
| §6 MCP tool surface      | architecture §3                                       | D-7, D-19                            |
| §7 CLI commands          | architecture §5.2 handlers row, §7.3 runtime topology | D-19                                 |
| §8 Signal catalog        | architecture §3                                       | D-3 (Zod), D-18 (playbook injection) |
| §9 Checkpoint schema     | architecture §5, §6                                   | D-16 (LangGraph), Gate 6 (projectId) |
| §10 Example playbooks    | architecture §7                                       | D-18                                 |
| §11 Agent transcript     | architecture §6 scenario tests                        | P-5 (reproducibility)                |
| §12 Dependency inventory | architecture §10 (tier variations), §8.2 R-1/R-15     | D-7                                  |
| §13 CI pipeline          | architecture §8.1 (Q-gates)                           | D-12                                 |
| §14 Reference machine    | architecture §8.1 (Q-6, Q-11a)                        | —                                    |
| §15 Dogfooding           | architecture.md (implicit)                            | P-9, P-4                             |

---

**End of document.** Phase 1 production baseline locked 2026-04-17 after ~20 rounds of Codex + Copilot debate + 14-item walk-through + 11-item production-fix pass + final pre-launch audit. Revisions require an ADR per `foundations.md` Drift Prevention rules.
