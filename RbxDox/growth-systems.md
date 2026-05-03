# Growth Systems (Farming, Planting, Crops)

> Source: distilled from farming/sandbox/idle-garden patterns (Stardew, FarmVille, gardening sims)

## Purpose
A "plant a seed → it grows over time → harvest" system is its own subgenre and a clean composition of building blocks already covered: state-machine ticking ([world-mechanics.md](world-mechanics.md), [oop-patterns.md](oop-patterns.md)), L-system geometry ([procedural-construction.md](procedural-construction.md)), inventory grants ([inventory.md](inventory.md)), interaction prompts ([interactions.md](interactions.md)), persistence ([datastores.md](datastores.md)), and offline accrual ([offline-processing.md](offline-processing.md)). This card pulls them into a single working system: per-plot growth state, stage-by-stage visual transitions using the L-system pattern, watering, harvest, persistence, offline catch-up.

## Architecture overview

```
Plot (BasePart, tagged "Plot")
├── Attributes: SeedType, PlantedAt, GrowthStage, WateredUntil
├── ProximityPrompt — interaction (plant / water / harvest)
└── PlantVisual (Model, generated per growth stage; replaced on stage-up)

Server holds a per-plot table mirroring the attributes (for fast access).
DataStore persists per-player plot state (or per-plot if shared).
```

## Plant catalog

A static catalog defining every crop variety: stages, durations, harvest yield, plant geometry per stage. Lives in `ReplicatedStorage`:

```lua
--!strict
export type GrowthStage = {
    name: string,                                          -- "Sprout", "Young", "Mature", "Fruiting"
    durationSec: number,                                   -- time in this stage before advancing
    buildFn: (plot: BasePart, plant: PlantInstance) -> Model,    -- generates the visual
}

export type PlantDef = {
    id: string,
    name: string,
    seedItemId: string,                                    -- inventory item required to plant
    harvestItemId: string,                                 -- inventory item granted on harvest
    harvestYield: NumberRange,                             -- random per harvest (e.g., 1-3 apples)
    stages: {GrowthStage},                                 -- ordered; index 1 = freshly planted
    multiHarvest: boolean?,                                -- if true, regrow from late stage rather than reset
    iconId: string?,
}

local Plants = {}

Plants.Apple = {
    id = "Apple", name = "Apple Tree",
    seedItemId = "AppleSeed",
    harvestItemId = "Apple",
    harvestYield = NumberRange.new(2, 5),
    stages = {
        {name = "Seed",     durationSec = 30,  buildFn = buildSeedVisual},
        {name = "Sprout",   durationSec = 60,  buildFn = buildSproutVisual},
        {name = "Sapling",  durationSec = 180, buildFn = buildSaplingVisual},
        {name = "Mature",   durationSec = 360, buildFn = buildMatureTree},
        {name = "Fruiting", durationSec = math.huge, buildFn = buildFruitingTree},   -- final
    },
    multiHarvest = true,        -- after harvest, drops back to "Mature" and re-fruits
}

Plants.Carrot = {
    id = "Carrot", name = "Carrot",
    seedItemId = "CarrotSeed",
    harvestItemId = "Carrot",
    harvestYield = NumberRange.new(1, 2),
    stages = {
        {name = "Seed",   durationSec = 30,  buildFn = buildSeedVisual},
        {name = "Sprout", durationSec = 60,  buildFn = buildCarrotTopVisual},
        {name = "Mature", durationSec = math.huge, buildFn = buildCarrotMature},
    },
    multiHarvest = false,        -- harvest = uproot; plot returns to empty
}

return Plants
```

## Stage builders — using the L-system pattern

Each stage's `buildFn` returns a Model. For the apple tree, depth scales with maturity (using the L-system from [procedural-construction.md](procedural-construction.md)):

