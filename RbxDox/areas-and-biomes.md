# Areas, Biomes & Levels

> Source: distilled from open-world biome patterns, obby-stage patterns, and zoned-RPG patterns

## Purpose
Almost every Roblox game subdivides its world: an obby has stages, an open-world RPG has biomes (forest / desert / dungeon), a tycoon has owned plots, a horror map has rooms. This card covers **areas as a structural concept** — defining the bounded volumes, detecting which area the player is in, swapping per-area state (music, lighting, weather, enemy pool, spawn rules), entry/exit transitions, discrete stage/level systems for linear games, and the decision between organising several areas in one place versus splitting into separate places via TeleportService. Per-system specifics (music = `sound.md`, lighting = `lighting.md`, weather = `time-and-weather.md`, NPCs = `npcs.md`) live in those cards; this card is the composition layer.

## Three flavors — pick the right shape

| Shape | When to use | Example | Reference |
|---|---|---|---|
| **Continuous biomes** | Open-world; smooth transitions; large explorable map | RPG with forest → desert → tundra; survival biomes | This card |
| **Discrete stages / levels** | Linear progression; reset on death; clear "you're on level 3" | Obby; level-based puzzle game; tower-defense map slots | This card |
| **Separate places** (TeleportService) | Genuinely independent gameplay; large enough to need its own server | Hub world + minigame places; lobby + match-server places | [teleport.md](teleport.md) |

You can mix all three in one experience: a hub place with continuous biomes, plus separate match places teleported to.

The dividing question for "should this be a separate place or a same-place area?":
- **Same place** if the player walks/teleports between areas frequently and they're not too large.
- **Separate place** if it has fundamentally different gameplay rules, OR if streaming all of it together is too heavy, OR if you want a fresh server per session.

## Defining an area

Four common patterns, in increasing complexity:

### 1. Tagged-part bounding volume (simplest)
A single anchored Part (often invisible) marks the area. Tag it; the part's CFrame and Size define the bounds.

```lua
-- In Studio: place a Part where the area is, scale it to cover the area, set Transparency = 1, CanCollide = false, CanQuery = false
-- Tag it "Area" and add Attributes for metadata
areaPart:AddTag("Area")
areaPart:SetAttribute("AreaId", "ForestGlade")
areaPart:SetAttribute("DisplayName", "Forest Glade")
areaPart:SetAttribute("MusicId", "rbxassetid://...")
areaPart:SetAttribute("LightingPreset", "ForestDay")
```

Pros: dead simple, easy for designers to author. Cons: rectangular only; overlapping bounds are ambiguous.

### 2. Region3 (computed bounding box)
A `Region3` is a math primitive (not an Instance). Useful for code-defined areas:

```lua
local region = Region3.new(Vector3.new(-50, 0, -50), Vector3.new(50, 100, 50))
```

Spatial-query API: `workspace:FindPartsInRegion3(...)` (deprecated; prefer `:GetPartBoundsInBox`). For a *check* of "is this point inside this region", point-in-box test:

```lua
local function pointInRegion(pos: Vector3, region: Region3): boolean
    local min, max = region.CFrame.Position - region.Size/2, region.CFrame.Position + region.Size/2
    return pos.X >= min.X and pos.X <= max.X
       and pos.Y >= min.Y and pos.Y <= max.Y
       and pos.Z >= min.Z and pos.Z <= max.Z
end
```

### 3. Polygon / non-rectangular bounds
For irregularly shaped areas (a winding river valley), use multiple tagged parts as **a set of bounding boxes**, and consider the player "in the area" if they're inside ANY of them. UI (minimap) draws an outline using the union.

For genuinely concave shapes, store a polygon (list of Vector3 points) per area and do a 2D point-in-polygon test on the X/Z plane:

```lua
local function pointInPolygon2D(p: Vector3, polygon: {Vector3}): boolean
    local inside = false
    local j = #polygon
    for i = 1, #polygon do
        local pi, pj = polygon[i], polygon[j]
        if ((pi.Z > p.Z) ~= (pj.Z > p.Z))
           and (p.X < (pj.X - pi.X) * (p.Z - pi.Z) / (pj.Z - pi.Z) + pi.X) then
            inside = not inside
        end
        j = i
    end
    return inside
end
```

