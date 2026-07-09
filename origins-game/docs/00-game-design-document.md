# 00 — Game Design Document

> The single source of truth for **what** ORIGINS is. Other docs describe **how** it is built.

---

## 1. Vision statement

**ORIGINS is a cooperative civilization-building game where a server of players transforms an
untouched island into a living civilization over days, weeks, and months.**

It is not a survival game with a health bar and a night timer. Survival pressure exists (disasters,
scarcity), but it is the *backdrop*, not the point. The point is **collective progress made
visible** — the moment the server unlocks Iron Working and the first blacksmith's forge lights up,
everyone on the island should feel it.

The design north star: **"The world should tell the story of its players."** A stranger who joins a
mature world should be able to *read* its history — which forest was cleared first, where the
drought struck, which player over-built the eastern ridge.

## 2. Genre & positioning

| Influence | What we take from it |
|-----------|----------------------|
| Minecraft | Systemic gathering/crafting/building; sandbox freedom |
| Civilization | Shared tech tree; era-based visual evolution; long arcs |
| Rust | Persistent shared world, resource scarcity, base-building — **minus** the PvP-first loop |
| Animal Crossing | Cozy, unhurried, daily-return progression; charm over grind |
| Roblox Islands | Approachable cooperative economy on Roblox |
| The Survival Game | Roblox-native gathering feel and pacing |

**Positioning line:** _"A civilization you build together, one island at a time."_

## 3. Target audience

