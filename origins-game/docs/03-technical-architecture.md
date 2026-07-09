# 03 — Technical Architecture

> The engineering foundation. Written to be true in five years, not just at launch. Where multiple
> approaches exist, this doc **compares and recommends**.

---

## 1. Architectural principles (the non-negotiables)

1. **Server-authoritative.** The server owns all truth. Clients render and request. This is the only
   defensible posture at millions of visits.
2. **Data-driven content.** Resources, recipes, buildings, research, disasters are **Definitions**
   (data), not code. Adding content = adding data. This is the primary expandability mechanism.
3. **Modular services with clear ownership.** Every piece of state has exactly one owning service.
   No shared mutable globals.
4. **Single source of time.** One simulation clock drives all time-based logic and persistence.
5. **Persistence is a first-class system, not an afterthought.** (See [Saving](./08-saving-system.md).)
6. **Everything typed.** Strict Luau + shared type definitions catch whole classes of bugs before
   runtime.
7. **Test the pure logic.** Game rules live in pure, testable modules; Roblox instances are the thin
   shell around them.

## 2. High-level shape

```
                         ┌─────────────────────────────────────────┐
                         │            ReplicatedStorage             │
                         │  Shared: Definitions · Net · Types ·     │
                         │  Utilities · Packages                    │
                         └───────────────┬─────────────────────────┘
                                         │ (both sides require shared)
         ┌───────────────────────────────┴───────────────────────────────┐
         ▼                                                                 ▼
┌──────────────────────┐   validated RPC (Net)   ┌───────────────────────────────┐
│      SERVER          │  ◄───── request ──────   │           CLIENT              │
│  ServerScriptService │   ────  state  ──────►   │  StarterPlayerScripts        │
│                      │        (deltas)          │                               │
│  Services:           │                          │  Controllers:                 │
│  Gathering, Inventory│                          │  Input, UI, Placement preview,│
│  Crafting, Building, │                          │  Camera, Effects, Social      │
│  Farming, Research,  │                          │                               │
│  Disaster, WorldSim, │                          │  UI framework (reactive)      │
│  Progression,        │                          │                               │
│  PlayerData, World-  │                          └───────────────────────────────┘
│  Save, Social        │
│                      │
│  DataStore / Memory  │  ◄── persistence ──►  (World + Player saves)
└──────────────────────┘
```

## 3. Framework choice — Services & Controllers

**The question:** how do we organize server/client code into modules that find and talk to each
other, without a tangle of `require` spaghetti?

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **Ad-hoc scripts + `require`** | No dependency to learn | No lifecycle, no ordering, no discovery; rots fast; untestable | ❌ Not for a 5-year project |
| **Knit** (popular framework) | Batteries included: services/controllers, networking, lifecycle; well known | Extra abstraction; networking opinions may fight our typed Net layer; maintenance is community-dependent | ⚠️ Viable |
| **Lightweight in-house framework** (thin service-locator + lifecycle) | Full control, minimal, fits our Net/Definitions exactly, no external risk, easy to teach | We maintain it; small upfront cost | ✅ **Recommended** |

**Recommendation: a minimal in-house framework** — a **service loader** that discovers modules,
resolves dependencies, and runs a two-phase lifecycle (`Init` then `Start`) on both server and
client. It is ~a few hundred lines, has no external dependency risk, and integrates cleanly with our
own typed **Net** and **Definitions**. If the team prefers batteries-included, **Knit is the
fallback**; the service/controller *shape* is identical either way, so this is reversible.

**Server = Services** (own state + logic). **Client = Controllers** (own input, presentation,
prediction). **Shared = Definitions, Net contract, Types, Utilities.** Details in
[module architecture](./06-module-architecture.md).

## 4. Data-driven design (the expandability engine)

Content is declared as **Definition tables** in `ReplicatedStorage/Shared/Definitions`, e.g.:

