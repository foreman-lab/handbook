# Foundations

The values and lock-ins that define what `foreman` is. Principles guide judgment. Decisions are specific commitments that follow from them. Both are expensive to change.

## Principles

Foreman has two kinds of principles: how we build the project (`B*`), and what the product itself is (`P*`). A few cut across both halves — security, reproducibility, and openness apply to how we work and to what foreman does for its users.

### Build principles

**B1. Step-by-step, vertical slices.** One milestone in flight. Ship walking skeletons first, thicken through slices. No milestone over two weeks.

**B2. Open from day one.** We build in public: every commit, ADR, and design decision is visible from the first push. Foreman ships under Apache-2.0 with `DCO`-signed commits; we monetize via hosted service, never by crippling the OSS build.

**B3. Demand-driven, not schedule-driven.** Tripwires move us between tiers. Calendar dates do not.

### Product principles

**P1. Harness, not agent.** Foreman orchestrates AI coding agents; it never writes code or interprets domain content itself.

**P2. Domain-blind core.** The engine is a pure state machine. Agents render prompts from playbook data externally. Foreman never reads a playbook to decide what to do.

**P3. Agents are first-class.** Agents are the primary interface, not commodity tool-callers. They own prompt rendering and shape user outcomes; foreman serves them.

**P4. Three actors, clean roles.** Harness owns state and gates irreversible actions. Agents produce content. Operators (humans) judge and unblock. Boundaries are types, not conventions.

**P5. Security by design.** Security is a first-class concern from the first line of code, not a hardening pass later. Foreman applies the same stance to the workflows it orchestrates: irreversible actions are gated, credentials are explicit, denials are auditable.

**P6. Reproducibility.** Builds, tests, and CI are deterministic; nondeterminism is a bug, not a quirk. Foreman runs the same way: given the same inputs and state, the same action sequence — no wall-clock decisions, no hidden randomness.

**P7. Tiers speak wire protocol.** Higher tiers (solo, team, cloud) orchestrate the layer below via `MCP` and IPC, not by importing it. Think `docker` / `compose` / `kubernetes`, not library reuse.

**P8. One binary, multiple modes.** The `foreman` CLI runs the engine in-process or as a thin client to a local daemon. Solo and daemon are modes of one binary, not separate tiers.

**P9. Portable by default.** Every persistent artifact has a documented export format the day it ships. No dark data.

**P10. No telemetry by default.** Opt-in only. Non-negotiable for a tool that sees private code and prompts.

## Decisions

**D1. TypeScript on Node.js 20+.** The ecosystem AI coding agents live in. `MCP` SDKs target it. Python, Go, Deno, and Bun are rejected for this reason.

**D2. Async/await only.** No callbacks, no event emitters for control flow. Events are for observability, never orchestration.

**D3. Zod is the source of truth.** Schemas first; TypeScript types come from `z.infer`. Never hand-write a type that parallels a schema.

**D4. Throw typed errors internally; wire format at the edge.** Internal code throws a small error taxonomy (`BaseError`, `ValidationError`, `SecurityError`, `StorageError`). One catch at the protocol boundary converts to a wire response. No `Result` types in internal APIs.

**D5. `MCP` is the contract.** The external surface is two tools: `foreman__status` (read-only) and `foreman__step` (polymorphic write). The internal signal vocabulary is richer; the tool count is a transport grouping, not a semantic lid.

**D6. `MCP` SDK behind one adapter.** Every call to `@modelcontextprotocol/sdk` lives in a single directory. The SDK is pinned to an exact version. When it breaks, one file changes.

**D7. Playbooks ride inside briefs.** The harness never reads playbook files at runtime. Agents embed the full playbook contract in the initialization signal. No `playbooks/` directory in the engine.

**D8. Transport, handler, flow are three layers.** Transport validates and forwards. Handlers translate signals to flow commands. Flow runs the state machine. Cross-layer imports fail the build.

**D9. No engine side-channels.** No prebuilt agent primitives, no node-level retry or cache policies, no engine-managed streaming as default. All semantic transitions go through validated signals, state channels, and journaled events.

**D10. Solo tier is one binary with two modes.** The `foreman` CLI runs the engine in-process (single project, foreground) or as a thin client to a local daemon (many projects, same user, background). Both modes are the solo tier. Daemon is not a tier of its own. Multi-user and networked tiers (team, cloud) live in separate repos when built.

**D11. Two repos now.** `handbook` holds docs. `foreman` holds code. Future tier repos (team, cloud) are created only when that tier is scoped and started. No premature splitting; core is not extracted from `foreman` as a library.

**D12. Milestones are semver tags.** Each milestone ships as `0.1.0`, `0.2.0`, and so on. Walking skeleton first; vertical slices thicken it. One milestone in flight, none over two weeks.

**D13. Apache-2.0, `DCO`-signed.** Public OSS from the first commit. All tiers share the license. No open-core split.

**D14. Config precedence, fixed.** CLI flags beat env vars beat project config beat built-in defaults. Documented once, enforced everywhere.

**D15. `JSON` for machines, `YAML` for humans.** Journal, checkpoints, wire messages are `JSON`. Human-edited config may be `YAML`, parsed via `yaml@2.x` with Zod validation.

**D16. Naming: product name on external surfaces only.** `foreman` appears in the CLI binary, `MCP` tool names, env var prefix, and config file names. Internal types, classes, and functions use neutral names. The import path is the namespace.

**D17. Domain vocabulary is locked.** `node`, `brief`, `playbook`, `plan`, `work`, `evaluation`, `outcome`, `block`, `journal`, `checkpoint`, `operator`, `agent`, `harness`. Synonyms drift; we reject them at review.

**D18. No premature ports.** We add an outbound port only when real variation exists across tiers or tests, or when an alternate implementation is landing next. Ceremony interfaces with one implementation are rejected.

**D19. Vitest, ESLint, Prettier, GitHub Actions.** Standard TypeScript toolchain. CI gates: typecheck, test, lint, format.

**D20. Tier orchestration follows the Docker model.** Each tier wraps the tier below by speaking its wire protocol, not by linking to its code. Solo is `foreman` itself (foreground or daemon mode). Team is a LAN host that routes `MCP`/IPC to per-user `foreman` instances. Cloud is a multi-tenant supervisor that spawns tenant daemons. Any tier can be rewritten in another language without touching the layers below, the same way `docker compose` does not share a codebase with `dockerd`, and `kubernetes` does not share one with either. Concretely: `foreman` stays self-contained, core is never extracted, the wire protocol is the only contract.

## How this changes

These foundations shift through ADRs, not chat. To amend: open an `ADR` under `foreman-lab/handbook/adr/`, propose the change with migration notes, take it through PR review, and update this file in the same PR. Principles carry a higher bar than decisions: amending one is re-chartering, and the PR should say so in plain words.
