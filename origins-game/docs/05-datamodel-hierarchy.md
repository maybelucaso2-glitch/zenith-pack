# 05 — Roblox DataModel Hierarchy

> Where things live **inside the running Roblox game** (the DataModel), and why each service is the
> right home. This is the runtime counterpart to the on-disk [folder structure](./04-folder-structure.md).

---

## 1. The tree

```
game (DataModel)
│
├── ReplicatedStorage
│   ├── Shared/                 ← src/shared (Definitions, Net, Types, Util, Framework)
│   │   ├── Definitions/        ← content DATA (read by client + server)
│   │   ├── Net/                ← remote registry + typed wrapper
│   │   ├── Types/
│   │   ├── Util/
│   │   └── Framework/
│   ├── Packages/               ← Wally dependencies (shared)
│   └── Remotes/                ← RemoteEvent/Function instances (created from Net registry)
│
├── ReplicatedFirst
│   └── Loading                 ← minimal loading screen, runs first
│
├── ServerScriptService
│   └── Server/                 ← src/server (Services + bootstrap) — NEVER replicated to clients
│       ├── init.server
│       └── Services/
│
├── ServerStorage
│   ├── Assets/                 ← server-only models/templates (spawned into Workspace)
│   └── PrivateData/            ← server-only data modules (never sent to clients)
│
├── StarterPlayer
│   └── StarterPlayerScripts
│       └── Client/             ← src/client (Controllers + UI) — copied into each player
│
├── StarterGui                  ← (optional) static GUI roots; most UI is built by Controllers
│
├── Workspace
│   ├── World/                  ← the island, spawned/managed by the SERVER at runtime
│   │   ├── Terrain
│   │   ├── ResourceNodes/      ← streamed instances backed by GatheringService data
│   │   └── Structures/         ← streamed instances backed by BuildingService data
│   └── (StreamingEnabled = true)
│
├── Lighting                    ← driven by WorldSimulation (time-of-day) + Disaster (storm/fire)
├── SoundService
├── Teams                       ← (optional) not core to V1 co-op
└── Players
```

## 2. Service-by-service rationale

### ReplicatedStorage — the shared library
- **Why:** replicated to **both** server and clients, but **not executed** automatically. Perfect
  home for shared **modules** (Definitions, Net contract, Types, Util) that both realms `require`.
- **`Shared/Definitions`** must be here so the client can display/grey-out content using the *same*
  data the server validates against — one source of truth, no drift.
- **`Remotes/`** holds the actual `RemoteEvent`/`RemoteFunction` instances, created at boot from the
  `Net` registry so there's a single declared list.
- ⚠️ **Never** put secret logic or unreleased-content secrets here — clients can read everything in
  ReplicatedStorage. Sensitive data goes in **ServerStorage**.

### ReplicatedFirst — the loading screen
- **Why:** the first container replicated to clients, before the rest of the game streams in. Only a
  tiny, dependency-free loading UI belongs here so players see something instantly.

### ServerScriptService — the brain
- **Why:** server-only, executes on the server, **never replicated to clients**. All **Services**
  (authoritative logic + state) live here. This is where "server decides" physically lives.

### ServerStorage — server-only assets & secrets
- **Why:** server-only, **not executed and not replicated**. Ideal for **model templates** the
  server clones into `Workspace` (structures, node models) and any **private data** that must never
  reach a client (e.g., unreleased content, anti-exploit config, drop tables).
- Keeping asset templates here (not in ReplicatedStorage) reduces what clients download and hides
  content until unlocked.

### StarterPlayer/StarterPlayerScripts — the client
- **Why:** contents are copied into every player on join and run locally. All **Controllers** and
  **UI** live here. Purely presentation/input — no authority.

### Workspace — the world (server-owned)
- **Why:** the visible 3D world. Critically, **the island is spawned and managed by the server at
  runtime**, not baked as static geometry, so it can be data-driven, streamed, and persisted.
- **StreamingEnabled = true** so massive evolving worlds stay performant on phones; the server
  controls what each client streams.
- `ResourceNodes/` and `Structures/` are **instances backing data** owned by GatheringService /
  BuildingService — the data is the truth, instances are a view.

### Lighting / SoundService — atmosphere driven by systems
- **Why:** `WorldSimulation` drives day/night via Lighting; `DisasterService` overrides it during
  storms/fires. Centralizing atmosphere here keeps effects consistent and cheap.

### Players — runtime, not authored
- Player objects appear at runtime; `PlayerDataService` attaches each player's profile on join.

## 3. What deliberately does **not** exist as static instances

- **The island geometry** is not a static Workspace build — it's spawned/managed by the server so it
  can evolve, stream, and persist. (A base terrain may be pre-authored, but structures/nodes are
  runtime data.)
- **Most GUIs** are not static `StarterGui` trees — they're built by the reactive UI framework from
  replicated state, so they update from deltas and stay maintainable.

## 4. Security summary (which containers clients can see)

| Container | Client can read code/data? | Use for |
|-----------|---------------------------|---------|
| ReplicatedStorage | ✅ yes | Shared modules, Definitions, Net, public assets |
| ReplicatedFirst | ✅ yes | Loading screen |
| StarterPlayerScripts | ✅ yes (it's their code) | Controllers, UI |
| Workspace | ✅ yes (what's streamed) | The world |
| **ServerScriptService** | ❌ no | **Authoritative Services** |
| **ServerStorage** | ❌ no | **Server assets, secrets, private data** |

**Rule of thumb:** if a client seeing it would enable an exploit or spoil unreleased content, it goes
in **ServerScriptService** or **ServerStorage** — never ReplicatedStorage.

---

**Prev:** [« 04 — Folder Structure](./04-folder-structure.md) &nbsp;|&nbsp;
**Next:** [06 — Module Architecture »](./06-module-architecture.md)