- **Primary:** Roblox players 13+ who enjoy building, progression, and social play (Islands,
  Lumber Tycoon, Bloxburg, Sol's/creative-cooperative crowd).
- **Secondary:** Sandbox/strategy fans who want a lighter, social civ experience.
- **Session shape:** designed for both short drop-ins (20–40 min gathering runs) and long
  co-op sessions (2+ hours of coordinated building).

## 4. Core fantasy & player promises

- **"We built this."** Every structure on the island was placed by a real player.
- **"Our world is unique."** No two servers evolve the same way.
- **"Progress is shared."** Research and civilization tier are server-wide achievements.
- **"It remembers us."** The world persists; returning tomorrow, it is exactly as you left it,
  plus whatever your teammates did overnight.

## 5. Design philosophy (explicit non-goals)

Prioritize: long-term retention · high replayability · strong social gameplay · emergent stories ·
clean UI · high performance · expandability · professional architecture.

**Explicitly avoid:**
- ❌ Shallow simulator mechanics (auto-incrementing counters, "rebirth" loops).
- ❌ Clicker gameplay (mindless tap-to-win).
- ❌ Pay-to-win (paid power that trivializes shared progress).
- ❌ Disposable content that requires full rewrites to extend.

Everything should feel **handcrafted and polished**.

## 6. The civilization tiers (the spine of progression)

Progression is expressed as **Civilization Tiers** — a server-wide state that gates content and
changes the island's look. V1 ships the first arc; later tiers are in the [roadmap](./09-roadmap.md).

| Tier | Name | Unlocked by | The world visibly… |
|------|------|-------------|--------------------|
| 0 | **Landing** | (start) | Untouched island, no player structures |
| 1 | **Firelight** | Research: Fire | Campfires; cleared undergrowth; first paths |
| 2 | **Stone Age** | Research: Stone Tools | Stone tools; quarried rock; sturdier huts |
| 3 | **Harvest** | Research: Farming | Tilled fields; farms; food surplus |
| 4 | **Ironforge** | Research: Iron Working | Mines lit; blacksmith smoke; metal tools |
| 5 | **Settlement** | Research: Construction | Proper houses; planned layouts; storage depots |
| 6 | **Steelbound** | Research: Steel | Steel structures; the civilization "arrives" |

> Tiers are **data-driven** (see [systems](./02-gameplay-systems.md) and
> [architecture](./03-technical-architecture.md)) so new tiers are added without touching gameplay
> code.

## 7. Version 1 scope (locked)

V1 is intentionally **small and complete** rather than large and shallow. It must fully deliver the
core loop and one satisfying civilization arc (Tiers 0→6).

### 7.1 Resources

| Resource | Source | Gathered with | Primary uses |
|----------|--------|---------------|--------------|
| **Wood** | Trees | Axe | Fire, tools, buildings |
| **Stone** | Rocks / quarry | Pickaxe | Stone tools, buildings, mine |
| **Iron** | Iron ore (mine) | Iron Pickaxe | Iron tools, blacksmith, steel |
| **Wheat** | Farm plots | Hammer/hand (harvest) | Food, farming research |
| **Berries** | Bushes (forage) | Hand | Early food, no-tool entry resource |

Design note: **Berries** are the zero-tool bootstrap resource so a fresh player is never stuck.
**Wheat** requires the Farm building → creates a reason to build and to research Farming.

### 7.2 Buildings

| Building | Unlock tier | Purpose |
|----------|-------------|---------|
| **Campfire** | 1 Firelight | Cook/consume; social hub; light; enables Fire-era crafting |
| **Storage** | 1 | Shared server stockpile (see [Inventory](./02-gameplay-systems.md#inventory)) |
| **Small House** | 5 Settlement | Personal/claimed space; spawn point; cosmetic identity |
| **Farm** | 3 Harvest | Produces Wheat over time; requires tending |
| **Blacksmith** | 4 Ironforge | Smelt iron; craft iron/steel tools |
| **Mine** | 4 | Unlocks/accelerates Iron & deep Stone gathering |
| **Research Hall** | 2 Stone Age | Where the server pools research toward the next tier |

### 7.3 Tools

| Tool | Tier | Gathers / does |
|------|------|----------------|
| **Stone Axe** | 2 | Wood (basic) |
| **Stone Pickaxe** | 2 | Stone (basic) |
| **Iron Axe** | 4 | Wood (fast) |
| **Iron Pickaxe** | 4 | Stone + **Iron** (required for iron) |
| **Hammer** | 5 | Build/repair structures; harvest wheat |
| **Torch** | 1 | Light; required in Mine; disaster utility |

### 7.4 Research tree (server-wide)

```
Fire → Stone Tools → Farming → Iron Working → Construction → Steel
```

Linear in V1 (a clear, legible spine). The tree **structure supports branching** for later
versions (see [roadmap](./09-roadmap.md)); V1 simply ships one path. Each node has a **cost in
pooled resources** contributed by all players at the Research Hall.

### 7.5 Disasters

| Disaster | Trigger | Effect | Counterplay |
|----------|---------|--------|-------------|
| **Storm** | Scheduled/weighted | Damages exposed structures; halts outdoor work; lightning fire risk | Build sturdier (Construction); shelter; repair with Hammer |
| **Forest Fire** | Weighted, higher after drought | Spreads across trees; can reach wooden buildings | Firebreaks (cleared land); water; stone buildings resist |
| **Drought** | Scheduled/weighted | Farms slow/stop; berries scarce; raises fire risk | Food stockpiles; Storage buffering; farming research reduces impact |

Disasters are **the world reshaping itself** and the primary source of emergent stories. They are
**data-driven and server-authoritative** (see [Disaster System](./02-gameplay-systems.md#disaster)).

## 8. Monetization philosophy (non-pay-to-win)

Detailed later, but the **design constraint is set now** so systems are built to respect it:

- ✅ **Cosmetics** — building skins, tool skins, player emotes, world decorations.
- ✅ **Convenience that doesn't trivialize shared progress** — extra personal inventory tabs,
  cosmetic name colors, additional private worlds.
- ✅ **World/server passes** — owning a persistent private world (aligns with the persistent-world
  architecture; see [Saving](./08-saving-system.md)).
- ❌ **No paid resources, no paid research, no paid power.** Server-wide progress must always be
  *earned by the server*, or the core fantasy collapses.

## 9. Success metrics (what "working" means)

- **D1 / D7 / D30 retention** — the true test of a civilization game is the return visit.
- **Median world age** — how many real days worlds survive and keep growing.
- **Session length** and **co-op session ratio** (players building together vs. solo).
- **Tier completion funnel** — % of worlds reaching each civilization tier.
- **Story moments** — disasters survived, structures rebuilt (qualitative + telemetry).

## 10. Risks & senior-lead flags

| Risk | Why it matters | Mitigation (designed-in) |
|------|----------------|--------------------------|
| **Ephemeral servers kill the core fantasy** | Public Roblox servers reset when empty; "build over weeks" is impossible on them | **Persistent private/reserved-server worlds** — the defining architectural decision. See [Saving](./08-saving-system.md). |
| **Grind masquerading as depth** | Civ games rot into resource-counter simulators | Progress gated by *research + building*, not raw grind; disasters force adaptation |
| **Empty-world problem** | Co-op games feel dead when solo | Solo must be viable but slower; social features + world-hosting funnel players together |
| **Content extensibility** | Adding a resource shouldn't mean a rewrite | **Data-driven definitions** everywhere (see architecture) |
| **Exploits at scale** | Millions of visits = constant attack | **Server-authoritative** everything; centralized validated networking |

---

**Next:** [01 — Core Gameplay Loop »](./01-core-gameplay-loop.md)
