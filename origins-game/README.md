# ORIGINS

> A cooperative civilization-building game for Roblox.

Players spawn on an untouched island and, working **together**, slowly transform it into a
thriving civilization over days, weeks, and months. Gather resources, craft tools, raise
buildings, research technologies as a server, survive dynamic disasters, and leave behind a world
that tells the story of the people who built it.

ORIGINS blends the systemic depth of **Minecraft** and **Rust** (without a PvP focus), the
long-arc progression of **Civilization** and **Animal Crossing**, and the cozy cooperative economy
of **Roblox Islands** and **The Survival Game**.

## Design pillars

1. Every server begins with an untouched island.
2. Players gather resources.
3. Players craft better tools.
4. Players build structures.
5. The server researches technologies **together**.
6. Civilization visually evolves.
7. Dynamic disasters reshape the world.
8. Exploration unlocks new possibilities.
9. Every server develops differently.
10. The world tells the story of its players.

## Project status

**Phase 1 — Planning.** This repository currently contains **design documentation only**. No
gameplay code is written yet. The full system design is being locked before implementation so the
project can scale to millions of visits and remain maintainable for 5+ years.

## Documentation

All design docs live in [`/docs`](./docs). Start with the
[documentation index](./docs/README.md).

| # | Document | Topic |
|---|----------|-------|
| 00 | [Game Design Document](./docs/00-game-design-document.md) | Vision, audience, V1 scope, content |
| 01 | [Core Gameplay Loop](./docs/01-core-gameplay-loop.md) | Moment-to-moment → session → meta loops |
| 02 | [Gameplay Systems](./docs/02-gameplay-systems.md) | Every major system, fully specified |
| 03 | [Technical Architecture](./docs/03-technical-architecture.md) | Server authority, data-driven design, framework |
| 04 | [Folder Structure](./docs/04-folder-structure.md) | Repo + Rojo source layout |
| 05 | [DataModel Hierarchy](./docs/05-datamodel-hierarchy.md) | Roblox service tree |
| 06 | [Module Architecture](./docs/06-module-architecture.md) | Services, Controllers, Definitions |
| 07 | [Networking](./docs/07-networking.md) | Client/server contract, anti-exploit |
| 08 | [Saving System](./docs/08-saving-system.md) | Persistent worlds + player data |
| 09 | [Roadmap](./docs/09-roadmap.md) | Phased build plan + future updates |
| 10 | [Tooling & Conventions](./docs/10-tooling-and-conventions.md) | Rojo, Luau, tests, CI, git workflow |

## Tech stack (planned)

- **Language:** Luau (strict typing)
- **Sync:** [Rojo](https://rojo.space) — source lives in Git, syncs into Roblox Studio
- **Testing:** TestEZ / Jest-Lua
- **Lint / format:** Selene + StyLua
- **CI:** GitHub Actions
- **Persistence:** DataStoreService (session-locked), MemoryStore (transient)

---

_ORIGINS is in active pre-production. Design is intentionally locked before code._
