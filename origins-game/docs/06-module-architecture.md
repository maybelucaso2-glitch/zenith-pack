# 06 — ModuleScript Architecture

> The patterns every module follows so the codebase stays consistent, testable, and expandable over
> years and many contributors. This is the "how we write modules" contract.

---

## 1. The three module archetypes

| Archetype | Realm | Owns | Example |
|-----------|-------|------|---------|
| **Service** | Server | Authoritative state + logic for one slice | `GatheringService` |
| **Controller** | Client | Input + presentation + local prediction | `PlacementController` |
| **Shared module** | Both | Pure data or pure functions (no state, no side effects) | `RecipeDefinitions`, `Util.Spatial` |

**Rule:** state lives in exactly one Service. Shared modules are **pure** (data or stateless
functions). Controllers hold only *view/input* state, never game truth.

## 2. Service & Controller shape (lifecycle)

Both use a **two-phase lifecycle** run by the in-house framework loader:

```lua
-- Conceptual shape (illustrative, not final code)
local GatheringService = {}

GatheringService.Dependencies = { "InventoryService", "WorldSimulation" }

function GatheringService:Init()
    -- create state, register definitions. NO cross-service calls yet.
end

function GatheringService:Start()
    -- other services exist now: subscribe to ticks, bind remotes.
end

return GatheringService
```

**Why two phases (`Init` → `Start`):**
- `Init` runs on **all** modules first (set up own state, register), so there is **no load-order
  dependency** — a classic source of Roblox bugs.
- `Start` runs after every module is initialized, so cross-service references and event subscriptions
  are safe.

This is the single most important pattern for avoiding "works on my machine, breaks when files
reorder" fragility.

## 3. Dependency management

- Services **declare** dependencies (a list) and receive them from the loader (service-locator), or
  `require` shared modules directly.
- **No circular service dependencies.** If two services need each other, that's a design smell —
  extract the shared concern into a third module or communicate via events.
- **Shared modules never depend on Services** (keeps them pure and testable).

Dependency direction (allowed):
```
Controllers ──► Shared ◄── Services
     │                        │
     └──────► Net ◄───────────┘   (the only server↔client channel)
```

## 4. Definitions modules (data pattern)

Definitions are **plain data tables**, strongly typed, keyed by id:

```lua
-- Conceptual
export type ResourceDef = {
    id: string, display: string,
    toolRequired: string?, minToolTier: number,
    yield: number, respawnSeconds: number,
}

local Resources: { [string]: ResourceDef } = {
    Wood = { id = "Wood", display = "Wood", toolRequired = "Axe",
             minToolTier = 1, yield = 3, respawnSeconds = 45 },
    -- Stone, Iron, Wheat, Berries ...
}
return Resources
```

**Pattern rules:**
- Definitions contain **no behavior** — just data (ids, numbers, references, asset ids).
- Systems read Definitions; they never hardcode content values.
- Every Definition entry is **fully typed** so a malformed entry fails typecheck, not at runtime.
- Adding content = adding a keyed entry + (if a new *kind* of behavior) one handler module.

This is what makes "add a resource without a rewrite" literally true.

## 5. The Net module (server↔client contract)

- A **single registry** declares every remote and its argument/return **types**.
- A typed wrapper exposes `Net.Server` / `Net.Client` with:
  - **validation middleware** (payload shape + range/ownership checks) before any handler runs,
  - **rate limiting** per player per remote,
  - typed signatures so callers can't send the wrong shape.
- Systems bind handlers to named remotes; they never touch raw `RemoteEvent` instances.

Full behavior in [networking](./07-networking.md). Architecturally: **all cross-realm calls go
through Net** — there is no other sanctioned channel, which is what makes anti-exploit enforceable in
one place.

## 6. Pure-logic vs. Roblox-shell separation (testability)

Game **rules** are written as **pure functions/modules** that take state and return new state, with
**no Roblox instance access**. The Service is a thin shell that:
1. receives a validated request,
2. calls pure logic,
3. applies results to instances + replicates.

```
RPC ─► Service (shell) ─► pure rule module (testable) ─► result ─► apply to world + replicate
```

**Why:** pure modules are unit-testable with [TestEZ/Jest-Lua](./10-tooling-and-conventions.md)
**without a running game**. Over five years, a tested rules core is the difference between confident
refactors and fear.

## 7. Events & communication

- **Within a realm:** services communicate via a lightweight **signal/event** pattern (e.g., a
  `Signal` util) or direct calls for hard dependencies — never global state.
- **Across realms:** only via **Net**.
- **State → UI:** the client holds a **replicated state store**; UI components subscribe and re-render
  on change (reactive), rather than services imperatively poking GUI objects.

## 8. Error handling & resilience

- Service handlers wrap fallible work; a failed player request never crashes the server or corrupts
  state (validate first, mutate atomically).
- Persistence operations are guarded and retried (see [saving](./08-saving-system.md)); a failed save
  must never silently lose data.
- Prefer **fail-closed** on the server (reject ambiguous requests) and **fail-soft** on the client
  (show a message, reconcile to server truth).

## 9. Module checklist (applied in review)

A module is "right" when:
- [ ] It has one clear responsibility and owner.
- [ ] It follows the archetype (Service / Controller / pure Shared).
- [ ] State lives in exactly one place; shared modules are pure.
- [ ] It declares dependencies; no circular refs.
- [ ] Content values come from Definitions, not literals.
- [ ] Cross-realm calls go through Net with validation.
- [ ] Core rules are extracted into testable pure functions.
- [ ] It is strict-typed and passes lint/format.

---

**Prev:** [« 05 — DataModel](./05-datamodel-hierarchy.md) &nbsp;|&nbsp;
**Next:** [07 — Networking »](./07-networking.md)
