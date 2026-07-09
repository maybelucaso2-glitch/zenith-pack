# 09 — Roadmap

> The phased build plan (how we get V1 shipped in the right order) and the future-update vision (how
> ORIGINS grows for years). Sequencing is deliberate: **foundations before features, feel before
> content, persistence early.**

---

## Part A — Build phases to V1

Each phase ends in a **playable, testable milestone**. We don't build systems in isolation and
integrate at the end; we always have something that runs.

### Phase 0 — Foundations _(engineering only, no gameplay)_
- Repo, Rojo, Wally, Selene, StyLua, CI ([tooling](./10-tooling-and-conventions.md)).
- In-house **framework** (service/controller loader, `Init`/`Start` lifecycle).
- **Net** layer (registry, middleware, rate limiting) — even before features need it.
- **Types** + first **Definitions** skeletons.
- **Milestone:** empty game boots, framework runs, a test remote round-trips with validation.

### Phase 1 — Core loop vertical slice _(the "is it fun?" gate)_
- `WorldSimulation` (tick + clock), `GatheringService`, `InventoryService`.
- Berries + Wood + Stone; Stone Axe/Pickaxe; personal inventory + shared **Storage**.
- Minimal HUD (resources, tool). Server-spawned test island.
- **Milestone:** a player can gather, store, and feel the core loop. **Fun is validated here** before
  investing in the rest.

### Phase 2 — Persistence _(early, not late)_
- `PlayerDataService` (session-locked, versioned) + `WorldSaveService` (chunked, versioned).
- Reserved-server world flow + World directory.
- **Milestone:** leave and rejoin a world; everything gathered/stored/built is exactly as left.
  Persistence proven before content piles up (retrofitting saves is the classic disaster).

### Phase 3 — Building & Progression
- `BuildingService` (ghost preview → validated placement), `ProgressionService`.
- Campfire, Storage, Research Hall; build/repair with Hammer.
- **Milestone:** players reshape the island; structures persist and stream.

### Phase 4 — Research & the meta loop
- `ResearchService` (graph data, contributions, tier-up), world visual evolution on tier change.
- Full research spine: Fire → Stone Tools → Farming → Iron Working → Construction → Steel.
- **Milestone:** the server can advance civilization tiers together — the retention engine turns.

### Phase 5 — Economy depth: Farming, Iron, full toolset
- `FarmingService` (timestamp growth), Farm/Mine/Blacksmith, Iron + Wheat.
- Iron Axe/Pickaxe, Torch; full crafting via stations.
- **Milestone:** complete V1 resource/tool/building economy is live and balanced.

### Phase 6 — Disasters & Social
- `DisasterService` (Storm, Forest Fire, Drought) with telegraphs; `SocialService` goal banner +
  contribution feed.
- **Milestone:** the world reshapes itself; cooperation is *felt*, not just possible.

### Phase 7 — Polish, UI pass, performance, launch
- Reactive UI pass across all screens; onboarding/tutorial; audio/atmosphere.
- Performance hardening (streaming, pooling, delta tuning); anti-exploit review.
- Telemetry/metrics; balance pass; soft launch → measure retention.
- **Milestone:** **V1 ships.**

> **Sequencing rationale:** (1) prove the core loop is fun before building content on top of it;
> (2) land persistence *before* there's lots of state to migrate; (3) turn on the meta loop
> (research/tiers) as early as it's coherent, because it's the retention engine.

## Part B — Post-V1 update roadmap (the multi-year vision)

Designed so **none of these require rewrites** — each is data + a service/handler, thanks to the
data-driven architecture.

### Content expansions (mostly Definitions)
- **New resources:** Clay, Copper, Gold, Fish, Wool, Herbs…
- **New tools & tiers:** fishing rod, hoe, tool upgrades/durability, enchant-style perks.
- **New buildings:** Well, Market, Dock, Bakery, Barracks, Temple, Roads & decoration.
- **New disasters:** Earthquake, Flood, Blight, Harsh Winter, Meteor; disaster *chains* & seasons.
- **New civilization tiers** beyond Steelbound (Medieval, Industrial… era arcs).

### System expansions
- **Branching research tree** (the graph is already branch-ready) — specializations & parallel paths.
- **Freeform/modular building** (walls/floors/roofs), building upgrades, claims/permissions.
- **Farming depth:** crops, seasons, soil, irrigation, **livestock**.
- **Offline-progress simulation** on world reload.
- **World chronicle / history log** — literally makes "the world tells its story" a feature.
- **Roles/professions & server quests** — deepen cooperation.
- **Biomes & world generation** variety for replayability.

### Platform & meta
- **Cross-server features** (MemoryStore/MessagingService): world directory, global events, presence.
- **World backups/snapshots & rollback.**
- **Cosmetic economy** (building/tool skins, emotes) — the non-P2W monetization.
- **Localization & accessibility** passes.
- **Creator/community tools** long-term (custom worlds/mods within safe bounds).

## Part C — What we explicitly defer (and why)

| Deferred | Why not in V1 |
|----------|---------------|
| PvP / raiding | Not the fantasy; co-op-first. Could be an opt-in mode far later |
| Trading/economy marketplace | Needs a stable base economy first |
| Modding/UGC content | Requires a hardened, sandboxed content pipeline |
| External backend DB | DataStore suffices for V1; adds ops burden |
| Freeform building | Big scope; V1 uses placed structures to prove the loop first |

## Definition of Done for V1

- Core loop is fun and validated (Phase 1 gate passed).
- A persistent world survives leave/rejoin/restart with zero data loss (session-locked, versioned).
- The full Tier 0→6 research spine is completable cooperatively, with visible world evolution.
- All V1 resources/tools/buildings/disasters function and are balanced.
- Runs smoothly on mobile with StreamingEnabled; passes an anti-exploit review.
- Clean, reactive UI + onboarding; telemetry in place to measure retention.

---

**Prev:** [« 08 — Saving](./08-saving-system.md) &nbsp;|&nbsp;
**Next:** [10 — Tooling & Conventions »](./10-tooling-and-conventions.md)