Niche; only worth it for unusual layouts.

### 4. Model-based area (whole-map subdivision)
Each area is a `Model` (e.g., `workspace.Areas.Forest`); membership is "is this player's HumanoidRootPart inside the Model's bounding box":

```lua
local function findCurrentArea(pos: Vector3): Model?
    for _, area in workspace.Areas:GetChildren() do
        if area:IsA("Model") then
            local cf, size = area:GetBoundingBox()
            if pointInBox(pos, cf, size) then return area end
        end
    end
    return nil
end
```

Combines well with the build-then-parent pattern from [workspace-hierarchy.md](workspace-hierarchy.md) — each area is its own Model you can `:Clone`, hide, or stream independently.

## Region detection — which area is the player in?

For 1–2 dozen areas, the **per-Heartbeat O(N) check** is fine:

```lua
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")
local CollectionService = game:GetService("CollectionService")

local currentArea: {[Player]: string?} = {}

local function findAreaContaining(pos: Vector3): BasePart?
    for _, areaPart in CollectionService:GetTagged("Area") do
        if pointInPart(pos, areaPart) then return areaPart end
    end
    return nil
end

local function pointInPart(pos: Vector3, part: BasePart): boolean
    local local_ = part.CFrame:PointToObjectSpace(pos)
    return math.abs(local_.X) <= part.Size.X/2
       and math.abs(local_.Y) <= part.Size.Y/2
       and math.abs(local_.Z) <= part.Size.Z/2
end

RunService.Heartbeat:Connect(function()
    for _, p in Players:GetPlayers() do
        local hrp = p.Character and p.Character:FindFirstChild("HumanoidRootPart")
        if not hrp then continue end

        local area = findAreaContaining(hrp.Position)
        local newId = area and area:GetAttribute("AreaId")
        if newId ~= currentArea[p] then
            local oldId = currentArea[p]
            currentArea[p] = newId
            onAreaChanged(p, oldId, newId)
        end
    end
end)
```

