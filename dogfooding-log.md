---
status: Open — M3 Solo Alpha dogfooding log
owner: Operator (xuzhijie)
baselined: 2026-04-18
grounding: spec §15.2 (log format + canary-not-gate boundary); dogfooding-protocol.md §13 (template)
---

# Foreman Dogfooding Log — M3 close → M6

Per spec §15.2: this log captures friction encountered while using foreman to track foreman's own M4+ work. Format per `docs/dogfooding-protocol.md` §13. Out-of-scope ideas/features belong in `BACKLOG.md`, not here.

**Canary, not gate** (spec §15.2): if foreman breaks here, file a bug + work around it manually; do not edit the vitest suite to accommodate the bug.

---

### 2026-04-18 — `foreman init` in repo: first dogfooding act

**Context:** M3 Batch H Round 10 GAN (Codex) flagged `.foreman/` absent as a BLOCKER per §9 PASS #7 ("`.foreman/` dogfooding initialized in repo").

**Expected:** `foreman init` creates `.foreman/{checkpoints.db, config.json}` in the repo root; idempotent on re-run; envelope `{ok:true, data:{foremanDir, created:true}}`.

**Actual:** `.foreman/` created successfully at first try via `pnpm exec tsx packages/solo/src/cli/bin.ts init`. Envelope matched expectation. `.foreman/` is gitignored (already in `.gitignore`), so no accidental commit risk.

**Friction observed:** `pnpm exec tsx -e "<inline script>"` failed twice before I used `tsx packages/solo/src/cli/bin.ts init` — the CLI bin entry is the documented path for non-installed invocation. Minor ergonomic wrinkle; not a foreman bug.

**Hypothesis:** the inline-script approach needed absolute imports; bin.ts resolution is the intended dev-mode invocation. No code fix needed.

**Fix:** none — documenting the idiom for future dogfooders in `docs/dogfooding-protocol.md` would help. (Post-tag followup.)

**Related:** commit pending for H.4 tag-readiness remediation.