- `ResourceDefinitions` — id, display, tool required, yield, respawn time.
- `RecipeDefinitions` — id, inputs, output, tier, station.
- `BuildingDefinitions` — id, cost, model ref, footprint, unlock tier, function hooks.
- `ResearchDefinitions` — graph nodes, prerequisites, costs, unlocks (structured as a **graph** even
  though V1 is linear).
- `DisasterDefinitions` — id, weight, cooldown, duration, effect handler ref.

**Why this matters:** adding "Copper" or an "Earthquake" is a **data change reviewed like content**,
not an engineering task touching gameplay logic. Both client and server read the *same* shared
definitions, so there's one source of truth for balance and display. This directly answers the GDD
risk "adding a resource shouldn't mean a rewrite."

## 5. Simulation model

- **One fixed-timestep world tick** (`WorldSimulation`) accumulates real time and steps at a fixed
  cadence, decoupled from render frame rate. Subsystems (regrowth, farming, disasters) subscribe.
- **Time-based state stored as absolute timestamps**, computed lazily on read — so empty-server time
  and server restarts are handled correctly and cheaply (crop growth, node respawn).
- **Time-slicing:** heavy subsystems update on a cadence / in batches across ticks, never all at once.

Rationale: this is the standard scalable-simulation pattern; it keeps CPU flat as the world grows and
makes persistence math correct by construction.

## 6. Client/server boundary

- The client **never** mutates game state. It sends **intents** through the typed **Net** layer;
  the server validates and commits, then replicates **deltas** back.
- **Client-side prediction/preview** is presentation only (e.g., building ghost preview, tool swing
  animation) and always reconciled to server truth.
- Full contract in [networking](./07-networking.md).

## 7. Persistence model (summary)

- **Two stores:** `PlayerData` (per-player, session-locked) and `WorldSave` (per-world/server).
- **The defining decision:** persistent worlds are **private/reserved servers**, each keyed to its
  own `WorldSave`. Public servers are a shallow trial funnel. Full rationale + schema in
  [saving](./08-saving-system.md).
- **Versioned schemas + migrations** from day one.

## 8. Performance strategy

| Concern | Strategy |
|---------|----------|
| Large island, many instances | **StreamingEnabled**; server-spawned world; LOD |
| Thousands of nodes/structures | **Data-first, instance-second**; instance **pooling**; CollectionService tags |
| Per-frame cost | One tick + time-slicing; no per-node scripts; timestamp math |
| Network cost | **Delta replication**, batched/throttled events, rate-limited RPCs |
| UI cost | Reactive UI updated from deltas; pooled frames |
| Memory | Bounded build limits/zoning; compact serialization; stream out distant chunks |

## 9. Security & anti-exploit

- Server-authoritative + validate-everything at service boundaries.
- Single economy mutation chokepoint (`InventoryService`) for anti-dupe.
- Central **rate limiting** and payload validation in the Net layer.
- No remote grants trust to the client; no `loadstring`; sanitize all inbound payloads.

## 10. Tooling & language (see doc 10 for detail)

- **Luau strict mode**, shared `Types` module.
- **Rojo** to keep source in Git as `.luau` files, synced into Studio.
- **TestEZ / Jest-Lua** for pure-logic unit tests; **Selene** lint; **StyLua** format;
  **GitHub Actions** CI.

## 11. Key trade-off decisions, recorded

| Decision | Chosen | Why (short) |
|----------|--------|-------------|
| Authority | Server-authoritative | Only defensible at scale |
| Content | Data-driven Definitions | Expand without rewrites |
| Framework | In-house services (Knit fallback) | Minimal, fits our stack, reversible |
| Persistence unit | Private/reserved-server worlds | Ephemeral public servers break the core fantasy |
| Time | One fixed-step clock, timestamp state | Cheap, correct across restarts |
| UI | Reactive (Fusion/Roact) | Maintainable across many screens |
| Networking | Thin typed in-house Net | Central validation + anti-exploit |

---

**Prev:** [« 02 — Systems](./02-gameplay-systems.md) &nbsp;|&nbsp;
**Next:** [04 — Folder Structure »](./04-folder-structure.md)
