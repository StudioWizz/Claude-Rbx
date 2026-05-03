# Offline Processing (Idle Rewards, Passive Income)

> Source: distilled from idle-game and tycoon patterns common in Roblox

## Purpose
Many games reward players for time spent away — idle/clicker games accumulate currency offline, tycoons collect passive income from owned assets, RPGs grant "rested XP" or daily resource ticks. The pattern is always the same shape: **on save, record `os.time()`. On next load, compute `delta = now - lastSave`, apply rewards proportional to delta, cap at a maximum to prevent abuse.**

This card covers the discipline: where to do the math, what to cap, what to *never* trust the client about.

## Core pattern

```lua
--!strict
local DataStoreService = game:GetService("DataStoreService")
local Players = game:GetService("Players")

local store = DataStoreService:GetDataStore("PlayerData_v1")

local OFFLINE_RATE = 1            -- coins per second offline
local MAX_OFFLINE_HOURS = 12      -- cap (anti-abuse + sane reward)

type SaveData = {
    coins: number,
    lastOnline: number,           -- os.time() at last save
    -- ...
}

-- On load
local function loadAndAccrue(player: Player): SaveData
    local key = "Player_" .. player.UserId
    local data: SaveData? = nil
    pcall(function() data = store:GetAsync(key) end)
    data = data or {coins = 0, lastOnline = os.time()}

    local now = os.time()
    local elapsed = math.max(0, now - data.lastOnline)              -- guard against negative deltas
    local capped = math.min(elapsed, MAX_OFFLINE_HOURS * 3600)
    local earned = math.floor(capped * OFFLINE_RATE)

    data.coins += earned
    data.lastOnline = now

    if earned > 0 then
        showOfflineRewardToast(player, earned, capped)
    end
    return data
end

-- On save
local function save(player: Player, data: SaveData)
    data.lastOnline = os.time()
    pcall(function()
        store:UpdateAsync("Player_" .. player.UserId, function(_) return data, {player.UserId} end)
    end)
end

Players.PlayerAdded:Connect(function(p)
    local data = loadAndAccrue(p)
    -- attach data to per-player state
end)
```

## Cap discipline (the most important thing)

Without a cap, leaving a tab open for a week then logging in grants a week of idle reward. Worse, **a player can manipulate `os.time()` by changing their device clock** — the server's `os.time()` is safe, but if you trust client-reported deltas, exploits are trivial.

**Always:**
- Compute `delta` server-side using server `os.time()`.
- Cap at a fixed maximum (12 hours is typical; can be higher for prestige tiers).
- Treat negative deltas (`now < lastOnline`) as zero — protects against clock skew across DataStore regions.

