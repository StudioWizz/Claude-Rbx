# Time, Day/Night & Weather

> Source: distilled from the Lighting/Atmosphere APIs and standard Roblox time-of-day + weather patterns

## Purpose
Three related-but-distinct clocks live in any non-trivial Roblox game: **real time** (server `os.time()`, UTC, used for daily resets and scheduled events), **game time** (a server-tracked "in-world" clock that runs at game-pace — 1 game-day = 24 real minutes, say), and **`Lighting.ClockTime`** (the visual sun-position clock, 0–24, that drives sky/lighting). This card pulls them together, plus the weather-state-machine pattern that drives rain/snow/storm transitions. `lighting.md` covers the visual rendering side; this card is about the *control plane* — what time is it, what's the weather, who decides, when does it change.

## The three clocks

| Clock | Authority | Use for |
|---|---|---|
| **Real time** (`os.time()` server-side) | Authoritative, single source of truth | Daily resets, scheduled events, ProcessReceipt timestamps, offline accrual |
| **Game time** (server state, broadcast to clients) | Server-owned, replicated | In-world calendar, NPC schedules, "shop opens at game-noon", aging crops |
| **`Lighting.ClockTime`** (0..24) | Server-set, replicated automatically | Visual sun position, sky color, shadow direction |

These are **independent**: game time can pause without affecting real time; ClockTime can be set to a fixed dramatic 7AM forever even while game time advances. Pick the right clock for the task.

## Game time — server-owned, replicated

The standard pattern: a single `IntValue` (or Attribute on a designated Folder) holds the current game-time-in-seconds. The server advances it every Heartbeat. Clients read and convert to display.

```lua
--!strict
local RunService = game:GetService("RunService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local SECONDS_PER_GAME_DAY = 24 * 60       -- 1 real-minute = 1 game-hour (so 24 real-min = 1 game-day)

-- Replicated state
local clockFolder = Instance.new("Folder", ReplicatedStorage)
clockFolder.Name = "GameClock"
clockFolder:SetAttribute("GameTimeSeconds", 0)
clockFolder:SetAttribute("Paused", false)

local GameTime = {}

function GameTime.GetSeconds(): number
    return clockFolder:GetAttribute("GameTimeSeconds") :: number
end

function GameTime.GetHour(): number
    return (GameTime.GetSeconds() / SECONDS_PER_GAME_DAY * 24) % 24
end

function GameTime.GetDay(): number
    return math.floor(GameTime.GetSeconds() / SECONDS_PER_GAME_DAY)
end

function GameTime.SetPaused(paused: boolean)
    if RunService:IsServer() then
        clockFolder:SetAttribute("Paused", paused)
    end
end

-- Server only: tick the clock
if RunService:IsServer() then
    local realPerGameRatio = 24 * 60 * 60 / SECONDS_PER_GAME_DAY    -- game-seconds per real-second
    RunService.Heartbeat:Connect(function(dt)
        if clockFolder:GetAttribute("Paused") then return end
        local cur = clockFolder:GetAttribute("GameTimeSeconds") :: number
        clockFolder:SetAttribute("GameTimeSeconds", cur + dt * realPerGameRatio)
    end)
end

return GameTime
```

Clients see attribute changes via replication and can `:GetAttributeChangedSignal("GameTimeSeconds")` to react.

### Persisting game time across server restarts

Game time should survive shutdowns. Save it to a global DataStore key, read on server start, save periodically:

```lua
local store = game:GetService("DataStoreService"):GetDataStore("WorldState")
local KEY = "GameClock"

-- On server start
local ok, saved = pcall(function() return store:GetAsync(KEY) end)
if ok and saved then
    -- Advance by elapsed real-time since last save
    local elapsedReal = os.time() - (saved.savedAt or os.time())
    local elapsedGame = elapsedReal * (24 * 60 * 60 / SECONDS_PER_GAME_DAY)
    clockFolder:SetAttribute("GameTimeSeconds", (saved.gameSeconds or 0) + elapsedGame)
end

-- Save periodically + on BindToClose
local function save()
    pcall(function()
        store:UpdateAsync(KEY, function(_)
            return {gameSeconds = GameTime.GetSeconds(), savedAt = os.time()}, {}
        end)
    end)
end

task.spawn(function() while true do task.wait(60); save() end end)
game:BindToClose(save)
```

This way the in-world world-clock doesn't reset every server restart — players who log in 8 real hours later see 8 game-hours of in-world time have passed.

## Day/night cycle (visual, driven by game time)

Sync `Lighting.ClockTime` to the game-hour:

```lua
local Lighting = game:GetService("Lighting")
local RunService = game:GetService("RunService")

if RunService:IsServer() then
    RunService.Heartbeat:Connect(function()
        Lighting.ClockTime = GameTime.GetHour()
    end)
end
```

