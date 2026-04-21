# State Machine — Basic

A generic state machine module. Opaque to domain. Supports a **catalog** of machine definitions that grows at runtime and loads **lazily** from a persistence store. Deliberately minimal; every feature above the basic extends via documented seams rather than engine modifications. Playbook is one caller; the engine knows nothing about Playbook's 4-phase lifecycle, crafts, briefs, or any other domain term.

Tied to [`foundations.md`](foundations.md).

## The model

A **state** is a label for a position. Any string. Caller-defined.

A **state node** is a state plus metadata (is it terminal, and — later — entry/exit actions, timeouts). In graph terms, the vertex.

An **event** is an input to the machine: a type and an opaque payload.

A **transition** is a rule mapping `(from state, event type) → to state`. In graph terms, the edge.

A **machine definition** is the full graph: initial state, state nodes, transitions. Identified by `(id, version)` the caller chooses (semver, hash, anything).

A **machine** is a stateful entity traversing a graph. Has an id, current state, context, metadata, a reference to the definition it runs under (`definitionId`, `definitionVersion`), and a `revision` counter for optimistic concurrency.

The engine maintains a **definition catalog** that grows over time. Definitions are **persisted** in the store, **registered** into the catalog at runtime, and **loaded lazily** when a machine needs one.

## Engine vs orchestrator

The engine advances **one machine at a time**. Multi-machine orchestration — tree, DAG, pipeline, parallel — lives in an orchestrator above the engine. Playbook's tree orchestrator is one such caller; tests, debuggers, future tools are others.

The engine's job is fixed: "advance one machine's state when an event arrives, using whichever definition that machine belongs to." Everything else is caller work.

## Scope

**In**

- Generic single-machine finite state machine per dispatch.
- Caller-defined states (any string set) and event types (any string set).
- First-class machine definitions: initial state, state nodes, transitions.
- **Definition catalog** — many definitions per engine.
- **Runtime registration** — new definitions added while the engine runs.
- **Lazy loading** — definitions load from the store on demand.
- **Definition versioning** — each machine pinned to `(definitionId, definitionVersion)`; no silent upgrades.
- **Optimistic concurrency** — each machine has a monotonic `revision`; conflicting writes are rejected.
- Opaque payloads on events; engine never reads content.
- One outbound port: `Store` (save / load definitions and machines).
- Three use cases: `registerDefinition`, `startMachine`, `dispatch`.

**Out (added in later docs)**

1. **Router** — caller-provided port for dynamic transitions (retry loops, blocks, joins, approvals, timeouts all compose from it).
2. **Entry / exit actions** — state- and transition-level side effects.
3. **Guards** — boolean predicates on transitions.
4. **Definition migration** — rules for advancing a machine from older definition versions.
5. **Playbook's tree orchestrator** — pipe, expand, DFS — separate doc at orchestrator level.
6. **Journal and replay** — crash-recovery log.
7. **Parallel step processing** — concurrent machines under structured concurrency.
8. **Transport layers** — MCP, CLI, or any adapter that translates inbound calls into engine invocations.

Each gets its own architecture doc when its turn comes.

## Domain

### State and transition (the model)

We split the state machine into two ideas, the same way LangGraph does — but simpler.

**State** is data. **Transition** is movement.

The basic engine simplifies both:

- **State** is one channel: `state`. `context` carries content alongside but doesn't drive transitions.
- **Transition** is one pure function: `transition(machine, event, definition) → machine`. Rules are a flat table.

### Generic types

```ts
type State = string;

type Event = {
  type: string;
  payload: unknown;
};
// Callers MAY define a superset of Event (e.g., add `timestamp`, `correlationId`)
// as long as `type` and `payload` remain present. The engine reads only those two.

type StateNode = {
  id: State;
  terminal?: boolean;
  // future: entry?, exit?, meta?, timeout?
};

type Transition = {
  from: State;
  event: string;
  to: State;
};

type MachineDefinition = {
  id: string;
  version: string;
  initialState: State;
  states: StateNode[];
  transitions: Transition[];
};

type Machine = {
  id: string;
  definitionId: string;
  definitionVersion: string;
  revision: number;                 // starts at 0; engine increments on each save
  state: State;
  context: Record<string, unknown>;
  metadata: Record<string, unknown>;
};
```

Every field is either primitive or `unknown`. The engine does not know Playbook's specific states, event types, or anything about `payload` content.

### The step (how one event is processed)

Every `dispatch` is three generic operations:

