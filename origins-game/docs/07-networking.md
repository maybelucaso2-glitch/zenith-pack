# 07 — Client/Server Networking Plan

> The contract for how the client and server talk. In a server-authoritative game targeting millions
> of visits, networking **is** the anti-exploit surface — so it's designed as one guarded layer, not
> scattered remotes.

---

## 1. First principles

1. **Client sends intent; server decides.** No remote ever *tells* the server what happened — it
   *asks* the server to do something, and the server validates and commits.
2. **One channel:** every cross-realm call goes through the **Net** layer. No system touches raw
   `RemoteEvent`/`RemoteFunction`s.
3. **Validate at the boundary:** shape, rate, range, ownership, tier, affordability — before any
   handler logic runs.
4. **Replicate deltas, not dumps.** Send what changed, throttled, not full state tables.

## 2. Remote types & when to use them

| Primitive | Direction | Use for | Notes |
|-----------|-----------|---------|-------|
| **RemoteEvent** | C→S | Player intents (gather, place, craft, contribute) | Fire-and-forget; server validates; result comes back via state delta or a result event |
| **RemoteEvent** | S→C | State deltas, notifications, disaster warnings, feed updates | Batched/throttled |
| **RemoteFunction** | C→S | Rare request/response where the client must **wait** (e.g., open a UI needing a snapshot) | Used sparingly — yields; never trust for authority; guard against exploit-thrown errors |
| **UnreliableRemoteEvent** | S→C | High-frequency cosmetic/positional effects that tolerate loss | Only for non-authoritative FX |

**Default to RemoteEvents.** RemoteFunctions are avoided for hot paths (they yield and can be abused);
we prefer event + delta so the client is never blocked and the server is never hostage to a client
callback.

## 3. Raw remotes vs. a networking library — decision

| Option | Pros | Cons | Verdict |
|--------|------|------|---------|
| **Raw RemoteEvents/Functions everywhere** | No abstraction | Validation/rate-limiting copy-pasted per remote; easy to forget a check; no types | ❌ Unsafe at scale |
| **Third-party lib** (e.g., a networking package) | Batteries, buffers/serialization | External dependency & opinions; may not match our Definitions/typing; migration risk | ⚠️ Consider for buffer serialization later |
| **Thin in-house typed `Net` layer** | Central validation + rate limit + types; tiny; fits our stack; one place to audit | We maintain it | ✅ **Recommended** |

**Recommendation: a thin in-house `Net` layer** over raw remotes. It is the single place that
enforces every safety rule, so security is *structural*, not dependent on each engineer remembering
to validate. A third-party buffer-serialization package can be adopted **under** this layer later
without changing call sites.

## 4. The `Net` layer design

```
                         ┌────────────────────────── Net (shared) ──────────────────────────┐
  Client Controller ──►  │  Client API: Net.Client:Fire("Gather", payload)                    │
                         │      → serialize → send on the one RemoteEvent for "Gather"        │
                         └───────────────────────────────┬───────────────────────────────────┘
                                                          ▼
                         ┌────────────────────── Server middleware pipeline ──────────────────┐
                         │  1. Rate limit (per player, per remote)                             │
                         │  2. Schema/type validation (payload shape)                          │
                         │  3. Context checks (range, ownership, tier, cooldown)               │
                         │  4. Dispatch to the bound Service handler                           │
                         │  5. Handler mutates state atomically → enqueues deltas              │
                         └───────────────────────────────┬───────────────────────────────────┘
                                                          ▼
                         ┌──────────────── Replication (S→C) ────────────────┐
                         │  Batched delta events, throttled per tick          │
                         └────────────────────────────────────────────────────┘
```

- **Registry:** `Shared/Net/Remotes.luau` declares every remote name + its typed payload once.
- **Middleware:** rate-limit → validate → context-guard runs for *every* inbound event, centrally.
- **Binding:** Services register handlers by remote name (`Net.Server:On("Gather", handler)`); they
  receive only **validated** payloads.

## 5. Rate limiting & anti-exploit

- **Per-player, per-remote token buckets.** Gather/craft/place are capped to plausible human rates;
  excess is dropped and flagged.
- **Range & ownership checks** server-side for every world action (is the player near that node?
  do they own that structure?).
- **Atomic economy mutations** only via `InventoryService` (anti-dupe chokepoint) — a remote can
  request a craft, but only the service moves items, transactionally.
- **No client authority ever:** positions used for validation are server-known; client-reported
  numbers are inputs to validate, never outputs to trust.
- **Payload sanitization:** reject malformed/oversized payloads before handler logic; never
  `loadstring` client data.

## 6. Replication strategy

- **State store per client:** the server maintains what each client knows and sends **deltas** on
  change (inventory changed → send the delta; not the whole inventory).
- **Batching per tick:** deltas are coalesced and flushed on the simulation tick cadence, not on
  every micro-change, to bound event volume.
- **Interest management via StreamingEnabled:** the world (nodes/structures) replicates through
  Roblox streaming; gameplay state (inventory, research) replicates through Net deltas.
- **Snapshots on join:** a joining client gets a one-time snapshot of relevant state, then deltas
  thereafter.

## 7. Typical round-trips (examples)

**Gather:**
```
Client: tap tree ─► Net.Client:Fire("Gather", { nodeId })
Server: rate-limit ✓ → validate shape ✓ → range ✓ → tool tier ✓ → node not depleted ✓
        → InventoryService:TryAdd(player, "Wood", yield) → deplete node
        → delta: inventory +Wood, node health-- ─► client updates HUD + node visual
```

**Place building:**
```
Client: ghost preview (local only) ─► Net.Client:Fire("PlaceBuilding", { defId, cframe })
Server: rate-limit ✓ → tier unlocked ✓ → affordable ✓ → legal spot ✓
        → InventoryService deduct → BuildingService commit + register
        → delta: structure added (streamed in), inventory -cost
```

**Contribute to research:**
```
Client: Net.Client:Fire("ContributeResearch", { resourceId, amount })
Server: validate → withdraw from Storage → ResearchService:AddProgress
        → maybe TierChanged broadcast (S→C to everyone) → world evolves
```

## 8. Scalability concerns

- Hot remotes (gather) dominate traffic → keep payloads tiny (ids, not tables), batch deltas, cap
  rates.
- Broadcast events (tier-up, feeds) → send once, aggregated; never per-recipient recomputation.
- Consider **buffer serialization** for the highest-volume events in a later optimization pass
  (slots under the Net layer without touching call sites).

## 9. Future expansion

- Cross-server messaging (MemoryStore/MessagingService) for global events, presence, world hosting.
- Buffer-packed replication for bandwidth on huge worlds.
- Replay/telemetry stream (fire-and-forget analytics events through the same guarded layer).

---

**Prev:** [« 06 — Module Architecture](./06-module-architecture.md) &nbsp;|&nbsp;
**Next:** [08 — Saving System »](./08-saving-system.md)
