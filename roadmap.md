# Foreman Product Roadmap

**Status:** Locked baseline — 6 of 7 gates approved; 1 retracted (ship criteria moved to Phase 1 spec)
**Last updated:** 2026-04-16
**Governing principles P-1..P-10 and security requirements S-1..S-5:** see [`docs/foundations.md`](foundations.md). This roadmap does not duplicate them.
**Governance anchors:** P-9 (demand-driven, not schedule-driven) and P-10 (fully open source, hosted-service moat articulated by Phase 2 end).

Four tiers — Solo, Daemon, Team, Enterprise. Each tier extends the previous. Phase promotion is gated by concrete demand tripwires, not calendar dates.

---

## Approval log

| Gate  | Topic                                              | Status                 | Notes                                                                                                                                                                                                                                                                                             |
| ----- | -------------------------------------------------- | ---------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1     | Tier structure (Solo / Daemon / Team / Enterprise) | ✅ Approved 2026-04-16 |                                                                                                                                                                                                                                                                                                   |
| 2     | Phase 1 Solo scope — high-level                    | ✅ Approved 2026-04-16 | Revised via Copilot (scaffold cut; validate-skill/inspect/error-codes added; CLI subset; 2 examples)                                                                                                                                                                                              |
| ~~3~~ | ~~Ship criteria~~                                  | ⏭ Retracted           | Moved to future `docs/phase-1-solo/spec.md` (out of roadmap altitude)                                                                                                                                                                                                                             |
| 4     | Tripwire thresholds                                | ✅ Approved 2026-04-16 | Via Copilot pass (stars/npm dropped as vanity; language-tagged issues + dependent-repos + distinct authors adopted; 2-week cooldown)                                                                                                                                                              |
| 5     | 2a/2b split                                        | ✅ Approved 2026-04-16 | Option B — separate gates. 2a on Solo→Daemon tripwires; 2b on distinct UI-language tripwire. Rationale: agent-primary product (LangGraph/Temporal/Dagster/Airflow pattern); moat lives in daemon; defers UI framework decision per foundational-decisions.                                        |
| 6     | Hosted-service moat                                | ✅ Approved 2026-04-16 | Option D — primary moat = scoped token vending (extends S-1); hedge = `projectId` field in checkpoint schema; Phase 1 adds `TokenRequest` signal type + vending hook stub + `projectId`. Backups / analytics / always-on daemon remain as possible hosted offerings but are not the marquee moat. |
| 7     | Principles placement                               | ✅ Approved 2026-04-16 | Option D — link-only to `foundations.md` from header. No duplication in roadmap. Rationale: drift risk (every 6+ month project drifts on duplicated content); comparable projects (Rust RFCs, Kubernetes KEPs, Temporal, Next.js, LangChain) all link not duplicate.                              |

---

## Tier Overview

| Phase | Tier       | User                                     | Backend                     | Transport       | Primary tripwire to next phase                                                                                       |
| ----- | ---------- | ---------------------------------------- | --------------------------- | --------------- | -------------------------------------------------------------------------------------------------------------------- |
| 1     | Solo       | Individual dev                           | SQLite (per-directory)      | MCP stdio + CLI | Persistence-language issues, dependent repos, or distinct authors (Tripwires §)                                      |
| 2a    | Daemon     | Dev needing persistence / multi-terminal | SQLite (daemon-owned)       | + HTTP          | Multi-user language, revenue, or opt-in telemetry (Tripwires §)                                                      |
| 2b    | Dashboard  | Dev wanting visual inspection            | + UI layer                  | + WebSocket/SSE | **Independent tripwire** — UI-language issues, community UI PRs, hosted demand (Tripwires §). Does not gate Phase 3. |
| 3     | Team       | Small teams (2–20 devs)                  | Postgres                    | + auth/RBAC     | Enterprise inquiries, SAML/OIDC requests, or hosted team scale (Tripwires §)                                         |
| 4     | Enterprise | Compliance-bound orgs                    | Postgres + tenant isolation | + SSO           | N/A (final tier)                                                                                                     |

---

## Phase 1 — Solo (scope approved)

