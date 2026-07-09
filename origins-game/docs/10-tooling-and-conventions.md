# 10 — Tooling & Conventions

> The engineering discipline that keeps ORIGINS maintainable for 5+ years and many contributors.
> A professional project is defined as much by its tooling and conventions as by its architecture.

---

## 1. Language: Luau, strict mode

- **`--!strict`** at the top of every module. Strict Luau catches type errors before runtime — a huge
  win for a large, long-lived codebase.
- Shared **`Types`** module holds cross-cutting type definitions; Definitions are fully typed so bad
  content fails typecheck.
- Prefer explicit types on module boundaries (function params/returns, exported types).

## 2. Source sync: Rojo + Git

- **Source of truth is Git** (`.luau` files). **Rojo** syncs `src/` into Studio; Studio is a
  viewport/test harness, not where code is authored.
- **Why over Studio-native:** real version control, code review, diffs, branches, CI, and blame —
  none of which work on a monolithic `.rbxl`. This is the single biggest maintainability decision for
  a commercial project.
- A base place (`.rbxlx`) may hold non-scripted assets (terrain, models) checked in as needed; all
  logic lives in `src/`.

## 3. Dependencies: Wally

- **Wally** (`wally.toml` + `wally.lock`) manages Luau package dependencies, installed into
  `Packages/` and mapped by Rojo into `ReplicatedStorage/Packages`.
- Pin versions; commit the lockfile. Keep external deps minimal (our stack is deliberately
  in-house-first — see [architecture](./03-technical-architecture.md)).
- Likely early deps: a **Signal** util, **ProfileStore/ProfileService** (persistence), a **testing**
  runner, and (later) a **reactive UI** lib (Fusion or Roact).

## 4. Testing: TestEZ / Jest-Lua

- **Pure-logic modules are unit-tested without a running game** (the [module
  architecture](./06-module-architecture.md) mandates extracting rules into pure functions).
- Tests co-located as `Thing.spec.luau` (or under `tests/` — **choose one; this doc is the record:
  co-located**).
- Priorities to test: economy math (anti-dupe invariants), research/tier transitions, save
  migrations, growth/respawn timestamp math, validation guards.
- Aim for meaningful coverage of **rules**, not Roblox glue.

## 5. Linting & formatting

- **Selene** (`selene.toml`) — static analysis/linting for Luau; catches bugs and enforces rules.
- **StyLua** (`stylua.toml`) — deterministic formatting; **no formatting debates in review**.
- Both run in CI and ideally as pre-commit hooks. Formatting is never a manual concern.

## 6. Continuous Integration: GitHub Actions

- **`.github/workflows/ci.yml`** runs on every PR: install deps (Wally) → **Selene lint** → **StyLua
  check** → **run tests** → (optionally) **Rojo build** to verify the place compiles.
- **PRs cannot merge red.** CI is the quality gate that keeps `main` always shippable.

## 7. Git workflow

- **`main` is protected and always releasable.** No direct pushes.
- **Feature branches** → PR → review → CI green → merge. Small, focused PRs.
- **Conventional-style commits** (`feat:`, `fix:`, `refactor:`, `docs:`, `chore:`) for readable
  history and easy changelogs.
- Design changes update **`/docs`** in the same PR — docs and code evolve together (docs are part of
  the definition of done).

## 8. Naming & code conventions

| Thing | Convention | Example |
|-------|-----------|---------|
| Modules / types / classes | PascalCase | `GatheringService`, `ResourceDef` |
| Functions / locals | camelCase | `tryGather`, `nodeId` |
| Constants | SCREAMING_SNAKE or `Constants.X` | `MAX_STACK` |
| Definition ids | PascalCase string ids | `"IronPickaxe"` |
| Files | match the exported name | `GatheringService.luau` |
| Booleans | `is/has/can` prefix | `isUnlocked` |

- **One responsibility per module**; keep files focused.
- **No magic numbers** — tunables live in `Constants` or Definitions.
- **Comments explain _why_, not _what_**; match the surrounding code's density.

## 9. Environments & config

- Separate **dev / staging / production** experiences (or place versions) so tests never touch live
  worlds' DataStores.
- Feature flags / config module for risky features; never hardcode environment specifics.
- Secrets (if any HTTP services later) never in Git.

## 10. Observability

- **Structured logging** with levels (debug/info/warn/error); a logging util, not bare `print`.
- **Telemetry/analytics** events through the guarded [Net](./07-networking.md) layer:
  retention, tier funnel, session length, disaster outcomes.
- **Error reporting** (Roblox `ScriptContext`/Analytics or a service) so production issues surface.

## 11. Documentation discipline

- `/docs` is the **living design source of truth**; update it with the change that affects it.
- Public modules carry a short header comment (purpose + owner system).
- Architecture decisions of consequence are recorded (a lightweight ADR note in the relevant doc's
  "decisions" table).

## 12. Onboarding a new engineer (the 5-year test)

A new contributor should be able to:
1. Clone → `wally install` → `rojo serve` → open the place → running game.
2. Read `/docs` top-to-bottom and understand the whole system in an afternoon.
3. Find the owner of any behavior via the flat `Services/`·`Controllers/` layout.
4. Add a new resource/building/disaster by **editing Definitions** + (if needed) one handler, with
   tests, and ship it via a reviewed PR that CI validates.

If all four are true, the tooling has done its job.

---

**Prev:** [« 09 — Roadmap](./09-roadmap.md) &nbsp;|&nbsp;
**Back to:** [Documentation Index](./README.md)