```lua
-- ServerStorage/Growth/StageBuilders.lua

local function buildSeedVisual(plot: BasePart): Model
    local m = Instance.new("Model"); m.Name = "PlantVisual"
    local p = Instance.new("Part")
    p.Anchored = true; p.CanCollide = false
    p.Size = Vector3.new(0.4, 0.2, 0.4)
    p.Material = Enum.Material.Ground
    p.BrickColor = BrickColor.new("Brown")
    p.CFrame = plot.CFrame * CFrame.new(0, 0.6, 0)
    p.Parent = m
    return m
end

local function buildSproutVisual(plot: BasePart): Model
    local m = Instance.new("Model"); m.Name = "PlantVisual"
    local stem = Instance.new("Part")
    stem.Anchored = true; stem.CanCollide = false
    stem.Size = Vector3.new(0.2, 0.8, 0.2)
    stem.Material = Enum.Material.Grass
    stem.BrickColor = BrickColor.new("Earth green")
    stem.CFrame = plot.CFrame * CFrame.new(0, 0.9, 0)
    stem.Parent = m
    return m
end

local function buildSaplingVisual(plot: BasePart): Model
    -- depth = 1 L-system: one branch with two children + leaves
    return buildTree(plot.CFrame * CFrame.new(0, 0.5, 0), {
        depth = 1,
        baseLength = 2,
        baseThickness = 0.4,
        seed = plot:GetAttribute("RandomSeed") or 0,
    })
end

local function buildMatureTree(plot: BasePart): Model
    return buildTree(plot.CFrame * CFrame.new(0, 0.5, 0), {
        depth = 3,
        baseLength = 4,
        baseThickness = 0.8,
        seed = plot:GetAttribute("RandomSeed") or 0,
    })
end

local function buildFruitingTree(plot: BasePart): Model
    local tree = buildMatureTree(plot)

    -- Find leaf parts (terminal balls in the L-system); add fruit nearby
    for _, descendant in tree:GetDescendants() do
        if descendant:IsA("BasePart") and descendant.Shape == Enum.PartType.Ball
           and descendant.BrickColor == BrickColor.new("Forest green") then
            -- This is a leaf; add a fruit
            local fruit = Instance.new("Part")
            fruit.Shape = Enum.PartType.Ball
            fruit.Anchored = true; fruit.CanCollide = false
            fruit.Size = Vector3.new(0.5, 0.5, 0.5)
            fruit.Material = Enum.Material.Plastic
            fruit.BrickColor = BrickColor.new("Bright red")
            fruit.CFrame = descendant.CFrame * CFrame.new(0, -descendant.Size.Y/2 - 0.3, 0)
            fruit.Parent = tree
        end
    end
    return tree
end

return {
    buildSeedVisual = buildSeedVisual,
    buildSproutVisual = buildSproutVisual,
    buildSaplingVisual = buildSaplingVisual,
    buildMatureTree = buildMatureTree,
    buildFruitingTree = buildFruitingTree,
}
```

The `buildTree` function is the L-system from [procedural-construction.md](procedural-construction.md), parameterized to take a `seed` so the same plot generates the same tree shape every time (deterministic across server restarts).

For deterministic randomness:
```lua
local function buildTree(origin: CFrame, opts: {depth, baseLength, baseThickness, seed})
    local rng = Random.new(opts.seed or 0)
    -- Use rng:NextNumber()/NextInteger() instead of math.random for branch angles
    -- ...
end
```

## Plot data shape

Each plot is a tagged BasePart (designer drops them in the world). State lives in attributes (replicates to clients automatically) plus an in-memory mirror for fast tick access.

```lua
-- Plot attributes (designer-set):
plot:SetAttribute("PlotId", "Plot_001")
plot:SetAttribute("OwnerUserId", 0)             -- 0 = unowned (shared/public)

-- Plot attributes (runtime):
plot:SetAttribute("SeedType", "Apple")          -- nil = empty
plot:SetAttribute("PlantedAt", os.time())
plot:SetAttribute("StageIndex", 1)
plot:SetAttribute("WateredUntil", 0)            -- timestamp; growth runs at 1.5x while watered
plot:SetAttribute("RandomSeed", 0)              -- for deterministic L-system
```

## Growth ticker — server-side