1. **Update.** Store the event's payload into `context[event.type]`. Mechanical.
2. **Transition.** Look up `(machine.state, event.type)` in the definition's transitions; return the next state.
3. **Increment.** Bump `machine.revision` by 1 so the next save enforces the concurrency contract (see §Concurrency).

Combined in one pure function:

```ts
function transition(
  machine: Machine,
  event: Event,
  definition: MachineDefinition,
): Machine;
```

Throws if no row matches, or if `machine.state` is a terminal node.

The engine resolves `definition` on each dispatch by looking up the machine's `definitionId` and `definitionVersion` in the catalog, loading from the store lazily if not already cached.

Later features extend step 2 with a **Router port** — at specific rows, the engine asks a caller-provided router for the next state instead of reading it from the row.

### Concurrency

The engine treats dispatches as **per-machine serial**: at most one `dispatch` per `machineId` is in flight at a time. Enforced via optimistic concurrency on the `Store`:

- `Machine.revision` starts at `0` on `startMachine`.
- Each successful `dispatch` increments it by 1 before save.
- `Store.saveMachine` enforces that the stored revision equals `machine.revision - 1` at save time. If not, it throws a conflict error.

On conflict, the caller retries (reload, re-apply, re-save) or fails. The engine itself does not retry — that's a caller policy.

Callers may still need to serialize externally if they want to avoid conflict errors entirely (e.g., a lock per `machineId` at the orchestrator level). The revision field is the safety net; the orchestrator is the first line.

### Definition catalog and lazy loading

The engine holds an in-memory **catalog** keyed by `(definitionId, definitionVersion)`. The catalog is:

- **Lazy**: populated on demand. When a machine is dispatched and its `(definitionId, definitionVersion)` isn't in the catalog, the engine calls `Store.loadDefinition(...)` and caches the result.
- **Runtime-mutable**: callers register new definitions at any time via `registerDefinition`. Registration saves to the store and adds to the catalog.
- **Version-precise**: multiple versions of the same `id` can coexist.

Cache policy is adapter-agnostic in the basic version: caches forever unless explicitly evicted. A future revision may add eviction (LRU, TTL) without changing the external API.

## Application layer

### Use cases

```ts
interface Engine {
  // Add or update a definition. Saves to the store and catalogs it.
  registerDefinition(definition: MachineDefinition): Promise<void>;

  // Create a new machine under a specific definition at its initialState.
  // Does NOT apply an event. Use dispatch for the first state change.
  startMachine(args: {
    machineId: string;
    definitionId: string;
    definitionVersion: string;
    initialContext?: Record<string, unknown>;   // optional bootstrap data
    initialMetadata?: Record<string, unknown>;  // optional bootstrap caller-owned bag
  }): Promise<Machine>;

  // Advance an existing machine. Resolves its definition lazily from the catalog.
  dispatch(machineId: string, event: Event): Promise<Machine>;
}
```

**`registerDefinition`** — idempotent on `(id, version)`. Saves to the store, updates the catalog.

**`startMachine`** — fails if `machineId` already exists. Loads the specified definition (cache → store → error), creates a machine at `definition.initialState`, `revision: 0`, stamped with `(definitionId, definitionVersion)`, saves, returns. No event is applied; the machine is ready to receive its first real event via `dispatch`.

**`dispatch`** — loads machine, looks up its definition (cache → store → error), applies `transition(...)`, bumps `revision`, saves with optimistic check, returns. Fails if the machine doesn't exist, the event has no matching transition, the machine is in a terminal state, or the revision conflicts.

### Outbound port: `Store`

```ts
interface Store {
  // Definitions (the graphs)
  saveDefinition(definition: MachineDefinition): Promise<void>;
  loadDefinition(id: string, version: string): Promise<MachineDefinition | null>;

  // Machines (the instances)
  saveMachine(machine: Machine): Promise<void>;
  // Behavior:
  //   - If no machine exists with this id: insert (require machine.revision === 0).
  //   - If a machine exists: update only if stored.revision === machine.revision - 1.
  //   - Otherwise: throw a ConcurrencyConflictError.
  loadMachine(machineId: string): Promise<Machine | null>;
}
```

`saveDefinition` is idempotent on `(id, version)`.

Two implementations ship (per `D18`: ≥2 implementations to justify the port):

- `MemoryStore` — Map-backed; for tests.
- `SqliteStore` — `better-sqlite3`-backed; default for real use.

Both JSON-serialize definitions and machines on save.

## Hexagonal layout

