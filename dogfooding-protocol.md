---
status: Draft v1 — M3 Solo Alpha close; raw-signal dogfooding
owner: Operator (xuzhijie)
baselined: 2026-04-18
grounding: spec §2.5 (dogfooding boundary), §15.2 (log + canary-not-gate), m3-development-proposal D-M3-13
---

# Foreman Dogfooding Protocol — M3 Solo Alpha

## 1. Purpose

Per spec §2.5 + §15.2: dogfooding starts at the end of M3. Friction surfaces real UX bugs while they are still cheap to fix. This document is the operator walkthrough for raw-signal dogfooding — the 2-tool MCP surface + 3-subcommand operator CLI (+ hidden `mcp` run-mode) shipped in M3. Post-2026-04-18 domain-blind pivot (ADR 0007): playbook-based prompt assembly is now **external to foreman** (agents render prompts); playbook-driven dogfooding upgrade slipped from M4 to **M5 close** (when example playbooks + `agent-setup.md` ship).

**Boundary (spec §15.2):** vitest remains the **independent test oracle**. Foreman-in-use is a **canary, not a gate**. If foreman breaks during dogfooding, file a bug against foreman; do not soften the test suite to accommodate the bug.

## 2. Prerequisites

- Node.js ≥ 20, pnpm installed
- Repo checked out; run `pnpm install --frozen-lockfile`
- `pnpm build` to produce `dist/cli/bin.js` (or invoke via `tsx`)
- No LLM credentials, no external services. Foreman is pure infrastructure (P-1).

## 3. Setup

```bash
cd /path/to/foreman-using-repo
foreman init                    # creates .foreman/ + primes checkpoints.db + config.json
foreman status --node <id>      # reads current state (empty → NODE_NOT_FOUND expected)
```

