# 02 — Gameplay Systems

> Every major system, specified with a fixed template so they stay comparable and reviewable.
> Implementation detail lives in [architecture](./03-technical-architecture.md); this doc is the
> **system design**.

**Each system is documented as:**

- **Purpose** — why it exists.
- **Responsibilities** — what it owns (and, implicitly, what it does *not*).
- **Dependencies** — other systems/data it needs.
- **Scalability concerns** — where it breaks at 1M visits, and the guardrails.
- **Future expansion** — how it grows in later versions.
- **Recommended implementation** — the approach we'll build, with rationale.

> **Cross-cutting principle:** every system below is **server-authoritative** and **data-driven**.
> The client requests; the server validates and decides. Content (resources, recipes, buildings,
> research, disasters) is declared as **Definitions** (data modules) so designers add content
> without engineers touching logic. See [architecture](./03-technical-architecture.md).

### System map & ownership

| # | System | Layer | Owns the truth of… |
|---|--------|-------|--------------------|
| 1 | Resource & Gathering | Server | World nodes, harvest results |
| 2 | Inventory | Server | Player + shared Storage contents |
| 3 | Crafting | Server | Recipe validation, tool/item creation |
| 4 | Building / Construction | Server | Placed structures, placement rules |
| 5 | Farming | Server | Crop growth state |
| 6 | Research / Tech Tree | Server | Server-wide tier + research progress |
| 7 | Disaster | Server | Active disasters, world damage |
| 8 | World Simulation & Time | Server | The tick, time-of-day, scheduling |
| 9 | Progression / Unlocks | Server + Client | What is available at current tier |
| 10 | Player Data | Server | Per-player persistent profile |
| 11 | Social / Cooperation | Server + Client | Shared goals, presence, communication |
| 12 | UI / HUD Framework | Client | Presentation of all of the above |

---

## 1. Resource & Gathering System {#gathering}

**Purpose.** Turn the island into a source of raw materials via the core-loop verb "gather." The
first system players touch and the input to every other economy system.

**Responsibilities.**
- Own **resource nodes** in the world (trees, rocks, ore, bushes, farm yields) — their spawn,
  health/quantity, depletion, and respawn.
- Validate a gather request: player in range, correct tool of sufficient tier, node not depleted,
  cooldown respected.
- Yield resources into the player's inventory and apply node depletion.
- Manage **respawn / regrowth** of nodes over simulation time.

**Dependencies.**
- Inventory (deposit yields), Crafting (tool tier gates gathering), World Simulation (regrowth tick),
- Resource & Node **Definitions** (yield amounts, tool requirements, respawn timers),
- Networking (gather request), Disaster (fire can destroy tree nodes; drought slows regrowth).

**Scalability concerns.**
- **Thousands of nodes × many servers.** Do not run a script per node. Use a single manager
  iterating a data table; represent nodes as lightweight instances tagged via CollectionService.
- **Network spam** — gather is the most-called action. Rate-limit per player; validate range
  server-side; never trust client-reported yields.
- **Physics/render cost** of many node models → instance pooling + StreamingEnabled.

**Future expansion.**
- New resources = new Definition entries (Clay, Copper, Fish, Gold…). No logic change.
- Biomes with resource-affinity; rare/finite deposits; tool enchantments; gathering skill perks.

**Recommended implementation.**
- A single **`GatheringService`** owning a node registry (in-memory table keyed by node id,
  persisted as part of world state). Nodes are data first, instances second (spawned/despawned by
  streaming). Gather = validated RPC → decrement node → grant via `InventoryService`. Regrowth
  driven by `WorldSimulation` tick, not per-node timers. Rationale: one manager is
  cache-friendly, easy to persist, and trivially scalable vs. thousands of independent scripts.

---

## 2. Inventory System {#inventory}

**Purpose.** Track what each player carries **and** the server's shared **Storage** stockpile — the
communal treasury that funds research and building.

**Responsibilities.**
- Personal inventory: stacks, capacity limits, add/remove with validation.
- Shared **Storage** building contents: deposits/withdrawals, per-resource totals.
- Enforce capacity and prevent negative/duplicated counts (anti-dupe).
- Emit change events for UI and for research/building affordability checks.

