---
status: HISTORICAL — M0 complete (see m0-evaluation.md). Banner added 2026-04-19 as part of M5 docs refactor.
milestone: M0 — Repo Foundation
gate: G1 — CI green on empty commit; dependency-cruiser green
date: 2026-04-17
methodology: Foreman PGE (Plan → Work → Evaluate) + GAN review (Codex + Copilot)
---

> **⚠ HISTORICAL — milestone complete.**
>
> This proposal is the locked plan for milestone M0 (Repo Foundation), retained as history for git-blame traceability. Evaluation outcome: [`m0-evaluation.md`](m0-evaluation.md). Current milestone: [`m5-development-proposal.md`](m5-development-proposal.md).

# M0 Development Proposal — Repo Foundation

## 1. Goal

Establish the monorepo skeleton so every subsequent milestone inherits a green CI, enforced boundaries, and reproducible installs. No runtime code; no product logic. The artifact of M0 is an empty commit that passes every quality gate.

## 2. G1 Gate Criteria

- `pnpm install --frozen-lockfile` succeeds on Node 20.
- `pnpm typecheck` green.
- `pnpm lint` green.
- `pnpm test` green (empty suite acceptable).
- `pnpm format:check` green.
- `pnpm boundary-check` (dependency-cruiser) green.
- `pnpm licenses:check` green (vacuous at M0 — no prod deps).
- CI workflow passes on `ubuntu-latest` 2-core.

## 3. Task List (22 ordered items)

1. `.gitignore` (Node + macOS + editors)
2. `pnpm-workspace.yaml` (`packages/*`)
3. `turbo.json` (D-9 locked: Turborepo from day 1)
4. Root `package.json` — `"type": "module"`, `"packageManager": "pnpm@9.x.x"`, explicit devDeps (`typescript`, `vitest`, `@vitest/coverage-v8`, `eslint`, `typescript-eslint`, `prettier`, `dependency-cruiser`, `turbo`), scripts (`typecheck`, `lint`, `test`, `format:check`, `boundary-check`, `licenses:check`, `audit`)
5. Root `tsconfig.json` (strict, ES2022, NodeNext)
6. `packages/solo/package.json` — `"name": "@foreman-lab/solo"`, `"type": "module"`, `engines.node >=20`, `bin: dist/index.js` (non-blocking at M0)
7. `packages/solo/tsconfig.json` (extends root)
8. **CHECKPOINT:** `pnpm install` → generates `pnpm-lock.yaml`
9. `packages/solo/src/index.ts` — `export {};`
10. `eslint.config.js` (flat config, typescript-eslint v8 via `tseslint.config()`)
11. `.prettierrc.json`
12. `.dependency-cruiser.cjs` — rules: no-circular, no-orphans, layer stubs (`handlers/ → graph/`)
13. `vitest.config.ts` — `provider: 'v8'` (NOT c8)
14. `.npmrc` — scoped `ignore-scripts` policy
15. `scripts/audit-with-policy.mjs` — wrapper around `pnpm audit` applying allowlist from `.pnpmauditrc.json` (stub allowlist for M0)
16. `.pnpmauditrc.json` — allowlist file consumed by wrapper
17. `scripts/check-licenses.mjs` — reads `pnpm list --prod --json`, fails on non-allowlisted (vacuous at M0)
18. `.github/workflows/ci.yml` — jobs: install, typecheck, lint, test, format:check, boundary-check, licenses:check. **No** stub-skip for future jobs (audit/gitleaks deferred to M1 per spec §2.4).
19. `LICENSE` (Apache-2.0)
20. `README.md` — reading order (foundations → roadmap → architecture → phase-1-solo/\*)
21. `CONTRIBUTING.md` (stub — ADR process, PR checklist)
22. Local verify: `pnpm install && pnpm typecheck && pnpm lint && pnpm test && pnpm format:check && pnpm boundary-check && pnpm licenses:check`

**Polish (optional, zero-cost):**