Or, for a fixed-real-time cycle independent of game time (simpler, no game-time clock needed):
```lua
local SECONDS_PER_FULL_DAY = 600    -- 10 real-minutes per cycle
RunService.Heartbeat:Connect(function(dt)
    Lighting.ClockTime = (Lighting.ClockTime + dt * 24 / SECONDS_PER_FULL_DAY) % 24
end)
```

`Lighting.ClockTime` replicates from server to all clients automatically — set it server-side once and every player sees the sun move.

For richer day/night (color changes, ambient swap at sunset, fog density at night), run a per-Heartbeat update that interpolates between curated "preset" Lighting/Atmosphere configs based on the current hour. See [lighting.md](lighting.md) for the post-effect surface and the tween-by-zone pattern.

## Phase / period helpers

```lua
local function isDay(hour: number): boolean
    return hour >= 6 and hour < 18
end

local function isNight(hour: number): boolean
    return not isDay(hour)
end

local function getPhase(hour: number): string
    if hour < 5 then return "Night"
    elseif hour < 8 then return "Dawn"
    elseif hour < 18 then return "Day"
    elseif hour < 20 then return "Dusk"
    else return "Night" end
end
```

NPC behavior, monster spawns, shop hours all key off `getPhase(GameTime.GetHour())`.

## Weather — state machine pattern

Weather is modal: at any moment the world is in **one** state (Clear / Cloudy / Rain / Storm / Snow / Fog), and transitions follow rules. Implement as a server-owned state machine that updates Lighting/Atmosphere, spawns/removes particle emitters, plays/stops ambient sound.

```lua
--!strict
local Lighting = game:GetService("Lighting")
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

export type WeatherState = "Clear" | "Cloudy" | "Rain" | "Storm" | "Snow" | "Fog"

local Weather = {}
local stateAttr = ReplicatedStorage:GetAttribute("Weather") or "Clear"

-- Visual presets: Atmosphere properties to tween toward for each state
local PRESETS: {[WeatherState]: {[string]: any}} = {
    Clear  = {Density = 0.3,  Color = Color3.fromRGB(199, 170, 107)},
    Cloudy = {Density = 0.45, Color = Color3.fromRGB(150, 150, 150)},
    Rain   = {Density = 0.6,  Color = Color3.fromRGB(100, 110, 130)},
    Storm  = {Density = 0.85, Color = Color3.fromRGB(60, 70, 90)},
    Snow   = {Density = 0.5,  Color = Color3.fromRGB(220, 230, 240)},
    Fog    = {Density = 0.95, Color = Color3.fromRGB(180, 190, 200)},
}

local atmosphere = Lighting:FindFirstChildOfClass("Atmosphere")     -- assume one exists
local rainEmitter, snowEmitter, thunderSound          -- pre-authored, see vfx.md / sound.md

function Weather.SetState(newState: WeatherState, transitionTime: number?)
    transitionTime = transitionTime or 5

    local preset = PRESETS[newState]
    if preset and atmosphere then
        TweenService:Create(atmosphere, TweenInfo.new(transitionTime), preset):Play()
    end

    -- Toggle particle emitters
    if rainEmitter then rainEmitter.Enabled = (newState == "Rain" or newState == "Storm") end
    if snowEmitter then snowEmitter.Enabled = (newState == "Snow") end

    -- Toggle ambient sound
    if thunderSound then
        if newState == "Storm" then thunderSound:Play() else thunderSound:Stop() end
    end

    ReplicatedStorage:SetAttribute("Weather", newState)
end

function Weather.GetState(): WeatherState
    return ReplicatedStorage:GetAttribute("Weather") :: WeatherState
end

return Weather
```

### Weather transitions (probabilistic)

Don't lurch between states; transition probabilistically every game-hour, biased toward staying in the current state:

```lua
local TRANSITIONS: {[WeatherState]: {[WeatherState]: number}} = {
    Clear  = {Clear = 0.7, Cloudy = 0.25, Fog = 0.05},
    Cloudy = {Cloudy = 0.5, Clear = 0.3, Rain = 0.2},
    Rain   = {Rain = 0.6, Cloudy = 0.3, Storm = 0.1},
    Storm  = {Storm = 0.4, Rain = 0.5, Cloudy = 0.1},
    Snow   = {Snow = 0.7, Cloudy = 0.3},
    Fog    = {Fog = 0.5, Clear = 0.5},
}

local function rollNext(current: WeatherState): WeatherState
    local roll = math.random()
    local accum = 0
    for state, prob in TRANSITIONS[current] do
        accum += prob
        if roll <= accum then return state end
    end
    return current
end

-- Tick once per game-hour (or every 5 game-minutes for finer control)
local lastHour = -1
RunService.Heartbeat:Connect(function()
    local hour = math.floor(GameTime.GetHour())
    if hour ~= lastHour then
        lastHour = hour
        Weather.SetState(rollNext(Weather.GetState()))
    end
end)
```

The transition matrix is the lever — biased to stay (high diagonals), and reachability rules the climate (e.g., Snow only available if `season == "Winter"`).

### Region-based weather