**Dependencies.**
- Gathering (source of items), Crafting/Building/Research (sinks), Player Data (persist personal
  inventory), World Save (persist shared Storage), Networking, UI.

**Scalability concerns.**
- **Dupe exploits** are the #1 economy threat → all mutations server-side, atomic, transactional
  (deduct-then-grant never client-ordered).
- Replicating full inventory on every change is wasteful → **replicate deltas**, not whole tables.
- Shared Storage is a **contention point** (many players depositing at once) → serialize mutations
  through the service; use a queue/lock discipline.

**Future expansion.**
- Multiple storage tiers/depots; item weight; equipment slots; personal chests in houses; trading
  between players; item quality/durability.

**Recommended implementation.**
- **`InventoryService`** exposing `TryAdd/TryRemove/Transfer` as the *only* mutation path (no system
  edits inventories directly — they call the service). Data shape: `{ [itemId]: count }` maps.
  Personal inventory persists via `PlayerData`; Storage persists via `WorldSave`. Delta replication
  to clients. Rationale: a single chokepoint makes anti-dupe and validation enforceable in one place.

---

## 3. Crafting System {#crafting}

**Purpose.** Convert resources into tools and items via **recipes**, gating the tool progression
that in turn gates gathering.

**Responsibilities.**
- Own **recipe Definitions** (inputs → output, required tier/station).
- Validate a craft: player has inputs, meets tier, is at required station (e.g., Blacksmith for
  iron/steel).
- Consume inputs and grant output atomically.

**Dependencies.**
- Inventory (inputs/outputs), Progression (tier gate), Building (crafting stations like Campfire,
  Blacksmith), Recipe Definitions, Networking, UI.

**Scalability concerns.**
- Validation must be fully server-side (never trust "I crafted X").
- Recipe lookups are hot → recipes are static data, indexed by output id; O(1) lookups.
- Station proximity checks must be cheap (spatial check against known station positions).

**Future expansion.**
- Multi-station crafting; timed crafting queues; blueprints; upgrade/repair recipes; byproducts;
  recipe discovery.

**Recommended implementation.**
- **`CraftingService`** driven entirely by `RecipeDefinitions`. Craft flow: RPC → validate tier +
  station proximity + inputs via `InventoryService` → atomic consume/grant. Rationale: recipes as
  pure data means new tools/items ship as data entries; logic never changes.

---

## 4. Building / Construction System {#building}

**Purpose.** Let players **place structures** that reshape the island and provide function
(Storage, Farm, Blacksmith, etc.). The system most responsible for the world "telling a story."

**Responsibilities.**
- Own **placed structures**: position, rotation, owner, health, build/repair state.
- Validate placement: unlocked at tier, affordable, legal spot (terrain, collision, claim rules).
- Consume resources, spawn the structure, register it for other systems (a Farm registers with
  Farming; a Blacksmith becomes a crafting station).
- Handle **damage/repair** (disasters, decay) via the Hammer.
- Persist all placed structures as part of world state.

**Dependencies.**
- Inventory (build cost), Progression (tier unlock), Building Definitions (cost, model, footprint,
  function), World Save (persist), Disaster (damage), Farming/Crafting (buildings enable them),
  Networking, UI (ghost-preview placement).

**Scalability concerns.**
- **Unbounded placement** → server memory + replication blow-up. Enforce **build limits/zoning**,
  budget per world, and stream structures.
- Placement validation must be authoritative (no clipping/exploit placement).
- Persisting potentially thousands of structures → compact serialization + chunked/versioned saves
  (see [Saving](./08-saving-system.md)).
- Physics cost of many models → anchored parts, streaming, LOD.

**Future expansion.**
- Freeform/modular building (walls, floors, roofs); building upgrades/tiers; roads & decoration;
  claims/permissions; auto-generated village layouts; building health/decay economy.

**Recommended implementation.**
- **`BuildingService`** with a placed-structure registry (data + streamed instances). Placement:
  client shows a **ghost preview** and sends a placement *intent*; server re-validates everything
  and commits. Each Building Definition declares its `function` hooks (e.g., `onPlaced` registers a
  Farm with `FarmingService`). Rationale: definition-driven function hooks keep BuildingService
  generic while letting each building type do special things — highly expandable.

