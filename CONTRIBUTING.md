# Contributing

Thanks for your interest in improving the Foreman handbook.

## Scope

This repo holds architecture, ADRs, and roadmap. Code belongs in [`foreman-lab/foreman`](https://github.com/foreman-lab/foreman).

## How to propose changes

1. Open an issue describing the change (architectural amendment, new ADR, cleanup, etc.) before large edits.
2. Fork, branch, edit, open a PR.
3. Small typo/clarity fixes can skip the issue and go straight to a PR.

## ADR protocol

New ADRs follow the format of [`adr/0001-architectural-style-declaration.md`](adr/0001-architectural-style-declaration.md):

- Numbered sequentially (next number = max existing + 1).
- Status: `Proposed` → `Accepted` / `Rejected` / `Superseded`.
- Context, Decision, Consequences sections.
- Link from any docs the ADR affects.

Amendments to an accepted ADR require either a follow-up ADR (preferred) or an inline amendment block with a dated note.

## Developer Certificate of Origin (DCO)

Every commit must be signed off:

```
git commit -s -m "your message"
```

This appends `Signed-off-by: Your Name <your@email>` to the commit, certifying the [Developer Certificate of Origin v1.1](https://developercertificate.org/). By signing off, you agree that:

- You have the right to submit the change under the project's license.
- You understand the commit and sign-off are public.

## License

By contributing, you agree your contributions are licensed under Apache-2.0.

## Code of Conduct

See [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md).