```
   Caller
     │
     ▼
   ┌────────────────────────────────────────────┐
   │  Application                               │
   │    registerDefinition · startMachine       │
   │    dispatch                                │
   │                                            │
   │    (definition catalog, lazy-loaded)       │
   └────────────────┬───────────────────────────┘
                    │
                    ▼
   ┌────────────────────────────────────────────┐
   │  Domain (pure)                             │
   │    transition(machine, event, definition)  │
   └────────────────────────────────────────────┘
                    ▲
                    │ uses
   ┌────────────────┴───────────────────────────┐
   │  Outbound port                             │
   │    Store                                   │
   └────────────────┬───────────────────────────┘
                    │ implemented by
                    ▼
   ┌────────────────────────────────────────────┐
   │  Adapters                                  │
   │    MemoryStore · SqliteStore               │
   └────────────────────────────────────────────┘
```

The catalog lives in the application layer — a cache in front of the `Store` port.

## Composition root

```ts
async function createEngine(config: {
  store: Store;
  definitions?: MachineDefinition[];   // optional bootstrap set
}): Promise<Engine>;
```

Behavior:

1. If `definitions` is provided, call `registerDefinition` for each.
2. Returns an `Engine` exposing `registerDefinition`, `startMachine`, `dispatch`.

## Sequences

### A. `startMachine`

```mermaid
sequenceDiagram
    autonumber
    participant C as Caller
    participant E as engine.startMachine
    participant Cat as catalog
    participant S as Store

    C->>E: startMachine(args)
    E->>Cat: lookup(definitionId, version)
    alt cache miss
        E->>S: loadDefinition(id, version)
        S-->>E: definition
        E->>Cat: put(definition)
    end
    E->>S: loadMachine(machineId)
    S-->>E: null
    E->>E: build machine at initialState, revision: 0
    E->>S: saveMachine(machine)
    E-->>C: machine
```

### B. `dispatch` with warm cache

```mermaid
sequenceDiagram
    autonumber
    participant C as Caller
    participant E as engine.dispatch
    participant Cat as catalog
    participant S as Store
    participant Dom as transition

    C->>E: dispatch(machineId, event)
    E->>S: loadMachine(machineId)
    S-->>E: machine (revision R)
    E->>Cat: lookup(definitionId, version)
    Cat-->>E: definition
    E->>Dom: transition(machine, event, definition)
    Dom-->>E: new machine (revision R+1)
    E->>S: saveMachine(new machine)
    S-->>E: ok (or ConcurrencyConflictError if stored != R)
    E-->>C: new machine
```

### C. Runtime registration

```mermaid
sequenceDiagram
    autonumber
    participant C as Caller
    participant E as engine.registerDefinition
    participant S as Store
    participant Cat as catalog

    C->>E: registerDefinition(def)
    E->>S: saveDefinition(def)
    Note over S: idempotent on (id, version)
    E->>Cat: put(def)
    E-->>C: ok
```

## Extension points

The engine is deliberately minimal. Every future capability extends through one of these seams — none require modifying the engine's core.

1. **Generic types.** States, events, context, and metadata are all caller-defined data. The engine carries them, never interprets them.
2. **Open event envelope.** `Event` requires `{type, payload}`. Callers may carry extra fields (`timestamp`, `correlationId`, `source`, etc.) by declaring a superset type in their own code. The engine reads only `type` and `payload`; it neither validates nor strips the rest.
3. **`Store` as a port.** Any adapter that satisfies the interface works. Callers can **wrap** the port (decorator pattern) to add logging, metrics, encryption, retry, or caching without touching the engine or its shipped adapters.
4. **Caller-owned `metadata` bag.** Every machine carries `metadata: Record<string, unknown>`. The engine never reads or writes its contents. Callers use it for iteration counters, deadlines, approval state, correlation data — anything specific to their domain.
5. **Declared future ports.** The Out-of-scope list names the expected extension hooks (`Router`, `Actions`, `Guards`). Each becomes an optional port when it ships. Engines constructed without these ports fall back to the basic behavior; engines constructed with them gain the feature. Backward-compatible by construction.
6. **Immutable bedrock.** The types, ports, and invariants documented here are the stable API. New ports are added in minor versions; fields can be added in minor versions; renames and removals require a major version and an ADR. Callers can rely on this contract.

What this rules out (deliberately):

- **Monkey-patching** — the engine doesn't expose internals for runtime mutation.
- **Subclassing** — the engine exposes interfaces; there's no base class to extend. Composition over inheritance.
- **Plugins with unchecked access** — all extension is through typed ports or typed data. No stringly-keyed plugin registry.

