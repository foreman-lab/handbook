# handbook

Architecture, ADRs, and roadmap for [Foreman](https://github.com/foreman-lab/foreman) — an MCP harness for AI coding agents.

## What's here

- [`foundations.md`](foundations.md) — Foundational principles (P-1..P-10) and decisions (D-1..D-20). Locked baseline.
- [`roadmap.md`](roadmap.md) — Product tier progression (solo → team → cloud) and activation tripwires.
- [`architecture.md`](architecture.md) — Core architecture: state machine, layers, signal protocol, invariants.
- [`adr/`](adr/) — Architecture Decision Records (0001–0009).

## Reading order

1. [`foundations.md`](foundations.md) — the constitutional layer.
2. [`roadmap.md`](roadmap.md) — product tiers and progression.
3. [`architecture.md`](architecture.md) — the state machine + layer architecture.
4. [`adr/0001-architectural-style-declaration.md`](adr/0001-architectural-style-declaration.md) — the 7-pattern hybrid.
5. [`adr/0007-domain-blind-core.md`](adr/0007-domain-blind-core.md) — foreman composes prompt sections; agents render.
6. [`adr/0009-foreman-composes-prompt-sections-and-lazy-children.md`](adr/0009-foreman-composes-prompt-sections-and-lazy-children.md) — 6-section composition + lazy-load.

## Status

This handbook was seeded from a prior trial repo. Cleanup PRs are landing to:

- Strip milestone-specific (M0–M6) references from `architecture.md`, `error-codes.md`, `dogfooding-protocol.md`.
- Rewrite passages referring to `@foreman-lab/core` extraction — the new direction is **tiers-as-wire-protocol** (Docker model): higher tiers speak Foreman's MCP/IPC, they do not import core.
- Add new docs: `tiers-as-wire-protocol.md`, `binary-modes.md`, `solo-mode.md`.

See the repo's issue tracker for the cleanup backlog.

## License

Apache-2.0. See [LICENSE](LICENSE).

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md). Contributions require a DCO sign-off (`git commit -s`).
