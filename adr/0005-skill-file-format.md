---
status: Accepted (locked 2026-04-18 by operator sign-off simultaneously with M4 proposal vN per R-M4-10)
date: 2026-04-18
deciders: Operator (xuzhijie), after M4 proposal GAN rounds 1-3 (Codex + Copilot) locked Q-1a and Q-1b directions
supersedes: none
superseded_by: none
amends: none
related:
  - docs/adr/0001-architectural-style-declaration.md (D-18 skill injection; D-20 narrow port discipline — this ADR locks the operational contract those decisions imply)
  - docs/phase-1-solo/m4-development-proposal.md §3.2 "Runtime/authoring split", §11 Q-1a/Q-1b (superseded in part post-2026-04-18 domain-blind pivot)
  - docs/adr/0007-domain-blind-core.md (post-2026-04-18 pivot; retires PromptAssemblerPort and supersedes the never-authored ADR 0006 slot)
---

# ADR 0005: Skill file format, distribution, and `validate-skill` CLI contract

## Post-2026-04-18 domain-blind pivot amendment

This ADR remains **Accepted** as-is for the skill file format + distribution boundary + `validate-skill` CLI contract. The 2026-04-18 domain-blind pivot (ADR 0007) changed what happens *after* a valid `Skill` object lands in `brief.skill`: foreman no longer renders prompts. External agents read `brief.skill.methodology[phase]` verbatim from the `brief` channel via `foreman__status` and render prompts locally. The format + distribution + validation locks in this ADR are unaffected.

**ADR 0006 slot (`prompt-assembler-port`):** never authored. The `PromptAssemblerPort` design that ADR 0006 would have codified was reverted mid-M4 as part of the domain-blind pivot. ADR 0007 supersedes that planned work. Future ADRs pick up at 0008.

**`validate-skill` CLI positioning:** still transport-layer dev tool, no runtime coupling to any port — unchanged by the pivot since it was already port-free. Deferred from M4 to **M5** because the surrounding example-skill + agent-setup deliverables slipped.

## Status and scope

Accepted (operator sign-off 2026-04-18 simultaneously with M4 proposal vN; amended with pivot note 2026-04-19). Authored before M4 proposal locked to vN final; Codex Round 3 final GAN flagged the missing ADR as the single BLOCKER to lifecycle promotion. This ADR codifies the skill distribution boundary that the M5 `validate-skill` CLI + example-skill work builds against.

Scope: **authoring and distribution format** of skills (files operators or wrapper authors ship) + **`foreman validate-skill` CLI contract** (dev-time tool) + **runtime skill boundary** (what foreman actually accepts from wrappers at `foreman__step type=initialize` time).

Out of scope: prompt assembly semantics (now external to foreman per ADR 0007 — agents own rendering); tier-specific skill behavior (Phase 2a).

## Context

### D-18 boundary

Foundations D-18 ("skill injection") locks: foreman **never reads skill files at runtime**. Wrappers or operators are responsible for parsing skill distribution artifacts (YAML/TS) into the inline `Skill` object shape (`packages/solo/src/types/skill.ts:20-26`) and embedding that object into the brief as `brief.skill`. This is a hard D-18 boundary — runtime adapters accept only already-parsed `Skill` values. Any runtime file-read would violate D-18.

### The M4 tension (pre-pivot context — retained as history)

At the time this ADR was authored (2026-04-18), M4 planned to ship:

- The first formal port (`PromptAssemblerPort`) that consumes `brief.skill.methodology.<phase>` **[retired per ADR 0007 — the same-day domain-blind pivot relocated rendering to the agent layer; port + default adapter + supporting types were reverted before any agent consumed them]**
- A CLI `validate-skill` subcommand for operators authoring skills **[deferred from M4 to M5 because the surrounding example-skill work slipped; CLI itself is port-free and unaffected by the pivot]**
- An example skill at `examples/tdd-feature/` (first shipped skill) **[deferred from M4 to M5]**