## Invariants

- **I-1.** Domain imports nothing outside `src/domain/`.
- **I-2.** `transition` is pure: same `(machine, event, definition)` → same result.
- **I-3.** An event is either applied (machine saved) or rejected (no state change).
- **I-4.** Every outbound port has ≥2 implementations.
- **I-5.** No `any` in domain; Zod gates every event's and definition's shape.
- **I-6.** Engine never reads `payload`, `context`, or `metadata` content. They pass through unchanged.
- **I-7.** Engine never reads state labels for semantics; it only compares them as strings.
- **I-8.** Every machine carries `(definitionId, definitionVersion)`. Dispatch uses that exact version; no silent upgrades.
- **I-9.** `saveDefinition` and `registerDefinition` are idempotent on `(id, version)`.
- **I-10.** The catalog is a cache: any entry must be reconstructible from the store. Clearing the catalog never loses data.
- **I-11.** `Machine.revision` is monotonically non-decreasing across the machine's lifetime. Every successful `dispatch` increments it by exactly 1. Conflicting writes are rejected.

## Tests we expect

- **Domain tests** — `transition` against tables of `(state, event, definition) → state`. Pure, no I/O.
- **Use-case tests** — `registerDefinition`, `startMachine`, `dispatch` with `MemoryStore`.
- **Adapter contract tests** — run against `MemoryStore` and `SqliteStore`. Cover definition save/load, machine save/load, idempotent definition save, optimistic concurrency conflict.
- **Payload-opacity tests** — round-trip arbitrary payload shapes.
- **State-opacity tests** — configure with arbitrary random state labels; behavior unchanged.
- **Event-envelope extensibility tests** — dispatch events with extra fields beyond `{type, payload}`; assert the engine ignores them.
- **Lazy-loading tests** — dispatch a machine whose definition is only in the store; catalog populates from the store once.
- **Runtime-registration tests** — start with empty catalog; register at runtime; dispatch against it.
- **Multi-version tests** — two versions of one id coexist; machines advance under their respective versions.
- **Concurrency-conflict tests** — simulate a stale write; assert the conflict is raised and no state changes.
- **Terminal-state tests** — events into terminal machines are rejected.
- **Missing-definition tests** — dispatch against a machine whose definition is neither cached nor stored; clean error.
- **Store-decorator test** — wrap `MemoryStore` with a logging decorator; engine behavior unchanged; log captures calls.

## Appendix: Playbook's configuration (example)

Playbook authors let users create playbooks at runtime. Each playbook is a machine definition registered into the engine's catalog.

```ts
const engine = await createEngine({
  store: new SqliteStore("./.playbook.sqlite"),
  // no pre-registered definitions; all arrive at runtime
});

// user creates a new playbook → orchestrator registers its definition
await engine.registerDefinition({
  id: "playbook.user-123.tdd-workflow",
  version: "1",
  initialState: "Initializing",
  states: [
    { id: "Initializing" },
    { id: "Planning" },
    { id: "Working" },
    { id: "Evaluating" },
    { id: "Completed", terminal: true },
  ],
  transitions: [
    { from: "Initializing", event: "plan", to: "Planning" },
    { from: "Planning",     event: "work", to: "Working" },
    { from: "Working",      event: "eval", to: "Evaluating" },
    { from: "Evaluating",   event: "eval", to: "Completed" },
  ],
});

// orchestrator starts a machine at the definition's initialState (no event applied)
const machine = await engine.startMachine({
  machineId: "run-abc",
  definitionId: "playbook.user-123.tdd-workflow",
  definitionVersion: "1",
  initialContext: { brief: /* caller-defined */ },
});
// machine.state === "Initializing", revision === 0

// orchestrator then dispatches the first event to advance
await engine.dispatch("run-abc", { type: "plan", payload: /* caller-defined */ });
// → state === "Planning", revision === 1
```

Playbook's `Brief`, `Plan`, `Work`, `Eval` payload shapes live in Playbook's domain — the engine sees them only as `unknown`.

## How this changes

When a future feature in the "Out" list begins implementation, write a new architecture doc that adds it on top of this one. Each new doc states what it changes (types, ports, invariants), proposes the deltas, and lands as a PR alongside the code. This document stays as the bedrock — features extend via the documented extension points. No feature introduces domain-specific concepts into the engine. The engine stays payload-blind, state-blind, and domain-blind.