---

## 5. Farming System {#farming}

**Purpose.** Produce **Wheat** over time through tended Farm plots — the game's first "invest now,
reward later" mechanic and the anchor of the food economy.

**Responsibilities.**
- Track crop **growth stages** per farm plot over simulation time.
- Handle planting, tending (water/weed — tunable), and harvesting (Hammer/hand).
- Apply modifiers: drought slows/stops growth; farming research improves yield/resilience.

**Dependencies.**
- Building (Farm structure), World Simulation (growth tick), Inventory (seeds in / wheat out),
  Progression (Farming research), Disaster (drought), Crop Definitions, Networking, UI.

**Scalability concerns.**
- Do **not** tick each plot with its own script/heartbeat. Advance all plots from the central
  simulation tick over a data table.
- Growth is **time-based, not frame-based** — store timestamps and compute stage on read, so
  offline/empty-server time is respected and there's no per-frame cost.

**Future expansion.**
- Multiple crops; seasons; soil quality; irrigation; livestock; fertilizer; greenhouse tiers.

**Recommended implementation.**
- **`FarmingService`** storing per-plot `{ plantedAt, cropId, stageOverrides }`. Growth stage is a
  **pure function of elapsed simulation time** (computed lazily). The tick only checks for
  stage-change events (to update visuals/notify). Rationale: timestamp-based growth is
  cheap, correct across server restarts, and central to persistence integrity.

---

## 6. Research / Tech Tree System {#research}

**Purpose.** The **server-wide progression spine**. Players pool resources to complete research
nodes, advancing the **Civilization Tier** and unlocking content for everyone.

**Responsibilities.**
- Own the **research graph** (nodes, prerequisites, costs) as data.
- Accept resource contributions toward the active node; track progress.
- Complete nodes → advance tier → fire unlock events (tools, buildings, world evolution).
- Persist server research state as part of world state.

**Dependencies.**
- Inventory/Storage (resource contributions), Progression (applies unlocks), World Simulation
  (visual evolution on tier-up), Research Definitions, World Save, Networking, UI (research board).

**Scalability concerns.**
- Shared progress = **concurrent contributions** → serialize through the service; atomic increments.
- Must be **cheap to query** ("what's unlocked?") — cache current tier + unlocked set.
- Anti-exploit: contributions validated against real Storage deductions.

**Future expansion.**
- **Branching tree** (parallel paths, specializations); per-node bonuses; repeatable/"golden age"
  research; multiple research halls; era techs beyond Steel.

**Recommended implementation.**
- **`ResearchService`** holding `currentTier`, `activeNode`, and `progress`. Contribution =
  `InventoryService` withdrawal from Storage → increment progress → on completion, advance tier and
  broadcast `TierChanged`. The graph is `ResearchDefinitions` (already structured for branching even
  though V1 is linear). Rationale: designing the data as a graph now avoids a rewrite when branching
  ships later (a top risk called out in the GDD).

---

## 7. Disaster System {#disaster}

**Purpose.** Periodically **reshape the world** to create tension, adaptation, and emergent stories
— tension without PvP.

**Responsibilities.**
- Schedule disasters (weighted/random within tunable bounds), respecting fairness (telegraphed, not
  spawn-camping new worlds).
- Execute disaster effects authoritatively: Storm (structure damage, work halt), Forest Fire
  (spreads across tree nodes/wooden buildings), Drought (farming/berry/fire-risk modifiers).
- Telegraph incoming disasters to clients (warning UI, sky/audio cues).
- Apply and persist resulting world damage.

**Dependencies.**
- World Simulation (scheduling on the tick), Building (damage), Gathering (destroy/slow nodes),
  Farming (drought), Disaster Definitions (effects, weights, durations), World Save, Networking, UI.

**Scalability concerns.**
- Fire **spread** can be O(n²) if naive → grid/spatial spread with capped propagation per tick.
- Mass structure damage → batch updates, replicate summaries not per-part.
- Must be deterministic-enough to persist mid-disaster state or safely resolve on save.

**Future expansion.**
- New disasters (earthquake, flood, blight, meteor, harsh winter); disaster *chains*; defensive
  tech/buildings; "rebuild bonuses"; seasonal disaster profiles.