- `.nvmrc` (`20`)
- `.editorconfig`

## 4. Decisions (D-M0-1 .. D-M0-9)

| ID     | Decision                                                        | Rationale                                                                                  |
| ------ | --------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| D-M0-1 | pnpm + Turborepo from day 1                                     | D-9 locked; Turborepo is no-op passthrough with 1 package                                  |
| D-M0-2 | Vitest `provider: 'v8'`                                         | `@vitest/coverage-v8` is current; `c8` is obsolete for vitest                              |
| D-M0-3 | dep-cruiser `.cjs` config                                       | Node loads `.cjs` as CJS regardless of `"type": "module"`; dep-cruiser expects CJS         |
| D-M0-4 | `bin: dist/index.js` declared at M0                             | Non-blocking; becomes real at M1/M2 when `tsc` build lands                                 |
| D-M0-5 | Empty `export {};` stub                                         | dep-cruiser passes vacuously on empty graph; rules become falsifiable at M1                |
| D-M0-6 | **Omit** future CI jobs (no stub-skips)                         | Stub-skips create green jobs that lie about coverage; add real jobs when test files land   |
| D-M0-7 | Audit via `scripts/audit-with-policy.mjs` + `.pnpmauditrc.json` | `.pnpmauditrc.json` is not native pnpm config; wrapper script implements allowlist         |
| D-M0-8 | `"packageManager": "pnpm@9.x.x"` in root                        | Enforces pnpm version via corepack; required by R-15 (`--frozen-lockfile`)                 |
| D-M0-9 | Gitleaks + `pnpm audit` gate deferred to M1                     | Spec §2.4 explicitly defers; M0 keeps install/typecheck/lint/test/format/boundary/licenses |

## 5. GAN Review Log

### Round 1 — Original plan (20 tasks, 7 decisions)

- **Codex verdict:** REPLAN — 3 defects: c8 provider obsolete, missing reproducibility artifacts, stub-skip CI drift
- **Copilot verdict:** PROCEED after 4 MUST-CHANGEs

### Round 2 — Claude synthesis

Merged 6 MUST-CHANGEs: c8→v8, `packageManager` field, `type: module`, explicit devDeps, omit stub-skip CI, audit wrapper script. Revised to 22 tasks, 9 decisions.

### Round 3 — Copilot re-review of revised plan

- **Verdict:** PROCEED to WORK
- All 4 Copilot MUST-CHANGEs covered by revised plan
- Confirmed KEEPs: Turborepo (D-9 locked), `.cjs` extension, `bin` declaration, gitleaks deferral
- No net new findings — three-way convergence achieved

### Round 3 — Claude self-review

- Order verified: pnpm install checkpoint between tasks 8-9 catches devDep resolution before any config references them
- devDeps explicit in root package.json (task 4)
- Vacuous passes (dep-cruiser, licenses) documented as expected-at-M0
- No hidden scope creep (gitleaks/pre-commit/issue templates all explicitly deferred)

## 6. Local Verify Command

```bash
pnpm install --frozen-lockfile \
  && pnpm typecheck \
  && pnpm lint \
  && pnpm test \
  && pnpm format:check \
  && pnpm boundary-check \
  && pnpm licenses:check
```

Exit 0 on every step = G1 passes.

## 7. Out of Scope for M0

- Runtime code (graph nodes, handlers, ledger) — M1+
- MCP server — M2
- Agent transcripts — M3
- Dogfooding — M3 close
- Gitleaks + `pnpm audit` CI gate — M1
- Release artifacts (CHANGELOG, SECURITY.md, dependabot) — M6

## 8. Evaluation Protocol (post-WORK)

1. Run local verify command above.
2. Push to branch, observe CI on GitHub Actions.
3. Confirm all 7 CI jobs green on `ubuntu-latest`.
4. Record duration (baseline for future perf NFRs).
5. If green → emit M0 evaluation doc + tag `m0-complete`.
6. If red → root-cause, patch, re-verify. No `--no-verify` escapes.
