# 01 — Core Gameplay Loop

> How ORIGINS actually plays, at three timescales. A game's loops are its skeleton; every system in
> [doc 02](./02-gameplay-systems.md) exists to serve one of these loops.

We design loops at **three nested timescales**. Each must be independently satisfying, and each must
feed the next larger one.

```
┌──────────────────────────────────────────────────────────────────┐
│  META LOOP  (days → weeks → months)  — "advance the civilization"  │
│  ┌──────────────────────────────────────────────────────────┐    │
│  │  SESSION LOOP  (20 min → hours)  — "make the world better"  │    │
│  │  ┌────────────────────────────────────────────────────┐   │    │
│  │  │  CORE LOOP  (seconds → minutes) — "gather & act"     │   │    │
│  │  └────────────────────────────────────────────────────┘   │    │
│  └──────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

---

## 1. Core loop — seconds to minutes

The moment-to-moment verb loop. This is what a player's hands are doing.

```
        ┌──────────►  EXPLORE  ──────────┐
        │        (find nodes/biomes)     ▼
   ACT / BUILD                        GATHER
 (place, craft,                    (chop, mine,
  research, fight                   forage, harvest)
  disaster)                            │
        ▲                              ▼
        └────────────  STORE  ◄─────────┘
                 (deposit to Storage,
                  update inventory)
```

**One turn of the loop:**
1. **Explore** — move through the island; spot a resource node, biome, or need.
2. **Gather** — use the correct tool on a node → resource enters personal inventory.
   (Tool tier gates what you *can* gather: no Iron without an Iron Pickaxe.)
3. **Store / carry** — deposit into shared **Storage**, or keep for immediate crafting.
4. **Act** — spend resources: craft a tool, place/repair a building, contribute to research,
   respond to a disaster.
5. Loop tightens as tools improve (Iron Axe makes the gather step faster and more rewarding).

**Feel targets:** responsive tool swings; satisfying node depletion + respawn; clear numeric
feedback ("+3 Wood"); no dead time waiting.

## 2. Session loop — 20 minutes to a few hours

What a player accomplishes in one sitting. A session should always end with a **visible mark on the
world** — the anti-"wasted my time" guarantee.

```
   Arrive ──► Check world state ──► Pick a goal ──► Core loop ×N ──► Visible result ──► Leave
              (what changed while    (personal or                    (a building rose,
               I was gone? tier?      server-suggested)               research advanced,
               new research? damage?)                                 disaster survived)
```

**Session goals a player might pick (or the game suggests):**
- Fill Storage toward the next research node's cost.
- Craft the tool that unlocks the next resource.
- Raise a specific building the server needs.
- Repair/rebuild after a disaster.
- Prospect an unexplored part of the island.

**Design rule:** the game always surfaces a *"what's the server working toward"* prompt (the current
research goal + its resource shortfall). This turns idle players into contributors and makes co-op
legible.

## 3. Meta loop — days to months (the retention engine)

The reason ORIGINS is a *civilization* game and not a survival game. This loop belongs to the
**server/world**, not the individual.

```
  Tier N world ──► server pools research ──► TIER UP ──► world visibly evolves ──►
       ▲                                                          │
       │                                                          ▼
   disasters test                                        new resources/tools/
   & reshape the world  ◄──────────────────────────────  buildings unlock
```

**One turn of the meta loop:**
1. The server is at **Civilization Tier N**.
2. Players pool resources at the **Research Hall** toward the next node.
3. Research completes → **Tier up** → the island's appearance changes + new content unlocks.
4. New tools/buildings deepen the core loop (better gathering, new goals).
5. **Disasters** periodically stress the world, forcing rebuilding and adaptation — generating
   stories and resetting some goals.
6. Repeat toward **Steelbound** (Tier 6) and beyond (future tiers).

**Why this drives retention:** progress is *shared and persistent*. You return because (a) your
teammates advanced things overnight, (b) you're personally invested in a structure you built, and
(c) the world is uniquely yours — it has a history no other server has.

## 4. How the loops interlock (the key design insight)

| Loop | Owner | Reward | Failure to nail it means… |
|------|-------|--------|---------------------------|
| Core | The player's hands | Immediate (resources, feedback) | Game feels like a chore |
| Session | The player's visit | Medium (a mark on the world) | "I wasted my time" churn |
| Meta | The server/world | Long (civilization evolves) | No reason to return; it's just a sandbox |

A civilization game **lives or dies on the meta loop**, but you only *reach* the meta loop if the
core loop feels good enough to repeat hundreds of times. So V1 build priority is: **core loop feel
first, then the tier/meta payoff.** (Reflected in the [roadmap](./09-roadmap.md).)

## 5. New-player onboarding path (first 10 minutes)

Critical and often neglected. The first session must reach a reward fast.

```
Spawn on beach ──► Forage Berries (no tool, instant success) ──► guided to Campfire/Storage ──►
chop first tree (given/craft Stone Axe) ──► deposit Wood ──► see the server research bar move ──►
"you contributed" moment ──► free to explore
```

Design intent: within ~10 minutes a new player has (1) gathered, (2) used a tool, (3) contributed to
*shared* progress, and (4) understood the meta goal. **Berries-first** guarantees no early dead-end.

## 6. Failure & pressure (what creates tension without PvP)

ORIGINS has no PvP focus, so tension comes from **the world, not other players**:
- **Scarcity** — the right resource is not always nearby; tools gate access.
- **Disasters** — periodic, telegraphed, reshaping. Preparation is rewarded.
- **Decay/maintenance** (tunable, later tiers) — structures need upkeep/repair, giving veterans a
  role and preventing "finished" worlds from feeling static.

Pressure is **cooperative** — the server survives together, which strengthens the social core.

---

**Prev:** [« 00 — GDD](./00-game-design-document.md) &nbsp;|&nbsp;
**Next:** [02 — Gameplay Systems »](./02-gameplay-systems.md)
