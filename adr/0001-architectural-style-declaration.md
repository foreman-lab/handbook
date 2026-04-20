# ADR 0001: Architectural Style Declaration

**Status:** Accepted
**Date:** 2026-04-17
**Deciders:** Operator (xuzhijie), after debate with Copilot
**Supersedes:** —
**Superseded by:** —

---

## Context

During an SDLC best-practice review of the three foundational documents (`foundations.md`, `roadmap.md`, `architecture.md`), a gap was identified: while the architecture implements multiple design patterns (Hexagonal, Layered, Command Bus, Event Sourcing, CQRS, Finite State Machine, Dependency Injection), none were formally declared in a single place. Fragments existed (§2.2 heading said "minimum viable hexagonal"; §3.1 had one "CQRS-style" parenthetical), but no dedicated section named the overall architectural style.

Per SDLC best practice (arc42 §4 Solution Strategy, C4 model, 4+1 view), naming the architectural style explicitly:

- Provides a reference frame for downstream decisions (Phase 1 spec can point here; future ADRs can cite patterns)
- Helps future contributors reconstruct intent without archaeology
- Surfaces drift early — a change to the named style is now a visible event requiring an ADR

Foreman uses a **hybrid** style. No single pattern addresses all concerns — hexagonal for test isolation, layered for change-axis separation, command bus for the agent contract, event sourcing for audit trail, CQRS for read/write split, FSM for the business protocol, DI for tier variation.

## Decision

Amend `docs/architecture.md` with a new **§1.1 "Architectural Style"** that explicitly names the 7 design patterns and explains the hybrid rationale. Existing subsections §1.1 through §1.5 renumber to §1.2 through §1.6.

Architecture.md status line updates from _"Locked — Phase 1 baseline"_ to _"Locked — Phase 1 baseline; amended by ADR 0001"_.

## Consequences

### Positive

- Future contributors have a single reference frame for why the code is shaped the way it is
- Phase 1 Solo architecture doc (upcoming at `docs/phase-1-solo/architecture.md`) can reference §1.1 directly in its arc42 §4 Solution Strategy section
- Drift detection becomes explicit — amending style requires a follow-up ADR
- Anti-patterns are named (microservices, service locator, shared mutable state, inheritance-over-composition), preventing future accidental adoption

### Negative

- Breaks the lock on `architecture.md` that was set earlier the same day (~1 hour prior)
- Requires renumbering five existing subsections of §1; if external refs existed they would need updating (none do — verified via grep)
- Adds ~80 lines to a locked doc

### Neutral

- No code impact (pre-implementation)
- No breaking change to other docs; only `architecture.md` is amended
- No cross-reference breakage (verified: no doc cites `§1.1`..`§1.5` numerically)

## Alternatives considered

### Option B — Add as D-21 in foundations.md (rejected)

Architectural style is not a tooling/interface decision — it belongs with system design. Adding D-21 would inflate foundations.md and muddy the decision hierarchy (Tier A/B/C/D are about language, types, interfaces, structure — not style). Style is an architecture-doc concern.

### Option C — Defer to `phase-1-solo/architecture.md` §4 (rejected)

Architectural style applies to **all four tiers** (Solo, Daemon, Team, Enterprise), not just Phase 1. Declaring it only in the Phase 1 doc misplaces a cross-tier concern. The Phase 1 doc's arc42 §4 will reference the declaration here, not originate it.

## References

- `docs/foundations.md` D-17 (Handler/Transport Separation), D-20 (Outbound port discipline)
- `docs/architecture.md` §2.2 (existing minimum-viable-hexagonal declaration, to be formalized in §1.1)
- arc42 template §4 (Solution Strategy)
- C4 model (Simon Brown)
- 4+1 view model (Philippe Kruchten)
- Skill: `everything-claude-code:hexagonal-architecture` (used during the SDLC review that surfaced this gap)