**Recommended implementation.**
- **`DisasterService`** with a scheduler fed by the simulation tick and `DisasterDefinitions`
  (weight, cooldown, duration, effect handlers). Each disaster is a data entry + an effect handler
  module → adding a disaster = adding data + one handler. Spread uses a bounded spatial algorithm.
  Rationale: handler-per-disaster keeps the scheduler generic and the roster infinitely extensible.

---

## 8. World Simulation & Time System {#simulation}

**Purpose.** The **heartbeat** of the world: advances time-of-day, drives all time-based systems
(regrowth, crop growth, disaster scheduling), and coordinates world evolution visuals.

**Responsibilities.**
- Maintain authoritative **world clock** (day/night, world day count, elapsed sim time).
- Run a **fixed-timestep tick** that other services subscribe to (regrowth, farming, disasters).
- Drive **visual evolution** on tier-up (swap/augment world dressing per tier).
- Provide the single time source used for all timestamp math and persistence.

**Dependencies.**
- Nearly every server system subscribes to it. Depends on Networking (replicate time-of-day),
  World Save (persist clock), Research (tier-up visuals).

**Scalability concerns.**
- A single well-designed tick is far cheaper than many independent loops → **one scheduler,
  fan-out to subscribers**.
- Avoid doing heavy work every tick; **stagger** subsystem updates across ticks (time-slicing).
- Time math must survive server restarts (store absolute timestamps, not frame counts).

**Future expansion.**
- Seasons/weather cycles; calendar events; festivals; day-length tuning; catch-up simulation for
  world reload (simulate elapsed offline time on load).

**Recommended implementation.**
- **`WorldSimulation`** owning the clock and a subscriber registry with fixed-step accumulation
  (decoupled from render/`Heartbeat` frame rate). Subsystems register update callbacks with a
  cadence. Rationale: one authoritative, testable time source + time-slicing is the standard
  pattern for scalable simulation and keeps persistence math correct.

---

## 9. Progression / Unlocks System {#progression}

**Purpose.** The **gatekeeper** that answers "is this content available right now?" based on the
server's Civilization Tier (and later, per-player progression).

**Responsibilities.**
- Maintain the **unlocked set** (tools, buildings, recipes, resources) for the current tier.
- Provide fast `IsUnlocked(id)` checks to Crafting, Building, Gathering, UI.
- React to `TierChanged` and update the unlocked set + notify clients.

**Dependencies.**
- Research (source of tier changes), all content Definitions (declare their unlock tier),
  Networking (replicate unlocked set), UI (grey-out locked content).

**Scalability concerns.**
- Must be **O(1) to query** — precompute the unlocked set on tier change, don't recompute per check.
- Replicate a compact unlocked-set snapshot to clients, not the full definition tables.

**Future expansion.**
- Per-player skills/perks layered on server tier; prestige; achievement-gated unlocks; region-gated
  content.

**Recommended implementation.**
- **`ProgressionService`** holding a cached `Set<unlockedId>` derived from tier + Definitions,
  refreshed on `TierChanged`. Every other system asks it rather than re-deriving. Rationale: single
  cached authority avoids scattered tier-checks and keeps gating consistent.

---

## 10. Player Data System {#playerdata}

**Purpose.** Own each player's **persistent profile** — the per-player half of persistence (the
world half lives in [Saving](./08-saving-system.md)).

**Responsibilities.**
- Load a player's profile on join (session-locked), hold it in memory, save on leave/shutdown.
- Store: personal inventory, equipped tools, personal stats/cosmetics, per-world personal state
  (e.g., their claimed house), settings.
- Guarantee no data loss / no duplication across servers (session locking).

**Dependencies.**
- DataStore layer (`PlayerData` store), Inventory (personal contents), Building (owned structures),
  Networking (send profile view to client), World Save (some player state is world-scoped).

**Scalability concerns.**
- **Session locking is mandatory** to prevent item loss/dupe across servers (the classic Roblox
  data trap). Use a ProfileStore-style pattern.
- DataStore request budgets → batch, debounce, and autosave on interval + `BindToClose`.
- Migration: profiles **will** change shape over years → versioned schema + migration on load.

