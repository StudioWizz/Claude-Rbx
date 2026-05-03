# DataStores

> Upstream: <https://create.roblox.com/docs/cloud-services/data-stores>
> Source: `content/en-us/cloud-services/data-stores/index.md`, `best-practices.md`, `error-codes-and-limits.md`

## Purpose
**DataStoreService** is Roblox's persistent key/value store. It's the right place for player progression, inventory, settings, leaderboards — anything that should outlive a server. Server-only. Throttled and rate-limited; you must design with retries and idempotency.

## Core API

```lua
local DSS = game:GetService("DataStoreService")

local store = DSS:GetDataStore("PlayerData")          -- a regular DataStore
local ordered = DSS:GetOrderedDataStore("Top100")     -- supports sorted queries

-- Read
local data = store:GetAsync(key)

-- Write (overwrites)
store:SetAsync(key, value, {userId})                  -- userId list lets Roblox map data to players for compliance

-- Update (read-modify-write atomically)
store:UpdateAsync(key, function(old, info)
    old = old or {coins = 0}
    old.coins += 100
    return old, {userId}, {key = "value"}              -- return value, userIds, metadata
end)

-- Increment a number atomically
store:IncrementAsync(key, delta)                       -- OrderedDataStore-friendly

-- Remove
store:RemoveAsync(key)
```

All `*Async` calls **yield** and can fail (network, throttle). Always wrap in `pcall`:

```lua
local ok, result = pcall(function()
    return store:GetAsync(key)
end)
if not ok then warn("DS read failed:", result) end
```

## Best practices

### 1. Always use `UpdateAsync` for mutations
`SetAsync` overwrites blindly — concurrent servers (or rapid sessions) can stomp each other. `UpdateAsync` reads the current value and runs your transform server-side; if a conflict happens, it retries automatically.

```lua
local function addCoins(userId, n)
    local key = "Player_" .. userId
    return store:UpdateAsync(key, function(old)
        old = old or {coins = 0}
        old.coins += n
        return old
    end)
end
```

### 2. Retry with backoff
Roblox throttles aggressive callers. Wrap every async call in retry-with-backoff:

```lua
local function retry(fn, attempts: number?)
    attempts = attempts or 5
    local delay = 1
    for i = 1, attempts do
        local ok, res = pcall(fn)
        if ok then return true, res end
        if i == attempts then return false, res end
        task.wait(delay)
        delay *= 2
    end
end

local ok, data = retry(function() return store:GetAsync(key) end)
```

### 3. Save on critical events, not every frame
- On `PlayerRemoving` (player leaves) — save final state.
- On `BindToClose` (server shutting down) — save all online players' state with `:WaitForChild` so requests don't get killed.
- Optionally autosave every 60–120 s during long sessions.

```lua
game:BindToClose(function()
    for _, p in game.Players:GetPlayers() do
        savePlayer(p)
    end
end)
```

If the place is in production, BindToClose has a ~30 s window before forced shutdown — don't queue 100 sequential saves.

### 4. Session locking (avoid double-saving the same player from two servers)
The classic bug: a player teleports between servers and both servers race to save. Solutions:
- **ProfileService** / **ProfileStore** open-source library — recommended; battle-tested.
- Hand-rolled session lock: store a `sessionId` field and on read verify the calling server owns it.

### 5. Limit payload size
Each DataStore key holds up to **4 MB** of JSON-encoded data. Don't dump the whole player history in one key — partition by purpose (`PlayerSettings`, `PlayerInventory`).

## OrderedDataStore (leaderboards)

```lua
local lb = DSS:GetOrderedDataStore("Wins")
lb:SetAsync(userId, winCount)

local pages = lb:GetSortedAsync(false, 100)            -- descending, 100 per page
local page = pages:GetCurrentPage()
for rank, entry in page do
    print(rank, entry.key, entry.value)
end
if not pages.IsFinished then pages:AdvanceToNextPageAsync() end
```

Values must be integers. Useful for high-score boards.

## Limits and quotas

Quotas are per-experience and depend on player count, but the order of magnitude:

| Operation | Per minute, per server | Per minute, per key |
|---|---|---|
| GetAsync | 60 + (player count × 10) | 4 |
| SetAsync / UpdateAsync / IncrementAsync | 60 + (player count × 10) | 4 |
| RemoveAsync | same | 4 |

Hitting these throws `429`-style errors. Backoff and retry; don't write per-frame.

A single key can be written at most ~once per 6 seconds without throttling.

## DataStores in Studio

By default DataStoreService doesn't work in Studio for safety. Enable in `Game Settings → Security → Enable Studio Access to API Services`.

## Patterns

### Player load/save skeleton
```lua
local Players = game:GetService("Players")
local DSS = game:GetService("DataStoreService")
local store = DSS:GetDataStore("PlayerData_v1")

local cache = {}      -- userId → data, in-memory while online

local function load(player)
    local key = "U_" .. player.UserId
    local ok, data = retry(function() return store:GetAsync(key) end)
    if not ok then warn("load failed", player); return end
    cache[player.UserId] = data or {coins = 0, level = 1}
end

local function save(player)
    local data = cache[player.UserId]
    if not data then return end
    retry(function()
        return store:UpdateAsync("U_" .. player.UserId, function(_)
            return data, {player.UserId}
        end)
    end)
end

Players.PlayerAdded:Connect(load)
Players.PlayerRemoving:Connect(save)
game:BindToClose(function()
    for _, p in Players:GetPlayers() do save(p) end
end)
```

## Pitfalls

- **`SetAsync` for player data.** Use `UpdateAsync` — it's atomic and handles concurrent edits.
- **Saving every Heartbeat.** You'll exhaust the quota in seconds. Save on events.
- **No `pcall`.** Network blip → unhandled error → no save → angry player. Always pcall + retry.
- **Storing tables with mixed array/hash.** JSONEncode is happy but you can lose array order. Use `{1, 2, 3}` OR `{a=1, b=2}`, not both in one table.
- **Storing functions, Instances, or userdata.** They don't serialize. Store the data needed to reconstruct.
- **Forgetting `userId` list.** Pass `{player.UserId}` so Roblox can comply with player data export / right-to-be-forgotten requests.
- **Not handling `BindToClose` in Studio test mode.** It still fires, but DataStores may not be available — guard.

## See also
- [memorystores.md](memorystores.md) — ephemeral cross-server storage
- [security.md](security.md) — server-only persistence
- [services.md](services.md) — DataStoreService overview
- [player-lifecycle.md](player-lifecycle.md) — when to save (PlayerRemoving + BindToClose); reconnect coordination
- [save-slots.md](save-slots.md) — multi-key extensions of this card's single-key pattern
- [offline-processing.md](offline-processing.md) — the `os.time()` save-and-accrue pattern
- [inventory.md](inventory.md), [progression.md](progression.md), [pets.md](pets.md) — common profile fields persisted via this card
- [community-libraries.md](community-libraries.md) — **ProfileService / DataStore2** wrap the patterns on this card with battle-tested session locking
- [time-trials.md](time-trials.md) — OrderedDataStore for fastest-time leaderboards