These three shipped together in the original plan, so M4 had to lock: (a) what format skill authors use, (b) what `validate-skill` reads/validates/emits, and (c) where the authoring-vs-runtime boundary sits. Proposal GAN rounds surfaced (Q-1a + Q-1b) and converged on directions; this ADR ratifies the format + CLI + boundary decisions. **The pivot does not invalidate those three locks** — only the rendering-side consumer (PromptAssemblerPort) was retired; the skill format, `validate-skill` CLI, and D-18 boundary all stand as written below.

### Alternatives considered

- **A. JSON canonical.** Rejected: JSON is hostile for multi-line prose (methodology strings are often 30+ lines of Markdown-ish prose). Authors would escape newlines and lose editor support.
- **B. TS canonical.** Rejected: requires a build step or `ts-node`/`tsx` at operator-author time; breaks "one file on disk is the skill" simplicity; makes skill registries harder to publish as plain assets.
- **C. YAML canonical + optional TS helper.** Adopted. YAML handles prose cleanly; TS remains available for authors who want programmatic helpers (e.g., computed methodology composition) but produces a parsed `Skill` value at distribution time.
- **D. Runtime file lookup (`brief.skill.ref` or filesystem resolve).** Rejected: violates D-18 outright.

## Decision

### 1. Canonical distribution format: **YAML**

Skill authors publish YAML files conforming to the YAML projection of `SkillSchema` (`types/skill.ts`). Canonical example:

```yaml
# examples/tdd-feature/skill.yaml
name: tdd-feature
methodology:
  plan: |
    You are planning a test-driven feature implementation for: {{goal}}
    Scope (read-only: {{scope_readonly}}): {{scope_paths}}
    ...
  work: |
    Execute the plan: {{plan}}
    ...
  evaluate: |
    Given the plan {{plan}} and the work report {{work}},
    determine whether the work satisfies the plan.
    ...
scope:
  readOnly: false
  paths:
    - packages/solo/
```

Top-level keys match `SkillSchema` exactly (`name`, `methodology`, `scope?`). Methodology strings may use `{{var}}`, `${var}`, or any placeholder convention the agent layer recognizes. Post-2026-04-18 domain-blind pivot (ADR 0007): **the external agent interpolates placeholders against its own state context** — foreman no longer renders prompts, and no `PromptAssemblerPort` or `view.vars` bag exists at runtime. The `{{var}}` convention is recommended (documented in `docs/agent-setup.md` at M5) because the M5 `validate-skill` CLI does a shallow static check on it.

### 2. TS is an optional authoring helper, not a parallel runtime surface

Authors who want programmatic skill construction may ship a TS module:

```ts
// advanced-skill/index.ts
import type { Skill } from '@foreman-lab/solo/types';
export const skill: Skill = { name: ..., methodology: {...}, scope: {...} };
```

The TS file is resolved at **authoring time** (via wrapper tooling or `tsx`) into a plain `Skill` object. Foreman runtime never imports or executes TS skill modules.

### 3. `foreman validate-skill <path>` CLI contract

`validate-skill` is a **transport-layer dev tool** (lives under `cli/validate-skill.ts`); it does NOT depend on `PromptAssemblerPort`. Contract:

- **Input:** `<path>` argument (relative to `cwd`), pointing to a YAML file (default) or TS module (`.ts` extension)
- **Does NOT declare `--foreman-dir`**: validation is filesystem-local, not checkpoint-related (Round 2 Copilot H1: "accept but ignore" is antipattern)
- **Success output:** `WireResponse<Skill>` JSON on stdout (`{ ok: true, data: <parsed Skill> }`); exit 0. Operator pipes via `jq '.data'` to embed in a brief.
- **Failure output:** `WireResponse<never>` with `error.code = SKILL_VALIDATION_FAILED`; exit 1. `error.details.zodIssues` carries Zod parse output.
- **Shallow template-var extraction (Q-1b):** after Zod parse succeeds, extract `{{var}}` tokens from each `methodology.<phase>` string via `/\{\{(\w+)\}\}/g`. Post-2026-04-18 domain-blind pivot: the phase-var allowlist lives with the **M5 example-skill documentation + agent-setup** (not the retired `graph/view-mapping.ts`). Undeclared vars emit warnings to **stderr**; stdout remains valid `WireResponse<Skill>` and exit stays 0 (warnings do not fail the command, because skill authors may reference future-sprint vars or use agent-side extensions).