The `.foreman/` directory lives alongside your work (same cwd as the agent's edits). Foreman has no global state — each repo you use foreman in gets its own `.foreman/`.

## 4. The 2-tool MCP surface + CLI subcommands

Agents send MCP calls; operators send CLI calls. Both dispatch through the same handler.

**MCP (2 tools):** `foreman__status` (read) + `foreman__step` (polymorphic write over 7 signal types).

**CLI (3 operator subcommands shipped at M3):** `foreman {init, status, step}`. Plus one hidden run-mode `mcp` (used by MCP stdio clients; not operator-facing).

**CLI subcommands deferred to M4/M5** (mentioned in spec §7 but NOT yet registered at M3):

- `foreman inspect` — dump full state + last N journal events (deferred to **M5** post-2026-04-18 domain-blind pivot; was planned M4 alongside the now-reverted `PromptAssemblerPort`)
- `foreman validate-playbook <path>` — Zod-parse a playbook YAML file (deferred to **M5**; transport-only CLI, no runtime coupling to any port — see ADR 0005 playbook file format + ADR 0007 domain-blind core)

Attempts to run deferred subcommands at M3 will fall through commander's unknown-command handling.

## 5. Signal envelope

Every step signal is a JSON object:

```json
{
  "type": "initialize|plan|work|eval|resume|retry|token_request",
  "nodeId": "string",
  "payload": {
    /* variant-specific */
  }
}
```

See spec §8.4 for each variant's payload schema. Deep Zod validation happens in the handler — malformed JSON / missing fields / wrong payload shape → `SCHEMA_VALIDATION_FAILED` envelope with `details.issues` (Zod issues list).

## 6. Happy-path walkthrough (terminal node, 10 minutes)

Each command below expects `cwd` to be a foreman-initialized repo. `--signal` accepts literal JSON or `-` (read from stdin). Exit codes per spec §7: `0` success, `1` validation error, `2` unknown nodeId.

```bash
# (a) initialize a terminal node
foreman step --signal '{
  "type": "initialize",
  "nodeId": "task-1",
  "payload": {
    "brief": {
      "projectId": "default",
      "nodeId": "task-1",
      "kind": "terminal",
      "playbook": {
        "name": "raw",
        "phases": {
          "plan": "Outline steps",
          "work": "Execute the plan",
          "evaluate": "Assert completeness"
        }
      },
      "goal": "Add a foo helper"
    }
  }
}'
# Expected: {"ok":true,"data":{"nodeId":"task-1","type":"initialize"}}

# (b) submit a plan
foreman step --signal '{
  "type": "plan",
  "nodeId": "task-1",
  "payload": { "kind": "terminal", "proposal": "Add src/foo.ts exporting const foo = 42" }
}'

# (c) submit a work report (agent-authored)
foreman step --signal '{
  "type": "work",
  "nodeId": "task-1",
  "payload": { "report": "Wrote src/foo.ts", "toolCalls": [] }
}'

# (d) evaluate — passing
foreman step --signal '{
  "type": "eval",
  "nodeId": "task-1",
  "payload": { "findings": [], "meetsCriteria": true }
}'

# (e) read the projection
foreman status --node task-1
# Expected: journalCount=4 (init+plan+work+eval), evaluation.meetsCriteria=true
```

**D-M3-16 trap:** `initialize` payload has **no `kind` field** at the payload level. `brief.kind` is the single source of truth. Sending `payload: { kind: "terminal", brief: {...} }` is rejected with `SCHEMA_VALIDATION_FAILED` (strict Zod).

## 7. Block / resume walkthrough

Agent reports a critical finding → interrupt fires → operator resolves.

```bash
# (a) initialize + plan + work as above, then eval with a critical finding
foreman step --signal '{
  "type": "eval",
  "nodeId": "task-1",
  "payload": {
    "findings": [{"severity": "critical", "detail": "Missing license header"}],
    "meetsCriteria": false
  }
}'
# Graph pauses at n_evaluate (interrupt fired). status will show next=["n_evaluate"].

# (b-approve) operator approves — graph unpauses, eval events written
foreman step --signal '{
  "type": "resume",
  "nodeId": "task-1",
  "payload": { "action": "approve", "note": "license header added in follow-up" }
}'

# (b-override) operator overrides — same path as approve, requires justification
foreman step --signal '{
  "type": "resume",
  "nodeId": "task-1",
  "payload": { "action": "override", "justification": "accepting risk; tracked in Jira FOO-123" }
}'

# (b-fail) operator declares failure — evaluate node returns Command(goto:END)
foreman step --signal '{
  "type": "resume",
  "nodeId": "task-1",
  "payload": { "action": "fail", "reason": "not recoverable without rewrite" }
}'
```

**Traps:**

- Resume signal only valid while graph is paused at `n_evaluate`. A second resume returns `SCHEMA_VALIDATION_FAILED` with message "resume applies only when graph is paused at n_evaluate".
- `resume` is the signal name; internally the handler sends `new Command({resume: payload})` to `graph.invoke`. Per LangGraph 1.x docs, only `resume` is valid in invoke-input Commands — `goto`/`update` live in node-return Commands inside `evaluate.ts`.

## 8. Retry walkthrough (exhausted → retry → replan)

After `maxRetries` failed evals (default 3), outcome becomes `exhausted` and the operator can reset.

```bash
# After 3 failed evals, outcome is exhausted. Retry clears evaluation + writes marker.
foreman step --signal '{
  "type": "retry",
  "nodeId": "task-1",
  "payload": {}
}'
# Expected: ok:true. Journal gains a 'retry_reset' event (data:{nodeId}). Evaluation cleared.

# Agent re-plans + re-works; retry-count restarts from 0 (retry_reset marker awareness).
foreman step --signal '{"type":"plan","nodeId":"task-1","payload":{"kind":"terminal","proposal":"new approach..."}}'
```

**Trap:** `retry` payload is `{}` (empty). Adding `reason` or other fields → `SCHEMA_VALIDATION_FAILED` (α-only per v0.3 YAGNI; forward-compat to re-add).

## 9. Composite walkthrough (parent + children)

Composite parents decompose work into children. Each child is its own top-level thread (ADR 0004). Parent has a **work phase = consolidation** (Option B): agent reads children's outputs, synthesizes them, submits the parent's WorkReport.

```bash
# (a) initialize parent + 2 children
foreman step --signal '{"type":"initialize","nodeId":"parent","payload":{"brief":{...,"kind":"composite",...}}}'
foreman step --signal '{"type":"initialize","nodeId":"c1","payload":{"brief":{...,"kind":"terminal",...}}}'
foreman step --signal '{"type":"initialize","nodeId":"c2","payload":{"brief":{...,"kind":"terminal",...}}}'

# (b) parent plan — declare children
foreman step --signal '{
  "type": "plan",
  "nodeId": "parent",
  "payload": {
    "kind": "composite",
    "children": [briefC1, briefC2]
  }
}'

# (c) run each child to completion (init already done; plan → work → eval)
# ... same pattern as §6 for c1 and c2 ...

# (d) parent work — consolidation. Agent reads c1/c2 via foreman__status first,
#     then synthesizes using brief.playbook.phases.work as the prompt (D-18).
foreman step --signal '{
  "type": "work",
  "nodeId": "parent",
  "payload": { "report": "Consolidated: c1 delivered X, c2 delivered Y", "toolCalls": [] }
}'
# Pre-check: if any child lacks evaluation (or is paused), returns COMPOSITE_CHILD_NOT_READY
# with details {parentNodeId, childNodeId, childNext}.

# (e) parent eval — kind-agnostic; assesses the consolidated WorkReport
foreman step --signal '{"type":"eval","nodeId":"parent","payload":{...,"meetsCriteria":true}}'
```

**Grandchild (composite-within-composite):** same pattern applies recursively. Each level's parent consolidates its own children; the grandparent consolidates grandparent's children (which include the now-passed intermediate composite). Wrapper drives aggregation at every level (ADR 0004 §Grandchild-ownership).

**Trap:** composite parent `work` requires `plan.children[*].evaluation` non-null AND `snapshot.next` empty for each child. A child paused at `n_evaluate` (awaiting resume) is NOT ready.

## 10. token_request rejection at M3

```bash
foreman step --signal '{"type":"token_request","nodeId":"task-1","payload":{"scope":"read","ttl":3600}}'
# Expected (post-2026-04-18 pivot; M4 Batch C1 dropped to Phase 2a per ADR 0007):
# {"ok":false,"error":{"code":"SCHEMA_VALIDATION_FAILED",
#   "message":"token_request handler deferred to Phase 2a",
#   "details":{"reason":"token_request deferred to Phase 2a"}}}
```

The signal schema stays in `StepSignalSchema` for forward-compat; M4 Batch C1 (unstub + pass-through vending) was dropped 2026-04-18 to Phase 2a per operator + GAN — foreman as infrastructure does not call external APIs, and pass-through vending provides zero security value vs the agent reading env directly (see ADR 0007). `TokenVendingPort` materializes at Phase 2a when Vault/HSM provides real scoping + revocation + audit.

## 11. What passes at M3

- ✅ All 7 write signals accepted (or cleanly rejected with structured envelopes)
- ✅ `foreman__status` CQRS read; no side effects
- ✅ 2-tool MCP surface + 3-subcommand operator CLI (+ hidden `mcp` run-mode); `inspect` + `validate-playbook` deferred to M4/M5
- ✅ Interrupt / resume / retry semantics match spec §8.4
- ✅ Composite parents consolidate via work phase; readiness pre-check gates
- ✅ End-to-end demo (`pnpm demo`) green in CI

## 12. What's deferred

- `token_request` handler — Phase 2a when `TokenVendingPort` ships (M4 Batch C1 dropped 2026-04-18 per ADR 0007)
- TTL / stale-node detection — Phase 2a (v0.3 YAGNI)
- Playbook-based prompt assembly — **external to foreman** (post-2026-04-18 domain-blind pivot; see ADR 0007). Agents render prompts from raw `brief.playbook.phases[phase]`; M5 ships `agent-setup.md` + 2 example playbooks.
- Ergonomic CLI wrappers (`foreman plan --proposal`, etc.) — Phase 2a sugar layer
- Scenario test suite G4 — M5

## 13. Capturing friction → `docs/dogfooding-log.md`

Per spec §15.2 — log all friction encountered during dogfooding. Template:

```markdown
### YYYY-MM-DD — <one-line symptom>

**Context:** what I was trying to do.
**Expected:** what I thought would happen.
**Actual:** what happened (copy the envelope / stderr).
**Hypothesis:** suspected cause.
**Fix (if known):** or `TBD — file issue`.
**Related:** commit SHA, test file, bug ID.
```

Out-of-scope ideas / features discovered during dogfooding → `BACKLOG.md` instead of the log.

## 14. When foreman breaks

Per spec §15.2 boundary:

1. Do **not** edit the vitest suite to accommodate the bug.
2. File a bug (issue in the repo) with the log entry's Context / Expected / Actual / Hypothesis.
3. If the bug blocks the dogfooded task, work around it manually (inspect state directly with `sqlite3 .foreman/checkpoints.db` or ad-hoc `graph.getState` via a throwaway `tsx` script; `foreman inspect` is M4 scope per §4) and continue.
4. The bug is triaged + fixed in a normal commit with a regression test against the vitest suite.

**Canary, not gate** — this is the boundary that makes dogfooding a first-class signal without making it a blocker.

## 15. References

- `docs/phase-1-solo/spec.md` §2.5 (dogfooding decision), §7 (CLI), §8.1–§8.4 (signal catalog + schemas + error codes), §15 (dogfooding practice)
- `docs/phase-1-solo/m3-development-proposal.md` D-M3-13 (this doc is the deliverable); D-M3-14 (token_request M3 rejection); §3.2 (ADR trace)
- `docs/adr/0003-state-machine-scope-and-mcp-surface.md` (2-tool MCP + wrapper-owns-auth)
- `docs/adr/0004-composite-independent-threads.md` (per-thread composite model + Option B consolidation)
- `docs/foundations.md` P-1 (pure infrastructure), D-18 (playbook injection), D-21 (signal/channel/journal boundary)