**For 100+ areas or 50+ players** the O(N×P) per-frame cost shows up. Optimisations:
- **Spatial hash**: bucket areas by world-grid cell; only check the bucket the player is in.
- **Throttle**: check every 0.25s instead of every Heartbeat (areas don't change that fast).
- **Trigger-driven** (preferred for many areas): use `Touched`/`TouchEnded` on the area part itself, with `CanTouch = true` and player character contact. Fires only on transition. Couple with `OverlapParams` for accuracy.

```lua
-- Trigger-driven (no per-frame loop)
local function setupArea(areaPart: BasePart)
    areaPart.CanTouch = true
    areaPart.CanCollide = false
    areaPart.CanQuery = false
    areaPart.Transparency = 1

    areaPart.Touched:Connect(function(hit)
        local p = Players:GetPlayerFromCharacter(hit.Parent)
        if not p then return end
        if currentArea[p] == areaPart:GetAttribute("AreaId") then return end
        local oldId = currentArea[p]
        currentArea[p] = areaPart:GetAttribute("AreaId")
        onAreaChanged(p, oldId, currentArea[p])
    end)

    areaPart.TouchEnded:Connect(function(hit)
        local p = Players:GetPlayerFromCharacter(hit.Parent)
        if not p then return end
        -- Don't reset on TouchEnded — player may have entered another area
        -- Let the new area's Touched overwrite. If they walked off the map, they get caught by the world-bounds kill-volume.
    end)
end

for _, p in CollectionService:GetTagged("Area") do setupArea(p) end
CollectionService:GetInstanceAddedSignal("Area"):Connect(setupArea)
```

**Overlap caveat**: areas should not overlap in the same Y range. If they do, define priority (`AreaPriority` attribute) and pick the highest in `findAreaContaining`.

## Per-area state — the central pattern

Each area has a "preset" of how the world should look/sound when you're in it. The handler runs the swap on transition:

```lua
type AreaPreset = {
    musicId: string?,
    lightingPreset: string?,        -- key into a Lighting/Atmosphere config table
    weatherState: WeatherState?,    -- forces a weather state
    spawnPool: {string}?,           -- ids of spawn points to use for respawn
    enemyPool: {string}?,           -- enemy ids that can spawn here
    ambientSoundId: string?,
    skyboxId: string?,
    cameraOverride: CFrame?,        -- for fixed-camera areas
    pvpEnabled: boolean?,
}

local PRESETS: {[string]: AreaPreset} = {
    ForestGlade = {
        musicId = "rbxassetid://forest_theme",
        lightingPreset = "ForestDay",
        weatherState = "Clear",
        ambientSoundId = "rbxassetid://birds_loop",
        enemyPool = {"Wolf", "Boar"},
        pvpEnabled = false,
    },
    DesertWastes = {
        musicId = "rbxassetid://desert_theme",
        lightingPreset = "DesertHarsh",
        weatherState = "Clear",
        ambientSoundId = "rbxassetid://wind_loop",
        enemyPool = {"Scorpion", "SandWyrm"},
    },
    PvpArena = {
        musicId = "rbxassetid://battle_theme",
        lightingPreset = "Stormy",
        pvpEnabled = true,
    },
}

local function applyPreset(player: Player, preset: AreaPreset?)
    if not preset then return end

    -- Music swap (client-side; server fires the change)
    AreaMusicRemote:FireClient(player, preset.musicId)

    -- Lighting (client-side; could be global if all players are in the area)
    AreaLightingRemote:FireClient(player, preset.lightingPreset)

    -- Server-side state
    player:SetAttribute("AreaPvpEnabled", preset.pvpEnabled or false)
    player:SetAttribute("AreaEnemyPool", preset.enemyPool and table.concat(preset.enemyPool, ",") or "")
end

local function onAreaChanged(player, oldId, newId)
    local preset = newId and PRESETS[newId]
    applyPreset(player, preset)

    AreaChangedRemote:FireClient(player, {oldId = oldId, newId = newId})
end
```

The client receives the preset id and applies music/lighting locally (per-player UX). Server-side game state (PvP enabled, enemy spawning pool) lives in player attributes the rest of the systems read.

For state shared by everyone in the area (a global "boss event" toggle), the preset effect is applied server-wide once.

## Entry / exit transitions

The crude "music snaps" feels jarring. Polish the transition:

```lua
-- Client-side
AreaMusicRemote.OnClientEvent:Connect(function(newMusicId)
    if currentMusic and currentMusic.SoundId == newMusicId then return end
    -- Fade out current
    if currentMusic then
        TweenService:Create(currentMusic, TweenInfo.new(1.5), {Volume = 0}):Play()
        local old = currentMusic
        task.delay(1.5, function() old:Stop(); old:Destroy() end)
    end
    -- Fade in new
    if newMusicId and newMusicId ~= "" then
        local s = Instance.new("Sound")
        s.SoundId = newMusicId; s.Looped = true; s.Volume = 0
        s.Parent = SoundService
        s:Play()
        TweenService:Create(s, TweenInfo.new(1.5), {Volume = 0.4}):Play()
        currentMusic = s
    else
        currentMusic = nil
    end
end)

AreaLightingRemote.OnClientEvent:Connect(function(presetId)
    if not presetId then return end
    local preset = LIGHTING_PRESETS[presetId]
    -- Tween Atmosphere properties toward preset (see lighting.md)
    TweenService:Create(atmosphere, TweenInfo.new(2), preset.atmosphere):Play()
    TweenService:Create(colorCorrection, TweenInfo.new(2), preset.colorCorrection):Play()
end)

AreaChangedRemote.OnClientEvent:Connect(function(data)
    if data.newId then
        local displayName = AREA_DISPLAY_NAMES[data.newId] or data.newId
        showAreaBanner(displayName)            -- "Forest Glade" banner fades in for 3s
    end
end)
```

For especially dramatic transitions (entering a dungeon, boss arena), add a screen-fade-to-black during the swap. See [tweens-animation.md](tweens-animation.md) and [menu-systems.md](menu-systems.md).

## Discrete stages / levels (obby, linear progression)

For games where the player progresses through numbered stages (Stage 1, Stage 2, ..., Stage N), each stage is its own area but with extra structure: a **stage index** the player is currently at, the ability to **skip back/forward**, and **respawning at the highest-reached stage**.

### Profile / leaderstat shape
```lua
profile.currentStage = 1
profile.maxStageReached = 1
-- Mirror to leaderstats so the scoreboard shows it
stats.Stage.Value = profile.currentStage
```

### Stage data
Each stage is a Model under `workspace.Stages` containing geometry, a SpawnLocation, a "finish" trigger:

```
workspace.Stages
├── Stage1 (Model)
│   ├── ... geometry ...
│   ├── SpawnLocation (anchored Part, tagged "StageSpawn", AreaId = "Stage1")
│   └── Finish (Part, tagged "StageFinish", attribute NextStage = "Stage2")
├── Stage2
│   ├── ...
└── Stage3
```

### Reaching the next stage
```lua
-- Server Script: handle every Touched on parts tagged "StageFinish"
for _, finish in CollectionService:GetTagged("StageFinish") do
    finish.Touched:Connect(function(hit)
        local p = Players:GetPlayerFromCharacter(hit.Parent)
        if not p then return end
        local profile = PlayerData.Get(p); if not profile then return end

        local nextStage = finish:GetAttribute("NextStage")
        if not nextStage then return end

        local nextNum = tonumber(nextStage:match("%d+"))
        if not nextNum or nextNum <= profile.currentStage then return end    -- already passed

        profile.currentStage = nextNum
        profile.maxStageReached = math.max(profile.maxStageReached, nextNum)

        -- Update leaderstats display
        p.leaderstats.Stage.Value = profile.currentStage

        -- Move them to the next stage's spawn (or let them walk forward — designer choice)
        teleportToStageSpawn(p, nextStage)
    end)
end
```

### Respawn at current stage
Override `Player.RespawnLocation` to point at the current stage's SpawnLocation:

```lua
local function getStageSpawn(stageNum: number): SpawnLocation?
    local stage = workspace.Stages:FindFirstChild("Stage" .. stageNum)
    return stage and stage:FindFirstChildOfClass("SpawnLocation")
end

p.CharacterAdded:Connect(function()
    p.RespawnLocation = getStageSpawn(profile.currentStage)
end)
```

See [spawn-mechanics.md](spawn-mechanics.md) for the broader spawn flow.

### Skip-stage (gamepass)
A common monetization: VIP can skip ahead. Validate gamepass (`monetization.md`), then advance the stage index server-side.

### Stage UI
- Top of screen: "Stage 12 / 50"
- Mini-map / progress bar showing how far through the obby
- Leaderstats column "Stage" via `leaderstats.md`

## When to use separate places (TeleportService) vs same-place areas

Same-place areas are simpler. Reach for separate places when:

| Symptom | Reason |
|---|---|
| Total parts > ~40K with everything loaded | Streaming helps but eventually splitting is cheaper |
| Each area has fundamentally different rules (PvP / PvE / building / racing) | Cleaner separation |
| Lobby + match pattern (matchmaking → reserved server) | Each match needs its own server state |
| Player counts vary wildly per area | Different server sizes |
| Memory / CPU budget exceeded | Force-split to keep frame time |

Once split, use [teleport.md](teleport.md) for the handoff. Cross-place data passes via `TeleportData` (small payloads only — validate at the target).

## Streaming for large worlds

If the whole map is one place, enable Streaming so the client only loads what's near the player:

```lua
workspace.StreamingEnabled = true
workspace.StreamingMinRadius = 64
workspace.StreamingTargetRadius = 256
```

Per-Model granularity: set `Model.ModelStreamingMode` on each area Model to control whether it streams as a unit. See [performance.md](performance.md) for the full discussion.

**Streaming changes the area-detection rules**: on the client, instances may not exist yet. Always `:WaitForChild` for area-tagged parts on the client. Most area logic is server-side, where the full DataModel is always present.

## Boundaries — kill volumes and invisible walls

To prevent the player leaving the world (or an area):

### Kill volume (the void)
A large invisible Part below the map; Touched → `humanoid:TakeDamage(humanoid.MaxHealth)`. See [world-mechanics.md](world-mechanics.md).

### Invisible walls
Anchored, `Transparency = 1`, `CanCollide = true`, `CanQuery = true` — physically blocks movement. Tag them so designers can find them later.

### Area-locked content
"You cannot enter the Dungeon yet" — the dungeon's entry has a ProximityPrompt (or a simple Touched check) that gates access by quest progress, level, or item:

```lua
dungeonEntry.Touched:Connect(function(hit)
    local p = Players:GetPlayerFromCharacter(hit.Parent)
    if not p then return end
    local profile = PlayerData.Get(p); if not profile then return end

    if profile.level < 10 then
        notifyPlayer(p, "Level 10 required.")
        bouncePlayerBack(p)
        return
    end
    -- Otherwise let them through (or actively teleport)
end)
```

Or use a literal locked door — see [interactions.md](interactions.md).

## Map / minimap integration

Once areas exist as discrete entities, surface them in the UI:
- **World map** menu (ESC → Map): show all areas as colored zones with their display name. Visited areas in full color; unvisited in gray.
- **Minimap** (corner HUD): zoom in on the area immediately around the player; label the area name at top.
- **Area-banner** on entry: large temporary text showing the area's name, fades after 3s. Common in RPGs (Skyrim style).

For map rendering, the area Models contain the bounds; convert to 2D screen positions with a fixed camera-from-above projection.

## Area-restricted content

Filter what spawns / drops / appears based on the current area. The pattern is the same across systems:

```lua
local function filterByArea(pool: {string}, areaId: string?): {string}
    if not areaId then return pool end
    local preset = PRESETS[areaId]
    if not preset or not preset.enemyPool then return pool end
    -- Intersect: only return items in BOTH the global pool and the area's allowed pool
    local allowed = {}; for _, e in preset.enemyPool do allowed[e] = true end
    local result = {}
    for _, e in pool do if allowed[e] then table.insert(result, e) end end
    return result
end
```

Apply when spawning enemies (filter [npcs.md](npcs.md) by area), rolling loot (filter inventory drops by area), offering quests (filter [quests.md](quests.md) by giver location).

## Discovery / fog-of-war

Track which areas the player has visited:

```lua
profile.visitedAreas = {}     -- {[areaId] = firstVisitedAt}

local function onAreaChanged(p, oldId, newId)
    local profile = PlayerData.Get(p); if not profile or not newId then return end
    if not profile.visitedAreas[newId] then
        profile.visitedAreas[newId] = os.time()
        -- Grant exploration XP / achievement
        Progression.GrantXp(profile, "Combat", DISCOVERY_XP)
        notifyPlayer(p, "Discovered: " .. (PRESETS[newId].displayName or newId))
    end
end
```

The world-map UI uses `visitedAreas` to render unvisited zones as gray / blank. Pair with [progression.md](progression.md) for first-discovery XP rewards.

## Patterns

### Per-area background music
Already shown above — fade-out current → fade-in new on area change. SoundService-parented Sounds, not part-parented (we want global volume to the player, not 3D).

### Boss arena lockdown
On entering a boss area, lock the entry door behind the player; unlock when the boss dies or the player dies:

```lua
local function lockArena(player: Player)
    arenaDoor.CanCollide = true
    arenaDoor.Transparency = 0.3
    -- Remove the lock when boss dies or player leaves area (via Died handler / area-change)
end
```

### Dynamic area spawning (procgen)
For procedurally generated dungeons, instantiate areas at runtime:
```lua
local areaModel = TEMPLATES.DungeonRoom:Clone()
areaModel:SetAttribute("AreaId", "Dungeon_" .. roomId)
areaModel:PivotTo(targetCFrame)
areaModel.Parent = workspace.Areas

-- Find the bounding part and tag it
local bounds = areaModel:FindFirstChild("AreaBounds")
if bounds then bounds:AddTag("Area") end
```

### "Last safe area" — for revival
When the player dies, respawn at the last safe area's spawn point rather than the world default. See [spawn-mechanics.md](spawn-mechanics.md) "last-known-position revival" for the per-Heartbeat tracking.

```lua
local function onAreaChanged(p, oldId, newId)
    local profile = PlayerData.Get(p); if not profile or not newId then return end
    if not PRESETS[newId].pvpEnabled then    -- safe area
        profile.lastSafeAreaId = newId
    end
end
```

### Area-locked teleport pad
A teleport pad that only works when its destination area is "discovered":
```lua
teleportPad.Touched:Connect(function(hit)
    local p = Players:GetPlayerFromCharacter(hit.Parent)
    local destinationArea = teleportPad:GetAttribute("DestinationAreaId")
    if not p or not profile.visitedAreas[destinationArea] then
        notifyPlayer(p, "You haven't discovered this place yet.")
        return
    end
    teleportToArea(p, destinationArea)
end)
```

## Pitfalls

- **Areas overlap in the same Y range without priority.** Player is "in" Forest AND Dungeon simultaneously, music alternates every Heartbeat. Use `AreaPriority` attribute and take the highest.
- **Per-Heartbeat full scan with hundreds of areas.** Profile shows the area-detection loop dominating. Switch to trigger-driven `Touched` or spatial-hash bucketing.
- **`Touched` on the area part fires for non-character parts.** Filter to character/Humanoid via `Players:GetPlayerFromCharacter`.
- **Music re-starts every time the player enters the same area.** Cache the current music id; only swap if it changed.
- **Lighting tween conflict.** Two area-change events fire close together; one tween cancels the other halfway. Use a short delay before applying, OR stop the previous tween before starting the new one.
- **Area state stored client-side and exploited.** "I'm in PvE area, immune to damage" — server must own area state per player. Don't trust the client's claim of which area it's in.
- **Discrete-stage SkipStage exploit.** Client claims "I finished Stage 5" → leaderstats reflect Stage 5. Server must validate the touched Finish part matched a stage > current.
- **Stages defined by Touched but the geometry traps the player.** Player walks back into Stage 1 from Stage 5; the Stage 1 finish fires and resets them. Use `>= currentStage + 1` check or one-way trigger volumes.
- **`StreamingEnabled` + area parts.** When chunks unload, area-tagged parts vanish from the client; client-side area detection breaks. Keep area logic server-side; push the AreaId to the client.
- **Separate-place migration loses player state.** Splitting into multiple places means TeleportData is the only carry-over (≤30KB, JSON). For richer state, persist in DataStore before teleport.
- **Discovery progress not saved per slot.** In multi-slot games (`save-slots.md`), each slot has its own visited-areas table.
- **Map UI rendering all area Models on every Heartbeat.** Cache the projected positions; recompute only when the camera or map zoom changes.
- **Fade-to-black transition with no recovery.** If the lighting/music tween fails, the screen stays dark. Always fire the un-fade after a max-timeout, even if the destination signal didn't arrive.
- **Boss arena lockdown that traps the player after the boss is dead.** Always unlock on boss death AND on the player's death AND on area-leave. Three triggers cheaper than one missed trigger.

## See also
- [workspace-hierarchy.md](workspace-hierarchy.md) — organizing area Models in Workspace
- [tags.md](tags.md), [attributes.md](attributes.md) — designer-driven area metadata
- [lighting.md](lighting.md) — per-area lighting / atmosphere presets
- [time-and-weather.md](time-and-weather.md) — region-based weather is the same pattern
- [sound.md](sound.md) — per-area music swap with PlayerGui or SoundService parenting
- [npcs.md](npcs.md) — area-restricted enemy spawning
- [enemy-spawning.md](enemy-spawning.md) — spawners filter by area; per-area enemy pools narrow the choice
- [enemy-difficulty.md](enemy-difficulty.md) — per-area default difficulty
- [quests.md](quests.md) — quest-givers per area
- [spawn-mechanics.md](spawn-mechanics.md) — per-area spawn pools, last-safe-area respawn
- [world-mechanics.md](world-mechanics.md) — kill volumes, SpawnLocation, region triggers
- [performance.md](performance.md) — Streaming for large worlds
- [teleport.md](teleport.md) — when to split into separate places instead
- [progression.md](progression.md) — discovery XP rewards
- [leaderstats.md](leaderstats.md) — Stage column for obby-style games
- [save-slots.md](save-slots.md) — per-slot visited areas
- [security.md](security.md) — server-authoritative area state
- [menu-systems.md](menu-systems.md) — world map UI, area-banner on entry
- [interactions.md](interactions.md) — area-locked doors via ProximityPrompt
