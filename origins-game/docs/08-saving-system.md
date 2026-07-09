# 08 — Saving System Design

> **The most important architectural document in the project.** A civilization game is defined by
> persistence: if the world doesn't reliably remember what players built, nothing else matters. This
> doc explains the defining decision (persistent worlds), the two data stores, and the rules that
> keep data safe.

---

## 1. The defining problem: Roblox servers are ephemeral

Roblox does **not** give you a persistent server. A public, matchmade server **shuts down when it
empties**, and its in-memory world is gone. Players are matched into *whatever* server has room, so
there is no stable "our world" to return to.

**This is fatal to ORIGINS' core fantasy** ("build a civilization over days, weeks, months") unless
we architect around it. Persistence is not a save file we bolt on — it is the shape of the whole
product.

## 2. The defining decision: a **World** is a persistent private/reserved server

**A "world" = a reserved/private server instance keyed to its own persistent save.** The world's
entire state lives in a DataStore under a stable **World ID**; whenever that world's server boots, it
**loads** its state; while running, it **autosaves**; on shutdown, it **flushes**.

```
World ID  ──►  reserved server  ──►  loads WorldSave(WorldId) on boot
                                 ──►  autosaves periodically + on events
                                 ──►  flushes on BindToClose (shutdown)
Next time anyone opens that world ──►  same reserved server key ──►  same WorldSave ──► continuity
```

**How players get a persistent world (product/UX):**
- A player/group **owns or creates a World** (a reserved server via `TeleportService:ReserveServer`,
  its access code stored server-side and tied to the World ID). Friends join *that* world.
- Owning persistent worlds is a natural, **non-pay-to-win** monetization surface (world slots), fully
  consistent with the [GDD](./00-game-design-document.md#8-monetization-philosophy-non-pay-to-win).

**Public "demo" servers** may exist as a **shallow trial** (progress may reset / not persist long
term) to funnel players toward creating a real persistent world. They are the exception, not the
model.

### Why not "just save per player" like most Roblox games?

Because ORIGINS' progress is **shared world state**, not per-player state. The island, structures,
research tier, farms, and disaster scars belong to the *world*, and every member must see the same
truth. So we need a **world-scoped save** in addition to per-player profiles — this is the core
structural difference from a typical Roblox game, and the reason persistence is a headline system.

### Alternatives considered

| Approach | Why not (for V1) |
|----------|------------------|
| Per-player-only saves (no world save) | Can't represent a shared evolving island; breaks the core fantasy |
| External backend DB (OrderedDataStore-bypass, HTTP to own server) | Powerful later, but heavy ops burden and cost; DataStore is sufficient for V1 |
| Single global world (one DataStore key for all servers) | Can't scale; contention; every server would fight over one world |
| **Reserved-server world + DataStore per World ID** | ✅ Fits Roblox primitives, scales horizontally (one key per world), matches the fantasy |

## 3. Two stores, two lifecycles

| Store | Scope | Keyed by | Holds |
|-------|-------|----------|-------|
| **PlayerData** | Per player | UserId | Personal inventory, equipped tools, cosmetics, settings, cross-world identity |
| **WorldSave** | Per world | World ID | Civilization tier, research progress, structures, farms, resource-node state, shared Storage, world clock, disaster state |

They are saved and loaded independently, but a player's world-scoped bits (e.g., their claimed
house) are recorded in **WorldSave** (they belong to the world), while account-level bits live in
**PlayerData**.

## 4. Data integrity rules (non-negotiable)

1. **Session locking.** Both stores use **session locking** (ProfileStore/ProfileService pattern) so
   two servers can never hold the same key at once — this is what prevents the classic Roblox
   **item-loss and duplication** bugs. A world/profile is *locked* to the server using it.
2. **Load → hold in memory → save.** Never read-modify-write DataStore on every change. Load once,
   mutate the in-memory copy (validated through services), persist on a cadence.
3. **Autosave cadence.** Periodic autosave (e.g., every N seconds) **+ on significant events**
   (tier-up, major build) **+ on player leave** (their bits) **+ `game:BindToClose`** (flush world on
   shutdown, with the mandated shutdown delay).
4. **Atomic & validated.** All state changes go through services (never ad-hoc), so saved data is
   always internally consistent (no negative counts, no orphan structures).
5. **Retry with backoff.** DataStore calls are wrapped with retries/backoff; a transient failure must
   never silently drop data.
6. **Never lose on failure.** If a save fails at shutdown, prefer keeping the lock/retrying over
   releasing to a corrupt state.

## 5. Versioned schema + migrations (from day one)

Over five years the save format **will** change. We plan for it now:

- Every saved blob carries a **`schemaVersion`**.
- On load, if `schemaVersion < current`, run ordered **migration functions**
  (`v1→v2→v3…`) to bring old saves forward before the game uses them.
- Migrations are **pure, tested functions**. This means an old world from launch day still loads
  years later — essential for a persistent-world game where worlds outlive versions.

```
load blob ─► read schemaVersion ─► run migrations up to current ─► validate ─► hand to services
```

**Never** rename/repurpose a field in place without a migration. Additive-first schema design.

## 6. Serialization strategy

- **Definitions are not saved** — only **references** (ids) + **instance state** are. A structure is
  saved as `{ defId, cframe, ownerId, health, state }`, not its model. On load, the `defId` looks up
  the current Definition and spawns the model. This keeps saves tiny and lets content/balance change
  without breaking old worlds.
- **Compact shapes.** Structures/nodes stored as arrays of small records; avoid verbose keys.
- **Chunking for large worlds.** A single DataStore value is size-limited, so a mature world's data
  is **partitioned** (e.g., structures chunked by region, or split across multiple keys under the
  World ID) and reassembled on load. Design the WorldSave as a set of parts from the start so growth
  never hits the value-size ceiling.
- **Timestamp-based world state.** Crop growth and node respawn are stored as **absolute timestamps**
  (see [simulation](./02-gameplay-systems.md#simulation)), so on reload the world can compute elapsed
  time and even **simulate offline progress** (crops that finished while the world was closed).

## 7. Save/load lifecycle (world)

```
SERVER BOOT (reserved world)
  └─► WorldSaveService:Load(WorldId)
        acquire session lock → read parts → migrate → validate
        → hydrate services (Research tier, Buildings, Farms, Nodes, Storage, Clock, Disaster)
        → spawn world instances from data → world is live

WHILE RUNNING
  └─► autosave every N s + on tier-up/major events
        serialize live service state → write parts (retry/backoff)

PLAYER LEAVES
  └─► PlayerDataService:Save(player) (their profile) ; their world bits already in WorldSave

SERVER SHUTDOWN (BindToClose)
  └─► flush WorldSave + all PlayerData, release locks (within shutdown grace window)
```

## 8. Player data lifecycle

```
JOIN   ─► PlayerDataService:Load(userId) [session-locked] ─► send profile view to client
PLAY   ─► services mutate in-memory profile (validated) ─► periodic autosave
LEAVE  ─► save + release lock
SHUTDOWN ─► BindToClose flush all
```

## 9. Failure modes & safeguards

| Failure | Safeguard |
|---------|-----------|
| DataStore transient error | Retry with exponential backoff; keep lock |
| Server crash before save | Frequent autosave bounds loss to the interval; timestamps recover time-based state |
| Two servers, one world | Session locking refuses the second; no split-brain |
| Corrupt/oversized blob | Chunked parts + validation on load; last-good fallback per part |
| Schema drift | Versioned migrations run on load |
| Dupe attempt via rejoin | Session-locked profiles + server-authoritative economy |
| Shutdown mid-save | `BindToClose` grace window + retry before release |

## 10. Scalability

- **One key(-set) per World ID = horizontal scale.** Ten thousand worlds are ten thousand
  independent save scopes; no global contention.
- **DataStore request budgets** respected via autosave cadence + batching (not per-change writes).
- **MemoryStore** (future) for transient cross-server data (presence, world directory, global
  events) — not needed for V1 persistence.

## 11. Future expansion

- Backups/versioned world snapshots (roll back a griefed/disaster-wrecked world).
- World "chronicle" — a saved history log powering the "world tells its story" pillar.
- External analytics/backup pipeline (HTTP to owned service) once ops maturity supports it.
- Cross-world player identity & account-level inventory (MemoryStore + PlayerData split).
- Offline-progress simulation on load (advance farms/regrowth for elapsed closed time).

## 12. Recommended implementation summary

- **`PlayerDataService`** — session-locked profiles via **ProfileStore** (or ProfileService), the
  proven Roblox pattern; versioned schema + migrations; autosave + `BindToClose`.
- **`WorldSaveService`** — a **world-scoped** store applying the *same* session-locking discipline,
  keyed by **World ID**, storing **chunked, versioned, reference-based** world state; hydrates
  services on boot and serializes them on save.
- **Reserved-server worlds** via `TeleportService:ReserveServer`, World IDs + access codes tracked in
  a directory store.
- Build persistence **first-class and early** (see [roadmap](./09-roadmap.md)) — never retrofit it.

---

**Prev:** [« 07 — Networking](./07-networking.md) &nbsp;|&nbsp;
**Next:** [09 — Roadmap »](./09-roadmap.md)