### 4. Runtime boundary (D-18 reaffirmation)

The foreman runtime (graph nodes, handlers) accepts **only** the inline `Skill` object shipped in `brief.skill`. It never:

- Reads files from disk
- Resolves references (e.g., `brief.skill.ref: "tdd-feature"`)
- Accepts raw YAML or TS module strings
- Performs validation on YAML syntax (that's validate-skill's job, not the runtime's)
- **Renders prompts from methodology text** (post-2026-04-18 pivot; ADR 0007 — agents own rendering)

`foreman__step type=initialize` Zod-parses `brief.skill` through `SkillSchema`; parse failure → `SCHEMA_VALIDATION_FAILED` (existing M3 code). No runtime error code for skill lookup (no `SKILL_NOT_FOUND` — the condition cannot exist per D-18). No runtime error code for prompt rendering (rendering is agent-side — errors surface inside the agent).

### 5. What this ADR does **not** decide

- How agents render prompts from `brief.skill.methodology[phase]` (agent-side per ADR 0007; M5 `docs/agent-setup.md` documents canonical conventions)
- Phase 2a tier-specific skill extensions (deferred; Phase 2a ADR)
- Skill hot-reload semantics (deferred; non-goal in the historical M4 proposal §10)

## Consequences

### Positive

- **One distribution format to learn** (YAML); TS available as escape hatch
- **Authoring tooling is trivially portable** — skill registries ship YAML files; wrappers load once and hand off inline
- **Runtime never sees an unparsed skill** — D-18 holds by construction
- **`validate-skill` emits a pipeable envelope** — operators author briefs via `foreman validate-skill … | jq '.data'`, matching §4 dogfooding flow in M4 proposal
- **Template-var warnings at authoring time** — authors learn about `{{var}}` mismatches before the agent hits them at render time (post-2026-04-18 pivot: runtime `PROMPT_ASSEMBLY_FAILED` error code is retired; rendering errors surface in the agent layer)

### Negative

- **Two format paths** (YAML + TS) double the authoring documentation surface slightly; TS remains a minority path (expected <10% of skills)
- **Warnings are soft** — validate-skill does not fail on undeclared template vars; operators who ignore warnings may ship skills whose agents raise rendering errors at runtime. Post-2026-04-18 pivot: mitigation shifts from runtime `PROMPT_ASSEMBLY_FAILED` (retired) to clear authoring-time warnings + agent-side error surfaces documented in M5 `docs/agent-setup.md`.
- **No lazy-loading story at runtime** — if operators want a skill registry with N skills, the wrapper must parse all of them upfront or lazily at brief-construction time. Acceptable for Phase 1 (single-tenant operator tools); Phase 2a daemon may revisit

### Neutral

- The warning-to-stderr decision means CI gates on `validate-skill` exit-code-only: exit 0 = parseable YAML + valid Zod shape, not necessarily "all vars will resolve at runtime". This matches Phase 1's optimistic stance; Phase 2a stricter mode is a future ADR

## Batch mapping (M4 proposal)

- **Batch F** (`cli/validate-skill.ts`) implements §3 above + emits stdout/stderr contracts
- **Batch G** (`examples/tdd-feature/skill.yaml`) is a canonical YAML skill matching §1; must pass validate-skill exit 0 (Batch G acceptance)
- **Batch H** (docs) folds this ADR's decisions into `docs/phase-1-solo/spec.md` §10 and `docs/dogfooding-protocol.md` skill-authoring template

## Changelog

- **v1 (2026-04-18)** — Draft authored as M4 v0.4 proposal prerequisite; Q-1a YAML-canonical lock + Q-1b shallow-extraction lock ratified. Pending operator sign-off.
- **v1 Accepted (2026-04-18)** — Operator sign-off completed simultaneously with M4 proposal lock (vN). Status flipped Draft → Accepted per R-M4-10 (proposal §5 risk register). No content changes since v1 draft.