**Future expansion.**
- Cross-world player identity; account-level cosmetics/inventory; friends/parties; cloud settings;
  cross-server memory (MemoryStore) for presence.

**Recommended implementation.**
- **`PlayerDataService`** wrapping a session-locked store (ProfileStore/ProfileService pattern or an
  in-house equivalent). Load-on-join, autosave, save-on-leave, `BindToClose` flush; **versioned
  schema with migration functions**. Rationale: session-locking + versioning is non-negotiable for a
  commercial economy game; see [Saving](./08-saving-system.md) for full detail.

---

## 11. Social / Cooperation System {#social}

**Purpose.** Make ORIGINS **feel cooperative** — surface shared goals and shared presence so players
naturally work together. Cooperation is a core pillar, so it gets a real system, not an afterthought.

**Responsibilities.**
- Surface the **server goal** (current research node + resource shortfall) prominently.
- Show **presence/activity** (who's gathering what, recent contributions, recent builds).
- Contribution feedback ("You added 40 Wood toward Iron Working") and server milestones.
- Lightweight communication scaffolding (pings/markers, emotes; text via Roblox chat).

**Dependencies.**
- Research (server goal), Inventory/Storage (contributions), Building (recent builds), UI (the
  social/HUD surface), Networking (broadcast events).

**Scalability concerns.**
- Broadcasting every micro-action = spam → aggregate and throttle (summaries, feeds, not per-event).
- Keep social state derived/ephemeral where possible to minimize persistence cost.

**Future expansion.**
- Roles/professions; shared objectives & server quests; leaderboards of contribution; guild/party
  systems; world chronicle/history log ("the world tells its story" made literal); map markers/pings.

**Recommended implementation.**
- A **`SocialService`** (server) aggregating events into throttled feeds + a **`SocialController`**
  (client) rendering the goal banner, contribution feed, and presence. Rationale: centralizing "what
  the server is doing" is the mechanism that converts parallel solo play into felt cooperation — the
  cheapest, highest-leverage retention lever.

---

## 12. UI / HUD Framework {#ui}

**Purpose.** Present every system cleanly and performantly. "Clean UI" is an explicit pillar, so UI
is treated as a **framework**, not per-screen one-offs.

**Responsibilities.**
- HUD (resources, tool, server goal, time-of-day, disaster warnings).
- Interaction UIs (crafting, building placement, research board, storage, inventory).
- A consistent **component system**, theming, and a **reactive state → view** binding so UI updates
  from replicated state without spaghetti.
- Input abstraction (mouse/keyboard, touch, gamepad) — Roblox is cross-platform.

**Dependencies.**
- Networking (replicated state feeds UI), every client Controller, a UI library/state layer.

**Scalability concerns.**
- UI churn causes frame drops → update from **deltas**, avoid rebuilding trees, pool frames.
- Cross-platform: must scale to phone screens (huge share of Roblox) from day one.
- Many screens → without a component framework this becomes unmaintainable fast.

**Future expansion.**
- Theming/skins (monetization); accessibility (text scale, colorblind); localization; minimap;
  building catalog; codex/tutorial system.

**Recommended implementation.**
- A **reactive UI approach** — evaluate **Fusion** (or Roact/React-Lua) for declarative,
  state-driven components — over imperative hand-managed `ScreenGui`s. Central client **UI store**
  holds replicated state; components subscribe. Rationale: declarative + reactive UI is dramatically
  more maintainable across dozens of screens and 5 years than imperative UI code, and updates
  efficiently from deltas. Final library choice is recorded in
  [tooling](./10-tooling-and-conventions.md).

---

## Anti-exploit posture (applies to all systems)

Because ORIGINS targets millions of visits, every system assumes a hostile client:
- **Server decides, client requests.** No system trusts client-reported outcomes.
- **Validate everything** at the service boundary (range, tier, ownership, affordability, cooldown).
- **Atomic economy mutations** only through `InventoryService` (anti-dupe chokepoint).
- **Rate-limit** hot RPCs (gather/craft/place) centrally in the [networking layer](./07-networking.md).

---

**Prev:** [« 01 — Core Loop](./01-core-gameplay-loop.md) &nbsp;|&nbsp;
**Next:** [03 — Technical Architecture »](./03-technical-architecture.md)
