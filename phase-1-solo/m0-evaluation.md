---
status: M0 Complete — G1 gate PASS (local)
milestone: M0 — Repo Foundation
gate: G1
date: 2026-04-17
methodology: Foreman PGE — Evaluate state
---

# M0 Evaluation — Repo Foundation

## Result: G1 GATE PASS (local verify)

CI verification pending first push; local verify on Node 24.14.0 + pnpm 9.15.0 green across all 7 gates.

## Gate-by-Gate

| #   | Gate                             | Result  | Notes                                           |
| --- | -------------------------------- | ------- | ----------------------------------------------- |
| 1   | `pnpm install --frozen-lockfile` | ✅ PASS | 249 packages, lockfile stable, 357ms            |
| 2   | `pnpm typecheck`                 | ✅ PASS | `tsc --noEmit` via turbo, 953ms                 |
| 3   | `pnpm lint`                      | ✅ PASS | typescript-eslint v8 flat config, 0 issues      |
| 4   | `pnpm test`                      | ✅ PASS | vitest 2.1.8, `passWithNoTests`, 683ms          |
| 5   | `pnpm format:check`              | ✅ PASS | prettier 3.4.2 with `.prettierignore`           |
| 6   | `pnpm boundary-check`            | ✅ PASS | dep-cruiser 17.3.10 (upgraded, see Adjustments) |
| 7   | `pnpm licenses:check`            | ✅ PASS | vacuous — 0 prod deps at M0                     |

## Artifacts Shipped (25 files)

**Root scaffolding (M0-A, 5):** `.gitignore`, `pnpm-workspace.yaml`, `turbo.json`, `package.json`, `tsconfig.json`

**Solo package (M0-B, 3):** `packages/solo/package.json`, `packages/solo/tsconfig.json`, `packages/solo/src/index.ts`

**Lockfile (M0-C, 1):** `pnpm-lock.yaml` (2600 lines, 249 pkgs)

**Tooling (M0-D, 4):** `eslint.config.js`, `.prettierrc.json` (v1 retained), `.dependency-cruiser.cjs`, `vitest.config.ts`

**Policy (M0-E, 4):** `.npmrc`, `.pnpmauditrc.json`, `scripts/audit-with-policy.mjs`, `scripts/check-licenses.mjs`

**CI (M0-F, 1):** `.github/workflows/ci.yml`

**Docs (M0-G, 5):** `LICENSE` (Apache-2.0), `README.md`, `CONTRIBUTING.md`, `.nvmrc`, `.editorconfig`

**Polish (M0-H, 2):** `.prettierignore`, `docs/phase-1-solo/m0-evaluation.md` (this file)

## Adjustments from Proposal

### Adj-1: dep-cruiser 16.7.0 → 17.3.10

**Trigger:** Node 24.14.0 (local dev) rejects `import { R_OK } from 'node:fs'` used internally by dep-cruiser 16.7.0. Upstream bug, patched in 17.x.

**Decision:** Upgrade at M0 rather than defer. Rationale: blocks local verify for anyone on Node 22+; CI still uses Node 20 but local dev experience is a hard dogfood requirement.

**Impact:** devDep bump only. No semantic change — rules file schema identical between 16.x and 17.x.

### Adj-2: `.prettierignore` added (not in original proposal)

**Trigger:** First `pnpm format:check` caught 21 files outside project scope (`.claude/`, `copilot-plugin/`, `pnpm-lock.yaml`).

**Decision:** Add `.prettierignore` excluding vendor/tooling directories. Zero policy change — only defines format scope.

### Adj-3: `.dependency-cruiser.cjs` orphan-rule refinement

**Trigger:** Entry points (`packages/*/src/index.ts`, future `cli.ts`) are legitimately orphan modules — that's their job.

**Decision:** Extend `no-orphans.from.pathNot` to exclude `packages/*/src/(index|cli).ts`. Empirical refinement; rule intent unchanged.

## Decisions Validated

All 9 decisions (D-M0-1..9) survived WORK. Key validations:

- **D-M0-1 (Turborepo day 1):** Correctly functions as no-op passthrough with 1 package. `turbo run typecheck` adds ~200ms overhead — acceptable.
- **D-M0-2 (v8 provider):** `@vitest/coverage-v8` installed cleanly; no c8 regression.
- **D-M0-6 (no stub-skips):** CI workflow `verify` job runs 7 real verification steps, no placeholders. (Split-into-separate-jobs deferred to post-M0; single-job acceptable for M0 scope.)
- **D-M0-8 (packageManager field):** `corepack` auto-activated pnpm 9.15.0 on first `pnpm` invocation. Confirmed.

## Open Risks Carried Into M1

1. **CI verification pending.** Local green ≠ CI green. First push will tell. If ubuntu-latest Node 20 behaves differently, fix-forward.
2. **dep-cruiser warning suppression for entry points** may mask real orphans at M1 when files land. Revisit the pathNot regex at M1 close.
3. **Node version spread** (local 24, CI 20, engines >=20). No issue today, but semver-minor Node bugs could diverge. Consider locking CI + dev to 20.x LTS.

## Transition to M1

**Next milestone:** M1 — Contracts (per spec §3.2: Zod schemas + error taxonomy, ~1 week).

**Preconditions met:**

- ✅ Empty commit passes all gates (once committed).
- ✅ Monorepo structure ready for `packages/solo/src/types/`, `packages/solo/src/errors/`.
- ✅ Test runner configured (vitest 2.1.8 + v8 coverage).
- ✅ dep-cruiser ready to enforce handlers→graph boundary when those dirs land (M2+).

**Correction:** initial M0 evaluation draft said "M1 = Ledger + effective-state primitives." That was wrong — ledger lands in M2+ with the graph nodes. Spec §3.2 M1 = Contracts only. See `docs/phase-1-solo/m1-development-proposal.md`.

**Recommendation:** Commit M0, push, confirm CI green, then open M1 planning (proposal → GAN review → WORK → evaluate).

## GAN Round 4 — Post-WORK Adversarial Review

Conducted 2026-04-17 after WORK completion, before commit.

**Copilot verdict:** CLEAN (1 nit only — LICENSE copyright year).

**Codex verdict:** REPLAN — 6 must-fix items.

### Fixes applied before commit

1. **`.npmrc`:** flipped `ignore-scripts=false` → `ignore-scripts=true`; removed redundant `enable-pre-post-scripts=true`. Codex flagged as supply-chain regression. (Security-correct per proposal.)
2. **`.dependency-cruiser.cjs`:** `no-orphans` severity `warn` → `error`. Codex flagged G1 vacuously green at `warn`. Entry-point exemption in `pathNot` protects legitimate `index.ts` / `cli.ts`.
3. **`scripts/audit-with-policy.mjs`:** audit-tool-failure path added. Previous version silently returned zero advisories on parse error; now exits code 2 with diagnostic if JSON malformed.
4. **`docs/phase-1-solo/m0-evaluation.md`:** corrected "7 real jobs" → "7 real verification steps" (ci.yml is 1 job with 7 steps).
5. **`packages/solo/package.json`:** `bin.foreman` corrected from `dist/cli.js` to `dist/index.js` to match proposal §3 task 6.
6. **`LICENSE`:** copyright line updated from `2026` to `2025-2026 foreman-lab contributors` (Copilot nit; preserves original creation year).

### Rejected

- **Codex MUST-3** (CI trigger only on push to main): misread. `.github/workflows/ci.yml:6-7` already declares `pull_request.branches: [main]`. No change needed.

### Three-reviewer convergence

After round 4 fixes, Claude + Codex + Copilot all agree G1 is genuinely (not vacuously) satisfied.
