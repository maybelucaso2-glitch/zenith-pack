# 04 — Folder Structure

> How the project is laid out on disk (Git) and how Rojo maps it into the Roblox DataModel. The repo
> is the source of truth; Studio is a viewport.

---

## 1. Repository root

```
origins-game/
├── README.md                     # project overview
├── docs/                         # ← all design documentation (this folder)
├── src/                          # ← all Luau source (Rojo-managed)
│   ├── server/                   # → ServerScriptService
│   ├── client/                   # → StarterPlayer/StarterPlayerScripts
│   ├── shared/                   # → ReplicatedStorage/Shared
│   └── first/                    # → ReplicatedFirst (loading screen)
├── assets/                       # source art notes / model manifests (not binary-heavy)
├── tests/                        # test specs (or co-located *.spec.luau under src/)
├── scripts/                      # dev/CI helper scripts
├── default.project.json          # Rojo project mapping (src → DataModel)
├── wally.toml                    # package manifest (dependencies)
├── wally.lock
├── selene.toml                   # linter config
├── stylua.toml                   # formatter config
├── .github/workflows/ci.yml      # lint + test on PR
└── .gitignore
```

**Why `src/` split by realm (server/client/shared/first):** it mirrors the security boundary
directly. Server code is *physically* separated from client code, so it can never accidentally ship
to clients — the folder is the boundary, enforced by Rojo mapping, not convention.

## 2. `src/shared/` — code both realms use

```
src/shared/
├── Definitions/                  # DATA — the content of the game
│   ├── Resources.luau
│   ├── Recipes.luau
│   ├── Buildings.luau
│   ├── Research.luau             # graph-structured (branch-ready)
│   ├── Disasters.luau
│   ├── Tools.luau
│   └── Tiers.luau
├── Net/                          # networking contract (event/function registry + types)
│   ├── Remotes.luau              # declared remotes (single registry)
│   └── Net.luau                  # typed wrapper (rate limit + validation middleware)
├── Types/                        # shared Luau type definitions
│   └── init.luau
├── Constants/                    # tunables not tied to a single content entry
├── Util/                         # pure helpers (math, tables, guards, spatial)
└── Framework/                    # the in-house service/controller loader (shared core)
```

**Definitions is the heart of expandability.** New content lives here as data. Because it's in
`shared`, client (display/greying-out) and server (validation) read identical data — no drift.

## 3. `src/server/` — Services

```
src/server/
├── init.server.luau              # bootstrap: load framework, start services
└── Services/
    ├── GatheringService.luau
    ├── InventoryService.luau
    ├── CraftingService.luau
    ├── BuildingService.luau
    ├── FarmingService.luau
    ├── ResearchService.luau
    ├── DisasterService.luau
    ├── WorldSimulation.luau
    ├── ProgressionService.luau
    ├── PlayerDataService.luau
    ├── WorldSaveService.luau
    └── SocialService.luau
```

Each Service = one owning module for one slice of state (matches
[systems doc](./02-gameplay-systems.md)). Disaster effect **handlers** live beside `DisasterService`
(one file per disaster) so the roster is extended by adding a file + a Definition entry.

## 4. `src/client/` — Controllers

```
src/client/
├── init.client.luau              # bootstrap: load framework, start controllers
├── Controllers/
│   ├── InputController.luau
│   ├── UIController.luau
│   ├── PlacementController.luau   # building ghost preview
│   ├── CameraController.luau
│   ├── EffectsController.luau
│   └── SocialController.luau
└── UI/                            # reactive components (Fusion/Roact)
    ├── App.luau
    ├── HUD/
    ├── Crafting/
    ├── Research/
    ├── Storage/
    └── Components/                # shared UI primitives + theme
```

## 5. `src/first/` — ReplicatedFirst

```
src/first/
└── Loading.client.luau           # instant loading screen while game streams in
```

Kept tiny and dependency-free so it runs before anything else replicates.

## 6. Rojo mapping (`default.project.json`, conceptual)

```jsonc
{
  "name": "origins-game",
  "tree": {
    "$className": "DataModel",
    "ReplicatedStorage": {
      "Shared": { "$path": "src/shared" },
      "Packages": { "$path": "Packages" }        // Wally deps
    },
    "ReplicatedFirst": { "$path": "src/first" },
    "ServerScriptService": { "Server": { "$path": "src/server" } },
    "StarterPlayer": {
      "StarterPlayerScripts": { "Client": { "$path": "src/client" } }
    },
    "ServerStorage": { "$path": "src/serverstorage" }  // private assets/data (optional)
  }
}
```

Full DataModel reasoning is in [05 — DataModel Hierarchy](./05-datamodel-hierarchy.md).

## 7. Conventions

- **One module = one responsibility**, file named after what it exports (`GatheringService.luau`).
- **PascalCase** for modules/types, **camelCase** for locals/functions (see
  [conventions](./10-tooling-and-conventions.md)).
- **Tests co-located** as `Thing.spec.luau` next to `Thing.luau` (or under `tests/` — pick one; doc
  10 records the choice).
- **No binary art in Git** beyond small essentials; large models live as Roblox assets referenced by
  id in Definitions, with a manifest in `assets/`.

## 8. Why this structure scales for 5 years

- **Realm split = security by layout.** Server code can't leak to clients.
- **Definitions folder = content pipeline.** Designers add data; engineers rarely touch logic.
- **Flat Services/Controllers = discoverable.** New engineer finds the owner of any behavior in
  seconds.
- **Rojo + Git = real version control, review, CI** — impossible with Studio-native `.rbxl` workflows.

---

**Prev:** [« 03 — Architecture](./03-technical-architecture.md) &nbsp;|&nbsp;
**Next:** [05 — DataModel Hierarchy »](./05-datamodel-hierarchy.md)