One Heartbeat-driven sweep ticks every plot. Throttled to 1Hz (plants don't change second-to-second).

```lua
local Plants = require(game.ReplicatedStorage.Plants)
local CollectionService = game:GetService("CollectionService")
local RunService = game:GetService("RunService")

local lastTick = 0

local function effectiveElapsed(plot: BasePart): number
    local plantedAt = plot:GetAttribute("PlantedAt") or 0
    local now = os.time()
    local wateredUntil = plot:GetAttribute("WateredUntil") or 0

    -- Watered intervals grow at 1.5x; non-watered at 1x
    -- For simplicity: integrate from plantedAt to now, splitting at wateredUntil
    if wateredUntil <= plantedAt then
        return now - plantedAt
    elseif wateredUntil >= now then
        return (now - plantedAt) * 1.5
    else
        return (wateredUntil - plantedAt) * 1.5 + (now - wateredUntil)
    end
end

local function computeStageIndex(seedType: string, elapsed: number): number
    local def = Plants[seedType]; if not def then return 1 end
    local accum = 0
    for i, stage in def.stages do
        accum += stage.durationSec
        if elapsed < accum then return i end
    end
    return #def.stages
end

local function rebuildVisual(plot: BasePart, stageIndex: number)
    local seedType = plot:GetAttribute("SeedType")
    local def = Plants[seedType]; if not def then return end
    local stage = def.stages[stageIndex]
    if not stage then return end

    local old = plot:FindFirstChild("PlantVisual")
    if old then old:Destroy() end

    local visual = stage.buildFn(plot)
    visual.Parent = plot

    -- Optional: tween-grow from tiny → full size for smooth transition
    -- (omitted; could TweenService:Create on every part's Size)
end

local function tickPlot(plot: BasePart)
    local seedType = plot:GetAttribute("SeedType")
    if not seedType or seedType == "" then return end       -- empty plot

    local elapsed = effectiveElapsed(plot)
    local newStage = computeStageIndex(seedType, elapsed)
    local oldStage = plot:GetAttribute("StageIndex") or 0

    if newStage ~= oldStage then
        plot:SetAttribute("StageIndex", newStage)
        rebuildVisual(plot, newStage)
    end
end

RunService.Heartbeat:Connect(function()
    local now = os.clock()
    if now - lastTick < 1 then return end       -- 1Hz
    lastTick = now

    for _, plot in CollectionService:GetTagged("Plot") do
        tickPlot(plot)
    end
end)

-- Initial setup (build the right stage on server start, after offline elapsed)
for _, plot in CollectionService:GetTagged("Plot") do
    tickPlot(plot)
end
CollectionService:GetInstanceAddedSignal("Plot"):Connect(tickPlot)
```

## Planting (interaction → consume seed → set state)

```lua
-- ProximityPrompt on each empty plot
PlantPrompt.Triggered:Connect(function(player)
    local plot = PlantPrompt.Parent :: BasePart
    if plot:GetAttribute("SeedType") and plot:GetAttribute("SeedType") ~= "" then
        return        -- already planted
    end

    local profile = PlayerData.Get(player); if not profile then return end

    -- Take the player's currently-selected seed (or open a seed picker UI)
    local seedItemId = profile.selectedSeed
    local def = nil
    for _, p in Plants do
        if p.seedItemId == seedItemId then def = p; break end
    end
    if not def then return end

    -- Validate ownership of the plot (if private plots)
    if plot:GetAttribute("OwnerUserId") ~= 0 and plot:GetAttribute("OwnerUserId") ~= player.UserId then
        return
    end

    -- Consume from inventory
    if not Inventory.Remove(profile.inventory, seedItemId, 1) then return end

    -- Plant
    plot:SetAttribute("SeedType", def.id)
    plot:SetAttribute("PlantedAt", os.time())
    plot:SetAttribute("StageIndex", 1)
    plot:SetAttribute("WateredUntil", 0)
    plot:SetAttribute("RandomSeed", math.random(1, 1e9))   -- per-plant deterministic seed

    rebuildVisual(plot, 1)
    notifyPlayer(player, "Planted " .. def.name)
end)
```

## Watering (interaction → temporary multiplier)

```lua
WaterPrompt.Triggered:Connect(function(player)
    local plot = WaterPrompt.Parent :: BasePart
    if not plot:GetAttribute("SeedType") then return end

    -- Player must have a watering can (held tool / inventory item)
    if not playerHasItem(player, "WateringCan") then return end

    -- Apply: extend watered duration
    local now = os.time()
    local current = plot:GetAttribute("WateredUntil") or 0
    plot:SetAttribute("WateredUntil", math.max(current, now) + 60)    -- +60 seconds of watered growth

    -- VFX (water particles), sound
    spawnWateringEffect(plot.Position)
end)
```

## Harvesting (interaction → grant items → reset or regrow)

```lua
HarvestPrompt.Triggered:Connect(function(player)
    local plot = HarvestPrompt.Parent :: BasePart
    local seedType = plot:GetAttribute("SeedType"); if not seedType or seedType == "" then return end
    local def = Plants[seedType]; if not def then return end

    local stageIndex = plot:GetAttribute("StageIndex") or 0
    if stageIndex < #def.stages then
        notifyPlayer(player, "Not ready to harvest")
        return
    end

    local profile = PlayerData.Get(player); if not profile then return end

    -- Grant harvest yield
    local rng = Random.new()
    local yield = rng:NextInteger(def.harvestYield.Min, def.harvestYield.Max)
    Inventory.Add(profile.inventory, def.harvestItemId, yield)

    notifyPlayer(player, string.format("Harvested %d × %s", yield, def.harvestItemId))

    -- Reset or regrow
    if def.multiHarvest then
        -- Drop back to the second-to-last stage; re-fruit after another duration
        local regrowStage = #def.stages - 1
        plot:SetAttribute("StageIndex", regrowStage)
        -- Reset PlantedAt so the duration timer for the final stage starts fresh
        local elapsedToRegrow = 0
        for i = 1, regrowStage do elapsedToRegrow += def.stages[i].durationSec end
        plot:SetAttribute("PlantedAt", os.time() - elapsedToRegrow)
        plot:SetAttribute("WateredUntil", 0)
        rebuildVisual(plot, regrowStage)
    else
        -- Plot returns to empty
        plot:SetAttribute("SeedType", "")
        plot:SetAttribute("PlantedAt", 0)
        plot:SetAttribute("StageIndex", 0)
        plot:SetAttribute("WateredUntil", 0)
        local visual = plot:FindFirstChild("PlantVisual")
        if visual then visual:Destroy() end
    end
end)
```

## Persistence — saving plot state

For **per-player private plots**, save the plot data alongside the player's profile:

```lua
-- profile shape addition:
profile.plots = {
    Plot_001 = {seedType = "Apple", plantedAt = 1735776000, randomSeed = 42, wateredUntil = 0},
    Plot_002 = {seedType = "Carrot", plantedAt = 1735776300, randomSeed = 71, wateredUntil = 0},
}

-- On player join: restore plot attributes from profile
local function restorePlots(player: Player, profile)
    for plotId, plotData in profile.plots or {} do
        local plot = findPlotById(plotId, player.UserId)
        if not plot then continue end
        plot:SetAttribute("SeedType", plotData.seedType)
        plot:SetAttribute("PlantedAt", plotData.plantedAt)
        plot:SetAttribute("RandomSeed", plotData.randomSeed)
        plot:SetAttribute("WateredUntil", plotData.wateredUntil)
        -- StageIndex computed by the next tick
    end
end

-- On player leave / autosave: persist plot attributes back to profile
local function persistPlots(player: Player, profile)
    profile.plots = {}
    for _, plot in CollectionService:GetTagged("Plot") do
        if plot:GetAttribute("OwnerUserId") == player.UserId then
            profile.plots[plot:GetAttribute("PlotId")] = {
                seedType = plot:GetAttribute("SeedType"),
                plantedAt = plot:GetAttribute("PlantedAt"),
                randomSeed = plot:GetAttribute("RandomSeed"),
                wateredUntil = plot:GetAttribute("WateredUntil"),
            }
        end
    end
end
```

For **shared / public plots**, save in a global DataStore key per plot. The MemoryStore in [memorystores.md](memorystores.md) is a fit if you want cross-server consistency (everyone sees the same garden state across servers).

## Offline accrual

This is the magic moment: a player plants a seed, logs off for 4 hours, returns to a fully-grown tree.

The persistence already handles it for free. **`PlantedAt` is `os.time()`** (real seconds), so when the server restarts and `tickPlot` runs `effectiveElapsed`, it computes the correct elapsed time including all offline hours. The first tick after restore jumps straight to the appropriate stage and rebuilds the visual.

```lua
-- On player join, after restorePlots:
for _, plot in CollectionService:GetTagged("Plot") do
    if plot:GetAttribute("OwnerUserId") == player.UserId then
        tickPlot(plot)        -- forces immediate stage + visual update from restored attributes
    end
end
```

For details on `os.time()` discipline, capping max-offline, and avoiding device-clock cheats, see [offline-processing.md](offline-processing.md).

## Patterns

### Multiple plant varieties — designer-driven plot layout
Designer drops 20 plots in the world, tags each `"Plot"`, sets `OwnerUserId = 0` (public) or assigns to a player-owned region. The grow system handles all of them via the same Heartbeat sweep.

### Pest / weed mechanics
Add a `PestState` attribute. Periodically (low chance per tick) some plots develop pests; pests slow growth and visually overlay weed-parts. Player removes pests via interaction. Adds a maintenance loop on top of the passive growth.

### Withering
If a plant reaches its final stage and isn't harvested for N days, it withers and the plot must be cleared. `WitheredAt` attribute, separate visual, harvestable as "compost" item.

### Seasonal restrictions
Wheat only grows in summer; pumpkins only autumn. Tie planting to the game-time season ([time-and-weather.md](time-and-weather.md)). Reject `Plant` triggers out of season; existing plants pause growth until their season returns.

### Quality / RNG yield
On harvest, roll quality (Bronze/Silver/Gold) — gold has 2× value. Quality affected by watering frequency, fertiliser items, player's gardening skill ([progression.md](progression.md) multi-pool XP).

### Skip-grow with currency
"Pay 50 coins to skip to next stage" — common in mobile farming games. Consume coins, set `PlantedAt` back so the elapsed time covers the next stage.

### Co-op gardens
Plot is `OwnerUserId = 0` (public); anyone can water; only the planter can harvest. Or fully-co-op: anyone can do anything, harvest goes to whoever triggers it.

### Tycoon-integrated dropper farms
Each plot continuously generates a "yield" resource that drops into a collector ([world-mechanics.md](world-mechanics.md) tycoon dropper pattern). Same growth system, different reward delivery.

### Player gardening skill
Each plant + harvest grants Gardening XP via [progression.md](progression.md) multi-pool XP. Higher skill unlocks rarer crops, increases yield, reduces growth time.

## Pitfalls

- **`tickPlot` running every Heartbeat for hundreds of plots.** Throttle to 1Hz (above) — plant growth doesn't need 60Hz precision.
- **Rebuilding the visual on every tick.** Only rebuild on stage change. The example's `if newStage ~= oldStage` check guards this; don't remove it.
- **Visual snapping between stages.** Add a TweenService Size lerp from old to new for smoother transitions, OR a brief shake / particle burst on growth so it feels intentional.
- **Non-deterministic L-system seeds.** Without `RandomSeed`, the tree shape changes after every server restart. Looks like a bug to players. Always seed.
- **`os.time()` going backwards** (system clock adjustment, daylight saving). Floor `effectiveElapsed` at 0 to avoid negative-elapsed weirdness.
- **Watering applied client-side.** Players "water" infinitely, plants grow instantly. Server validates the watering-can possession + applies the timer.
- **Persistence loses `RandomSeed`.** Plant restored on next session looks different. Always persist all 4 fields (seedType, plantedAt, randomSeed, wateredUntil).
- **Offline accrual not capped.** A player gone for 6 months returns to a forest of fully-grown apples — fine if intended, exploit if not. Cap effective elapsed at e.g. 7 days.
- **Multi-harvest regrow timer wrong.** The math `os.time() - elapsedToRegrow` lets the final stage's duration count down correctly. Test this with realistic stage durations.
- **Plot tag set in Studio but no PlotId.** The save/restore logic uses PlotId as the key; without it, plots restore wrong. Validate every plot has a unique PlotId on server start.
- **Harvest grants the wrong item.** Off-by-one between catalog `harvestItemId` and the actual inventory grant. Test each plant variety's full lifecycle.
- **Public plots in a server with hundreds of players.** Last-write-wins on attributes; griefers re-plant on top of others. For shared plots, lock by player attribute or use server-side authority on plot mutations.
- **Stage durations too short for testing, too long for ship.** Have a debug attribute `GrowthMultiplier` you can crank to 100× during dev, set to 1× before ship. Easier than re-editing the catalog.
- **Plants grown server-side but only the planter sees them.** Plot attributes replicate, so the visual rebuild needs to fire for everyone — make sure `rebuildVisual` runs on the server (it does, in this card's example).
- **Saving plot state every Heartbeat.** Save on PlayerRemoving, BindToClose, and an autosave timer (every 2 minutes is fine). DataStore quota dies otherwise.

## See also
- [procedural-construction.md](procedural-construction.md) — L-system tree pattern; `buildTree` function this card calls per-stage
- [world-mechanics.md](world-mechanics.md) — resource nodes; same tag-driven setup pattern
- [oop-patterns.md](oop-patterns.md) — state machine for plot lifecycle
- [interactions.md](interactions.md) — ProximityPrompt for plant / water / harvest
- [inventory.md](inventory.md) — seed consumption, harvest grant
- [datastores.md](datastores.md) — persisting per-player plot tables
- [offline-processing.md](offline-processing.md) — `os.time()` discipline; capped offline accrual
- [task-scheduling.md](task-scheduling.md) — Heartbeat-throttled tick loop
- [tags.md](tags.md), [attributes.md](attributes.md) — designer-driven plot setup
- [progression.md](progression.md) — Gardening XP as a multi-pool skill
- [tweens-animation.md](tweens-animation.md), [vfx.md](vfx.md), [sound.md](sound.md) — growth-stage transition polish
- [time-and-weather.md](time-and-weather.md) — seasonal planting restrictions
- [memorystores.md](memorystores.md) — cross-server shared garden state
- [security.md](security.md) — server-side validation of all plot mutations
- [save-slots.md](save-slots.md) — plots are per active slot
- [crafting.md](crafting.md) — harvested produce as crafting ingredients
- [shops.md](shops.md) — sell harvested produce; buy seeds