**Package:** `@foreman-lab/solo` • **Install:** `npm install -g @foreman-lab/solo` (primary) or `npx @foreman-lab/solo mcp` (try-it).
**Persona:** Individual developer running AI coding agents (Claude Code, Copilot CLI, Codex, Cursor, Aider) locally.
**Solo is the core product**, not an MVP replaced by Daemon. Daemon extends without supplanting.

### In scope

| Item                                                                                     | Notes                                                                                                                           |
| ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------- |
| LangGraph flow (init → plan → work → evaluate)                                           | D-16; 7 state channels                                                                                                          |
| SQLite checkpointer at `.foreman/checkpoints.db`                                         | `.foreman/` is project-root marker (like `.git/`)                                                                               |
| MCP stdio server (primary consumer)                                                      | P-4                                                                                                                             |
| CLI subset — 6 commands: `init`, `status`, `step`, `resume`, `inspect`, `validate-playbook` | Debug + CI only (not full MCP mirror)                                                                                        |
| Playbook injection via signal                                                            | D-18 — no filesystem loading                                                                                                    |
| 2 example playbooks: `tdd-feature`, `refactor-extract`                                   | In `examples/`                                                                                                                  |
| Interrupt-at-edge via LangGraph `interrupt()`                                            | Pause between nodes, not mid-node                                                                                               |
| `docs/agent-setup.md`                                                                    | Copy-paste snippets per agent                                                                                                   |
| `docs/error-codes.md`                                                                    | `ErrorCode` → cause → fix                                                                                                       |
| Per-directory `.foreman/` isolation                                                      | Multi-project works by default                                                                                                  |
| **`TokenRequest` signal type** (Gate 6)                                                  | Agent asks foreman for scoped credentials. Phase 1 Solo default: pass-through to local env. Phase 2 hosted: HSM / Vault-backed. |
| **Vending hook stub** in handlers (Gate 6)                                               | Interface-only in Phase 1. Default implementation reads local env. Hosted implementation later inserts scoped-vending logic.    |
| **`projectId: string` in checkpoint schema** (Gate 6 hedge)                              | Single field. Preserves future multi-project aggregation pivot without schema migration on every user's `.foreman/`.            |

### Out of scope

| Item                                                                          | Deferred to                                        |
| ----------------------------------------------------------------------------- | -------------------------------------------------- |
| Persistent daemon, HTTP, UI, multi-user, cross-project aggregation, telemetry | Phase 2+                                           |
| `foreman scaffold`                                                            | Replaced by `docs/agent-setup.md` (probably never) |
| JSONL journal export                                                          | Phase 2 if needed (SQLite is Phase 1 export)       |
| Composite playbook example                                                    | Phase 2                                            |
| Full CLI mirror of MCP introspection                                          | Never (zero human value)                           |

### Ship criteria

Detailed ship criteria live in [`docs/phase-1-solo/spec.md`](phase-1-solo/spec.md) (locked baseline; amended post-2026-04-18 domain-blind pivot per ADR 0007). The spec enumerates milestones M0-M6, gates G1-G10, and testing methodology.

### Tripwires to Phase 2

See Gate 4 (pending approval below).

---

## Phase 2 — Daemon (scope proposed)

### Phase 2a — HTTP daemon

**Package:** `@foreman-lab/daemon` (additive to Solo).
**Persona:** Solo user whose work crosses terminals/sessions/tools; wants persistence without per-command process spawn.

In:

- Long-running foreman process
- HTTP API (framework locked at Phase 2a start — Fastify / Hono / Express)
- Logging library (locked at Phase 2a start — pino likely)
- Multi-terminal / multi-tool access
- Solo mode stays first-class (P-8)
- Promote Phase 1 Q-5 (idle memory + FD count) from advisory to ship gate — thresholds re-locked at Phase 2a start based on daemon-mode steady-state (Phase 1 defaults: < 80 MB RSS, < 30 FDs)
- Promote Phase 1 Q-11b (first foreman domain signal latency, cold) from characterization-only to ship gate — target committed at Phase 2a start using Phase 1 production distribution (p95 + p99)
- **Journal compaction** — Phase 1 accepts unbounded journal growth (R-17); Phase 2a daemon introduces compaction (pruning old events beyond a configurable retention window, with archive export for audit). Exact compaction policy locked at Phase 2a start.

Out:

- No multi-user (localhost bind default)
- No auth beyond OS user check
- No UI (that's 2b)

### Phase 2b — Dashboard

**Package:** `@foreman-lab/dashboard` (depends on Daemon).
**Persona:** Dev wanting visual plan/journal inspection.

In:

- Thin web UI (framework locked at Phase 2b start — React / Vue / Svelte)
- Live status (WebSocket or SSE)
- Plan tree visualization
- Journal timeline with filter/search
- Graph visualization library (locked at Phase 2b start)

Out:

- No team collaboration
- No analytics beyond single-user

### Hosted-service moat (Gate 6 — APPROVED 2026-04-16)

**Primary moat: scoped token vending.** The hosted service vends time-bounded, scope-limited credentials to agents without operators running Vault / KMS / HSM infrastructure themselves. This is the marquee feature at Phase 2 ship.

**Why this beats the alternatives:**

- Directly extends S-1 (no ambient credentials) — a Phase 1 invariant
- Self-hosting at scale is genuinely painful: key management, rotation, audit trails, revocation
- Proven analog: n8n's credentials vault (strong demand, clear retention)
- Every enterprise prospect asks about credentials before dashboards
- Only candidate where the OSS user genuinely struggles to replicate

**Phase 1 design accommodation (per Gate 6):**

- `TokenRequest` signal type (agent → foreman) — Phase 1 Solo default: pass-through to local env
- Handler-layer vending hook — stub in Phase 1, hosted implementation inserts later
- `projectId` field in checkpoint schema — design hedge preserving multi-project aggregation as a cheap Phase 2+ pivot

**Dropped from marquee moat** (still available as secondary hosted features if demand surfaces):

- Always-on daemon (commoditizable via systemd)
- Automatic `.foreman/` backups (low switching cost — file copy)
- Opt-in usage analytics (trivially replicated)
- Multi-project aggregation (retained as design hedge only; not primary)

**If Phase 1 adoption data reveals different demand** — primary moat may be changed via ADR (`docs/adr/NNNN-moat-revision.md`). The `projectId` hedge means swapping to multi-project aggregation is a no-migration change.

---

## Phase 3 — Team (scope proposed)

**Package:** `@foreman-lab/team` (builds on Daemon).
**Persona:** Small teams (2–20 devs) sharing foreman state.

In:

- Postgres checkpointer (`PostgresSaver` — one constructor swap per D-16)
- Postgres client (locked at Phase 3 start — pg / Drizzle / Prisma)
- Auth library (locked at Phase 3 start — Lucia / Passport / @fastify/jwt)
- RBAC: admin / dev / observer
- Multi-project team dashboard
- Team audit log (separate from journal)
- Shared playbook library within team

Out:

- No SSO (Phase 4)
- No compliance certifications
- No VPC isolation

---

## Phase 4 — Enterprise (scope proposed)

**Package:** `@foreman-lab/enterprise` (builds on Team).
**Persona:** Enterprises with compliance, multi-tenancy, SSO requirements.

In:

- SSO (SAML / OIDC — library locked at Phase 4 start)
- Multi-tenant cryptographic isolation
- Audit log export (SIEM-compatible JSONL, retention)
- Private registry mirror (playbooks/prompts, air-gapped)
- SLA tiers + support contracts
- Compliance attestations pursued separately (SOC 2 Type II baseline)

Final tier. Post-ship work is incremental within tiers, not new phases.

---

## Tripwire Thresholds (Gate 4 — APPROVED 2026-04-16)

**Mechanism:** A phase promotes when any ONE tripwire signal fires AND sustains at the threshold for 2 consecutive weeks (except Team→Enterprise, which promotes on first fire).

### Solo → Daemon (Phase 1 → Phase 2)

| Signal                 | Threshold                                                                                                                                                                   |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Language-tagged issues | 15+ issues with persistence/state-loss language ("lost", "state", "restart", "another terminal", "tmux", "systemd", "docker", "--serve", "--watch") from 10+ distinct users |
| Dependent repos        | 15+ GitHub repos listing `@foreman-lab/solo` in `package.json`                                                                                                              |
| Distinct issue authors | 30+ distinct users filing issues in rolling 30 days                                                                                                                         |

### Daemon → Team (Phase 2 → Phase 3)

| Signal              | Threshold                                                                                           |
| ------------------- | --------------------------------------------------------------------------------------------------- |
| Multi-user language | 8+ requests mentioning "multi-user" / "shared" / "my team" / "RBAC" / "permissions" / "who changed" |
| Revenue             | 3+ hosted-service customers written-in about team tier                                              |
| Opt-in telemetry    | 30+ distinct orgs in opt-in telemetry (P-7 preserved: opt-in only)                                  |

### Team → Enterprise (Phase 3 → Phase 4)

No 2-week cooldown (enterprise procurement is already slow).

| Signal               | Threshold                                                                               |
| -------------------- | --------------------------------------------------------------------------------------- |
| Enterprise inquiries | 3+ with "SSO" / "SCIM" / "SAML" / "OIDC" / "audit" / "compliance" / "SOC 2" / "on-prem" |
| Revenue              | 1+ paid team-tier customer explicitly requesting SAML or OIDC                           |
| Hosted team scale    | 5+ orgs with 3+ active users on hosted team tier                                        |

### 2a → 2b (Daemon → Dashboard)

**Distinct tripwire per Gate 5 (Option B).** 2b ships only when UI demand signal fires.

| Signal             | Threshold                                                                                                                                                |
| ------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------- |
| UI-language issues | 10+ issues with inspectability language ("dashboard", "visualize", "graph view", "see the plan", "timeline", "web UI", "browser") from 5+ distinct users |
| Community UI PRs   | 2+ unsolicited PRs or RFC issues proposing dashboard implementation                                                                                      |
| Hosted demand      | 3+ hosted-service customers requesting visual inspection in writing                                                                                      |

Any ONE fires + sustains 2 weeks → 2b activates. Same mechanism as other tripwires. Separate ADR from 2a (`docs/adr/NNNN-phase-2b-activation.md`).

**Acceptable failure mode:** 2b tripwire may never fire. That is an honest "users didn't want UI" signal, not a project failure. 2a stands on its own.

### Tripwire review process (once Phase 1 ships)

Quarterly:

1. Run keyword scan against issues/discussions (scripted — `gh issue list --json title,body,author | grep -iE '<keywords>'`).
2. Count unique issue authors (rolling 30 days).
3. Count dependent repos via `https://github.com/<owner>/<repo>/network/dependents`.
4. Track hosted-service written inquiries (private log once Phase 2 ships).
5. Update `docs/tripwires.md` with current readings.

Signal firing opens `docs/adr/NNNN-phase-N-activation.md` with evidence + scope lock. Phase activation is explicit.

---

## Cross-phase invariants (never change)

| Invariant                                    | Mechanism                                  |
| -------------------------------------------- | ------------------------------------------ |
| Signal protocol (agent-facing API) is stable | Versioned schema; breaking = major bump    |
| P-1..P-10 enforced                           | Test suite; violations block release       |
| S-1..S-5 enforced                            | Integration tests per phase                |
| Apache 2.0 all tiers                         | LICENSE file, all packages                 |
| Agent never sees storage/transport           | Type boundary + review                     |
| No auto data migration across phases         | P-8; each phase has explicit export/import |

## Permanently out of scope (never added in any phase)

- Foreman calling LLMs (P-1)
- Foreman interpreting domain content (P-1)
- Default-on telemetry (P-7)
- Open-core feature gating (P-10)
- Proprietary persistence formats (P-6)
- "Skip operator approval" flag (S-2)

---

## Document status

- **Locked (6/7 gates):** tier structure, Phase 1 Solo scope, tripwire thresholds, 2a/2b split (separate gates), hosted-service moat (scoped token vending primary), principles placement (link-only)
- **Retracted (1/7):** ship criteria — moved to future `docs/phase-1-solo/spec.md`
- **Revision process:** locked items require approval + ADR (`docs/adr/NNNN-<topic>.md`) + doc update. Tier reordering, phase collapse, or moat pivot all require ADR.
- **Next document:** Phase 1 Solo spec (`docs/phase-1-solo/spec.md`) — will cover ship criteria, MCP tool surface, CLI command schemas, playbook contract Zod, signal types including `TokenRequest`, checkpoint schema including `projectId`, scenario tests, agent transcript format.
