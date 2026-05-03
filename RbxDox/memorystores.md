# MemoryStores

> Upstream: <https://create.roblox.com/docs/cloud-services/memory-stores>
> Source: `content/en-us/cloud-services/memory-stores/index.md`, `best-practices.md`, `hash-map.md`, `queue.md`, `sorted-map.md`

## Purpose
**MemoryStoreService** is fast, cross-server, ephemeral key/value storage. Latency is ~10–30 ms (vs DataStore ~100+ ms), and it's designed for high-throughput coordination between servers — matchmaking pools, session tokens, live leaderboards, rate counters.

Data has TTLs (hours-to-day max, configurable), is **not durable across catastrophes**, and shouldn't hold anything you can't reconstruct. Server-only.

## Three structures

| Structure | Use for |
|---|---|
| **HashMap** | Cross-server key/value, like a small distributed dict. Fast reads/writes by key. |
| **SortedMap** | Same but with score-based ordering and range queries. Live leaderboards, priority queues. |
| **Queue** | FIFO with optional priority and visibility timeout. Matchmaking pools, work queues. |

## HashMap

```lua
local MSS = game:GetService("MemoryStoreService")
local map = MSS:GetHashMap("OnlineSessions")

map:SetAsync(key, value, expirationSeconds)            -- expirationSeconds ≤ 3,888,000 (45d)
local v = map:GetAsync(key)
map:RemoveAsync(key)

-- Atomic update
map:UpdateAsync(key, function(old)
    return (old or 0) + 1
end, expirationSeconds)
```

Best for "I want fast keyed lookups across all my game servers."

## SortedMap

Like HashMap but with a numeric `sortKey` per entry, allowing range queries:

```lua
local lb = MSS:GetSortedMap("LiveLeaderboard")

lb:SetAsync(userId, payload, expirationSeconds, sortKey)        -- sortKey orders entries

local top = lb:GetRangeAsync(Enum.SortDirection.Descending, 100) -- top 100
for _, item in top do
    print(item.key, item.value, item.sortKey)
end

lb:UpdateAsync(userId, function(old, sortKey)
    return newValue, newSortKey
end, expirationSeconds)
```

Use for live in-experience leaderboards, real-time matchmaking ranks.

## Queue

```lua
local q = MSS:GetQueue("Matchmaking", invisibilityTimeout?)

-- Producer
q:AddAsync(payload, expirationSeconds, priority?)

-- Consumer
local items, id = q:ReadAsync(count, allOrNothing?, waitTimeoutSeconds?)
for _, item in items do
    -- process
end
q:RemoveAsync(id)        -- ack — otherwise items reappear after invisibilityTimeout
```

`ReadAsync` returns up to `count` items and a receipt id. Items remain "invisible" for `invisibilityTimeout` seconds; if you don't `RemoveAsync(id)`, they pop back into the queue (at-least-once semantics — handlers must be idempotent).

Use for matchmaking, distributed task queues, broadcast event fanout.

## Patterns

### Cross-server "is user online anywhere"
```lua
local map = MSS:GetHashMap("OnlinePlayers")

Players.PlayerAdded:Connect(function(p)
    map:SetAsync(tostring(p.UserId), game.JobId, 600)
end)
Players.PlayerRemoving:Connect(function(p)
    map:RemoveAsync(tostring(p.UserId))
end)
```

### Live leaderboard
```lua
local lb = MSS:GetSortedMap("Wins")
local function recordWin(player)
    lb:UpdateAsync(tostring(player.UserId), function(old, oldScore)
        return player.Name, (oldScore or 0) + 1
    end, 86400)
end

local function topN(n)
    return lb:GetRangeAsync(Enum.SortDirection.Descending, n)
end
```

### Matchmaking queue
```lua
local q = MSS:GetQueue("Match", 30)   -- 30s invisibility

-- Producer (player presses "Find Match")
q:AddAsync({userId = p.UserId, mmr = mmr}, 300)

-- Consumer (matchmaker server runs every 5s)
local items, id = q:ReadAsync(8, false, 0)
if #items >= 8 then
    -- pair them up, teleport them
    q:RemoveAsync(id)
else
    -- don't ack; let them reappear for next attempt
end
```

## Limits

- Per-experience throughput is shared. Both reads and writes count.
- Item size: ≤ 32 KB serialized.
- Expiration: ≤ 45 days.
- Each operation costs against your per-minute budget; design for batched/sparse access.

Errors come back as Lua errors with specific codes; pcall and back off as you would for DataStores.

## When to choose what

| Need | Use |
|---|---|
| Player state that must survive forever | DataStore |
| Real-time cross-server coordination, hours of life | MemoryStore |
| Cross-server one-shot message ("round started") | MessagingService — see [messaging-http.md](messaging-http.md) |
| Talking to an external service | HttpService |

## Pitfalls

- **Treating MemoryStore as durable.** Don't store player progression here.
- **Skipping `RemoveAsync` on Queue receipts.** Items reappear → duplicate processing. Idempotent consumers are mandatory.
- **Per-frame writes.** You'll burn quota and get throttled.
- **Forgetting expiration.** Required parameter; choose sensibly (longer wastes budget, shorter loses data).
- **Using server-side caching as truth.** A server-local cache and a MemoryStore can drift; for critical reads, hit the store.
- **Sending Instances.** Only serializable Lua values.

## See also
- [datastores.md](datastores.md) — durable storage
- [messaging-http.md](messaging-http.md) — pub/sub messaging
- [security.md](security.md) — server-only access