**Never:**
- Trust a client-supplied "I was offline for X seconds" value.
- Use `tick()` for persistence (it's monotonic per-server and resets on shutdown).
- Use `os.clock()` (CPU time, not wall-clock).

`os.time()` returns Unix epoch seconds in UTC, server-side. That's the only timestamp source you should persist.

## Showing offline rewards on join

The right UX is a small toast or modal: "Welcome back! You earned 4,320 coins offline." Don't surprise the player with a silent balance bump.

```lua
local function showOfflineRewardToast(player: Player, earned: number, secondsOffline: number)
    local hours = math.floor(secondsOffline / 3600)
    local minutes = math.floor((secondsOffline % 3600) / 60)
    OfflineRewardRemote:FireClient(player, {
        coins = earned,
        timeText = string.format("%dh %dm", hours, minutes),
        wasCapped = secondsOffline >= MAX_OFFLINE_HOURS * 3600,
    })
end
```

Client builds the UI (see [ui-system.md](ui-system.md), [tweens-animation.md](tweens-animation.md)).

## Rate scaling — let upgrades affect offline too

The simple `OFFLINE_RATE` constant doesn't work once players have upgrades that boost income. Two options:

### Option A: snapshot the rate on save
Save the player's current per-second income alongside `lastOnline`. On load, multiply elapsed by that rate.
```lua
data.coinsPerSecond = computeCurrentRate(data)   -- update whenever upgrades change
data.lastOnline = os.time()
```
Pro: simple, idempotent. Con: doesn't account for upgrades the player would have bought during offline time.

### Option B: tick the simulation forward in batches
For tycoons that have multi-stage income (resource A → B → C), simulate the offline interval in chunks (every "minute" of offline time, run one tick of the resource pipeline) up to the cap.
```lua
local TICK = 60   -- one game-tick = 60 real seconds
local ticksToRun = math.floor(capped / TICK)
for _ = 1, ticksToRun do
    runOneTick(data)
end
```
Pro: accurate for non-linear systems. Con: slow if cap is 12h × 60s = 720 ticks; consider a closed-form math shortcut where possible.

**Don't loop per-second** — even the 12-hour cap = 43,200 iterations, which yields and risks DataStore quota issues. Tick at coarse intervals (per-minute, per-5-minute) and use closed-form math where the income function is linear.

## Offline reward + prestige / multipliers

Common pattern: a "VIP Pass" gamepass (see [monetization.md](monetization.md)) extends `MAX_OFFLINE_HOURS` from 12 to 24, or grants a 2× multiplier. Apply server-side after looking up gamepass ownership:

```lua
local maxHours = MAX_OFFLINE_HOURS
local rate = OFFLINE_RATE
if MarketplaceService:UserOwnsGamePassAsync(player.UserId, VIP_PASS) then
    maxHours = 24; rate *= 2
end
```

## Combining with save slots

If your game uses multiple save slots ([save-slots.md](save-slots.md)), accrue offline only for the slot the player was last using (`lastUsedSlot` in the Meta key). Granting offline rewards to *every* slot would let players exploit by simply having more slots.

```lua
local meta = SaveSlots.GetMeta(player.UserId)
local slot = meta and meta.lastUsedSlot
if slot then
    local data = SaveSlots.LoadSlotAsync(player.UserId, slot)
    -- accrue offline for this slot only, then continue normal load
end
```

## Cross-server / multi-place caveats

If your experience has multiple places (a hub + minigames, or sharded servers via TeleportService — see [teleport.md](teleport.md)):

- Don't accrue offline rewards in *every* place's load — only the canonical "main" place.
- A player jumping between places isn't "offline" — `lastOnline` should be updated whenever they save in *any* place.
- Use a `lastSavedAnywhere` field, updated on every save in every place; use it as the offline-baseline.

## Anti-abuse

- **Cap server-side, always.** No "client says they were offline 100 hours."
- **Detect clock skew** (`now - lastOnline` is huge OR negative) and log/cap defensively.
- **Don't grant offline currency directly to a value the client can read before validation.** The pattern above grants on load, then sends the toast — the client never tells the server how much it should get.
- **Rate-limit "claim" buttons** if you have a manual "Claim offline rewards" button. Each click could attempt a re-grant if you're not careful — the load-time grant + cleared `pendingOffline` flag is the safest pattern.

## Patterns

### Manual "claim" button (idle game style)
Some idle games show "Tap to claim X coins" on join rather than auto-applying:
```lua
-- Server, on load
data.pendingOfflineReward = earned         -- compute but don't apply yet
-- Don't add to coins; client sees the pending amount and shows the claim button

-- Client clicks claim → fires ClaimOfflineRemote
ClaimOfflineRemote.OnServerEvent:Connect(function(player)
    local data = activeData[player]; if not data then return end
    if data.pendingOfflineReward and data.pendingOfflineReward > 0 then
        data.coins += data.pendingOfflineReward
        data.pendingOfflineReward = 0
        save(player, data)
    end
end)
```

### Daily login bonus (separate from idle accrual)
Tracks **calendar day boundaries** rather than continuous time:
```lua
local today = os.date("!*t", os.time()).yday    -- UTC day-of-year
local lastClaim = data.lastDailyBonusYday or 0
if lastClaim ~= today then
    data.coins += DAILY_BONUS
    data.lastDailyBonusYday = today
    data.consecutiveDays = (data.lastConsecutiveDay == today - 1) and (data.consecutiveDays + 1) or 1
end
```
Use UTC consistently — players should not be able to switch timezones for a second daily bonus.

## Pitfalls

- **`tick()` for persistence.** It's `os.clock()` analogue — non-monotonic across server restarts. Use `os.time()`.
- **No cap.** A player who leaves the game open for a year returns to a year of rewards. Always cap.
- **Cap too low.** A 1-hour cap on a game where 8-hour shifts are normal feels punishing. 12 hours is a reasonable floor for daily players.
- **Granting rewards before validating data.** If `loadProfile` succeeds but `lastOnline` is missing (legacy save), don't default to `os.time() - 100*3600` and grant 100 hours. Default to `os.time()` (zero earned) and start fresh.
- **Per-frame offline tick simulation.** Even a 12-hour cap is 43,200 seconds; iterating per-second yields constantly. Use coarse ticks or closed-form math.
- **Saving `lastOnline = os.time()` on every Heartbeat.** It's effectively continuous, which means offline calculations always show ≈0. Save `lastOnline` only on PlayerRemoving + BindToClose + the periodic autosave.
- **Trusting `os.date()` on the client.** Client local time can be anything. `os.date()` server-side is UTC-correct.
- **Granting offline rewards to AFK players.** A player who joined and then went AFK accumulates "online" time even though they're effectively offline. Either mark AFK time as offline (see [player-lifecycle.md](player-lifecycle.md)) or accept it.
- **Multi-server race.** If a player joins server A while server B is still in the middle of saving them, server A might read a stale `lastOnline` and grant double rewards. Use session locking (ProfileService) to prevent this.

## See also
- [datastores.md](datastores.md) — the underlying save/load mechanics
- [player-lifecycle.md](player-lifecycle.md) — when load and save fire (PlayerAdded → load → accrue; PlayerRemoving → save)
- [save-slots.md](save-slots.md) — accrue per active slot, not per slot
- [monetization.md](monetization.md) — gamepass/devproduct multipliers on offline rate
- [teleport.md](teleport.md) — multi-place implications
- [community-libraries.md](community-libraries.md) — **ProfileService** session locking prevents the multi-server race
- [security.md](security.md) — never trust client-reported time
- [growth-systems.md](growth-systems.md) — farming/plants are a canonical use case (plants grow while offline)