For maps with distinct biomes, Weather is per-region rather than global. Each region tag has its own state; the client picks which is "active" based on the player's position. Use [tags.md](tags.md) to mark regions and a per-Heartbeat overlap query to determine the player's region.

## Seasons (game-world calendar)

Seasons are a function of game-day. Define a year length and map days to seasons:

```lua
local DAYS_PER_SEASON = 30
local SEASONS = {"Spring", "Summer", "Autumn", "Winter"}

local function getSeason(): string
    local day = GameTime.GetDay()
    local seasonIndex = math.floor(day / DAYS_PER_SEASON) % #SEASONS + 1
    return SEASONS[seasonIndex]
end
```

Seasons gate weather pools (no Snow in Summer), NPC clothing changes, crop growth speeds, available quests, etc.

## Real-time scheduled events

Some things should fire at specific real-world UTC times — not game-time. Daily reset at 00:00 UTC, weekly raid at Saturday 18:00 UTC, holiday event during October.

```lua
local function scheduleDailyAt(utcHour: number, utcMinute: number, fn: () -> ())
    task.spawn(function()
        while true do
            local now = os.time()
            local t = os.date("!*t", now)
            local target = os.time({
                year = t.year, month = t.month, day = t.day,
                hour = utcHour, min = utcMinute, sec = 0,
            })
            if target <= now then target += 24 * 3600 end       -- next day
            task.wait(target - now)
            pcall(fn)
        end
    end)
end

scheduleDailyAt(0, 0, function() resetDailyQuests() end)        -- midnight UTC
scheduleDailyAt(18, 0, function() spawnDailyBoss() end)         -- 6PM UTC
```

For multi-server coordination (every server should fire the event simultaneously), pair with [messaging-http.md](messaging-http.md) MessagingService — one server is elected as the "fire trigger" and announces; others listen.

For weekly events, check `os.date("!*t", os.time()).wday` (1=Sunday, 7=Saturday). For monthly, `.day`. For "October only," `.month`.

## Holiday / event windows

```lua
local function isHolidayActive(): boolean
    local t = os.date("!*t", os.time())
    return t.month == 10 and t.day >= 25 and t.day <= 31         -- last week of October
end

if isHolidayActive() then
    Weather.SetState("Fog")
    -- Apply visual overrides, spawn special NPCs, enable holiday shop, etc.
end
```

## Pitfalls

- **Confusing `Lighting.ClockTime` with game time.** ClockTime is purely visual (drives sun position). Game logic shouldn't read it; read your own GameTime.
- **Per-frame `Lighting.ClockTime` from the client.** Set once on the server; replication handles the rest. Per-frame updates from many clients race.
- **Game-time drift from real-time.** If `Heartbeat` ticks differ across servers, game time diverges. For a single canonical world clock across all servers, persist to DataStore (or a global MemoryStore HashMap) and authoritatively update from a designated coordinator server — see [live-ops.md](live-ops.md).
- **Forgetting to pause game time when the world should be paused.** A "paused" pause-menu shouldn't advance the in-world clock for that player only — game time is global. Use UI freezing instead.
- **Weather changes that snap rather than tween.** Atmosphere/Lighting changes look jarring without tweening. Always pair `Weather.SetState` with a TweenService transition.
- **Particle emitters not parented to the player's view.** A global rain emitter at world origin only rains around the origin — players in distant areas see no rain. For convincing global rain, use a client-side emitter parented to the camera or the player's character.
- **Weather sound on every client unmodulated by region.** Ambient rain sound playing at full volume regardless of whether the player is indoors. Pair with a tag/region check.
- **`scheduleDailyAt` losing its slot if the script restarts.** The `task.spawn` loop dies on script reload; ensure it's started from a long-lived script.
- **Computing season from real-time instead of game-time** — players who logged in across a real-month boundary see seasons shift mid-session arbitrarily. Anchor seasons to game-day for in-world consistency, or to real-week if it's a real-world calendar event.
- **Multiple servers each rolling their own weather independently** — players in two servers see different weather, breaks the shared-world illusion. For a unified game-world weather, designate a coordinator (lowest JobId, or via MemoryStore lock) and broadcast via MessagingService.

## See also
- [lighting.md](lighting.md) — `Lighting.ClockTime`, `Atmosphere`, post-effects (the visual layer this card drives)
- [areas-and-biomes.md](areas-and-biomes.md) — per-area weather state and the canonical region-detection layer
- [vfx.md](vfx.md) — ParticleEmitter for rain/snow/leaves
- [sound.md](sound.md) — ambient weather audio (3D positional via `SoundService` or per-camera)
- [tags.md](tags.md) — region tagging for biome / per-region weather
- [messaging-http.md](messaging-http.md) — MessagingService for cross-server time/weather sync
- [live-ops.md](live-ops.md) — multi-server coordination of scheduled events
- [datastores.md](datastores.md) — persisting game time across server restarts
- [task-scheduling.md](task-scheduling.md) — `Heartbeat` and `task.spawn` for time loops
