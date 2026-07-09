# ORIGINS — Design Documentation

> This folder is the **living design source of truth** for ORIGINS. It is written to be read
> top-to-bottom by a new team member and to stay accurate as the project evolves. **Phase 1 is
> planning: no gameplay code exists yet — the system is designed first.**

## Reading order

Read in order the first time; each doc links to the next.

| # | Document | What you'll learn |
|---|----------|-------------------|
| 00 | [Game Design Document](./00-game-design-document.md) | The vision, audience, core fantasy, civilization tiers, and the locked V1 scope (resources, buildings, tools, research, disasters) + risks |
| 01 | [Core Gameplay Loop](./01-core-gameplay-loop.md) | The three nested loops (core → session → meta), onboarding, and how retention is engineered |
| 02 | [Gameplay Systems](./02-gameplay-systems.md) | All 12 major systems, each with Purpose · Responsibilities · Dependencies · Scalability · Future · Recommended implementation |
| 03 | [Technical Architecture](./03-technical-architecture.md) | Server-authority, data-driven design, framework choice, simulation model, performance & security |
| 04 | [Folder Structure](./04-folder-structure.md) | On-disk repo + Rojo source layout and why it's laid out that way |
| 05 | [DataModel Hierarchy](./05-datamodel-hierarchy.md) | Where everything lives in the running Roblox DataModel, per-service rationale |
| 06 | [Module Architecture](./06-module-architecture.md) | Service/Controller/Definition patterns, lifecycle, testability, the module checklist |
| 07 | [Networking](./07-networking.md) | The client/server contract, the typed Net layer, anti-exploit & replication |
| 08 | [Saving System](./08-saving-system.md) | **The defining decision** — persistent private-server worlds, two stores, integrity & migrations |
| 09 | [Roadmap](./09-roadmap.md) | Phased build plan to V1 + the multi-year update vision |
| 10 | [Tooling & Conventions](./10-tooling-and-conventions.md) | Rojo, strict Luau, testing, lint/format, CI, git workflow, naming |

## The five architectural pillars (TL;DR)

If you read nothing else, internalize these — every doc reinforces them:

1. **Server-authoritative.** The server owns all truth; clients send *intents* and render. The only
   defensible posture at millions of visits. → [03](./03-technical-architecture.md), [07](./07-networking.md)
2. **Data-driven content.** Resources, recipes, buildings, research, and disasters are **Definitions**
   (data), not code. Adding content = adding data, never a rewrite. → [03](./03-technical-architecture.md), [06](./06-module-architecture.md)
3. **Persistent worlds = private/reserved servers.** Public Roblox servers are ephemeral and would
   kill the "build over weeks" fantasy; each world is a reserved server keyed to its own save. →
   [08](./08-saving-system.md)
4. **Modular services with single ownership.** Every piece of state has exactly one owning service;
   two-phase (`Init`/`Start`) lifecycle; pure, testable rules. → [02](./02-gameplay-systems.md), [06](./06-module-architecture.md)
5. **Professional toolchain for longevity.** Rojo + Git + strict Luau + tests + lint + CI, so the
   project survives 5 years and many hands. → [10](./10-tooling-and-conventions.md)

## Status & how to use these docs

- **Status:** Phase 1 — Planning. Design is being locked before any gameplay code.
- **Discipline:** design changes update these docs in the same PR as the change. Docs and code evolve
  together (see [10 — Tooling](./10-tooling-and-conventions.md)).
- **Next step after sign-off:** Phase 0 foundations (repo tooling + framework + Net skeleton) per the
  [roadmap](./09-roadmap.md) — still no gameplay logic until the foundations are in place.

---

_"The world should tell the story of its players."_
