# Crafting & Recipes

> Source: distilled from RPG/survival/sandbox crafting patterns

## Purpose
A **crafting system** lets players combine inventory items via recipes to produce new items. The pattern: a **recipe registry** of input requirements + outputs, an **interaction surface** (workbench / crafting menu UI), **server validation** that the player has the ingredients, **atomic consume + grant**, and **gating** by skill/tier/station. Builds on `inventory.md` (item ownership), `interactions.md` (workbench prompt), and `security.md` (validation).

## Recipe data

```lua
--!strict
export type RecipeIngredient = {
    itemId: string,
    count: number,
}

export type RecipeOutput = {
    itemId: string,
    count: number,
}

export type Recipe = {
    id: string,
    name: string,
    description: string,
    ingredients: {RecipeIngredient},
    outputs: {RecipeOutput},
    craftTimeSec: number?,         -- nil = instant
    stationId: string?,            -- nil = anywhere; "Forge" = needs Forge nearby
    requiresLevel: number?,
    requiresUnlock: string?,       -- recipe-id prerequisite or skill-tree node
    iconId: string?,
}

local Recipes = {}

Recipes.IronSword = {
    id = "IronSword", name = "Iron Sword",
    description = "A basic iron sword.",
    ingredients = {
        {itemId = "IronIngot", count = 3},
        {itemId = "WoodenHandle", count = 1},
    },
    outputs = {{itemId = "IronSword", count = 1}},
    craftTimeSec = 5,
    stationId = "Forge",
    requiresLevel = 5,
}

Recipes.HealthPotion = {
    id = "HealthPotion", name = "Health Potion",
    ingredients = {{itemId = "RedHerb", count = 2}, {itemId = "EmptyVial", count = 1}},
    outputs = {{itemId = "HealthPotion", count = 1}},
    craftTimeSec = 2,
}

return Recipes
```

The catalog lives in `ReplicatedStorage` so client (UI) and server (validation) both see it.

## Crafting module — server core

```lua
--!strict
local Recipes = require(game.ReplicatedStorage.Recipes)
local Inventory = require(game.ServerStorage.Inventory)

local Crafting = {}

function Crafting.CanCraft(profile: PlayerProfile, recipeId: string, stationId: string?): (boolean, string?)
    local recipe = Recipes[recipeId]
    if not recipe then return false, "no-such-recipe" end

    -- Level / unlock gates
    if recipe.requiresLevel and profile.level < recipe.requiresLevel then
        return false, "level-too-low"
    end
    if recipe.requiresUnlock and not profile.unlockedRecipes[recipe.requiresUnlock] then
        return false, "not-unlocked"
    end

    -- Station check
    if recipe.stationId and recipe.stationId ~= stationId then
        return false, "wrong-station"
    end

    -- Ingredient check
    for _, ing in recipe.ingredients do
        if Inventory.Count(profile.inventory, ing.itemId) < ing.count then
            return false, "missing-ingredient:" .. ing.itemId
        end
    end

    -- Capacity check (will the outputs fit?)
    -- Optional; can also auto-drop overflow
    return true, nil
end

function Crafting.Craft(profile: PlayerProfile, recipeId: string, stationId: string?): (boolean, string?)
    local ok, err = Crafting.CanCraft(profile, recipeId, stationId)
    if not ok then return false, err end

    local recipe = Recipes[recipeId]

    -- Atomic consume
    for _, ing in recipe.ingredients do
        Inventory.Remove(profile.inventory, ing.itemId, ing.count)
    end

    -- Atomic grant
    for _, out in recipe.outputs do
        local leftover = Inventory.Add(profile.inventory, out.itemId, out.count)
        if leftover > 0 then
            -- Inventory full; drop or warn
            spawnLootBag(profile, out.itemId, leftover)
        end
    end

    return true, nil
end

return Crafting
```

The server validates *and* mutates atomically. If consume fails partway (rare with Lua single-thread), the order matters: do all the count checks first (inside `CanCraft`), then apply once.

## Crafting station via interactions

A workbench in the world opens the crafting UI when the player triggers its prompt:

```lua
-- Server Script under the workbench Model
local prompt = workbench:WaitForChild("ProximityPrompt")
prompt.Triggered:Connect(function(player)
    OpenCraftingRemote:FireClient(player, workbench:GetAttribute("StationId"))
end)
```

The client opens the menu showing the recipes available at that station. On craft request:

```lua
CraftRemote.OnServerEvent:Connect(function(player, recipeId, stationId)
    if typeof(recipeId) ~= "string" then return end

    local profile = PlayerData.Get(player); if not profile then return end

    -- Optional: re-validate the player is still near the station
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    local station = findStation(stationId)
    if station and hrp and (hrp.Position - station.Position).Magnitude > MAX_RANGE then return end

    local ok, err = Crafting.Craft(profile, recipeId, stationId)
    CraftResultRemote:FireClient(player, {ok = ok, err = err, recipeId = recipeId})
end)
```

For multi-step recipes (smelt ore → forge ingot → craft sword), each step is its own recipe; the player chains them. For continuous "craft N" queues, see Patterns below.

## Crafting time + animation

For recipes with `craftTimeSec > 0`, present a progress bar; consume ingredients on start, grant outputs on completion:

```lua
function Crafting.StartTimed(player: Player, recipeId: string, stationId: string?)
    local profile = PlayerData.Get(player); if not profile then return end
    local ok, err = Crafting.CanCraft(profile, recipeId, stationId)
    if not ok then return false, err end

    local recipe = Recipes[recipeId]
    -- Reserve ingredients now (consume), grant later
    for _, ing in recipe.ingredients do
        Inventory.Remove(profile.inventory, ing.itemId, ing.count)
    end

    CraftStartedRemote:FireClient(player, {recipeId = recipeId, durationSec = recipe.craftTimeSec})

    task.delay(recipe.craftTimeSec, function()
        if not player.Parent then
            -- Player left mid-craft: refund? grant when they rejoin? Designer decision.
            return
        end
        for _, out in recipe.outputs do
            Inventory.Add(profile.inventory, out.itemId, out.count)
        end
        CraftFinishedRemote:FireClient(player, {recipeId = recipeId})
    end)
end
```

For server crash safety, consider tracking pending crafts in the profile (so they resume on next session) or using the simpler "instant on success" model.

## Recipe unlocks

For discoverable / progression-gated recipes:

```lua
profile.unlockedRecipes = {
    HealthPotion = true,
    IronSword = true,
    -- MysticBlade not present = locked
}

-- Unlock via finding a recipe scroll in the world
function Crafting.Unlock(profile: PlayerProfile, recipeId: string)
    profile.unlockedRecipes[recipeId] = true
    -- Notify client
end
```

The crafting UI filters to show only unlocked recipes (and maybe "??? recipe" placeholders for hinted-at ones).

## Patterns

### Discovery / experimentation
Some games let players discover recipes by trying ingredient combinations:
```lua
local function findRecipeByIngredients(ingredients: {{itemId: string, count: number}}): Recipe?
    -- Sort both for canonical comparison
    for _, recipe in Recipes do
        if matchesIngredients(recipe.ingredients, ingredients) then return recipe end
    end
end
```
On submit, look up the matching recipe; reward discovery (unlock + small bonus). Failed combos either consume the ingredients (penalty) or refund (forgiving).

### Crafting queue
Player queues N crafts; server processes sequentially:
```lua
profile.craftingQueue = {
    {recipeId = "IronIngot", quantity = 10},
    {recipeId = "IronSword", quantity = 1},
}
```
Run the queue in a server-side loop, granting one output at a time, so the player can watch their stockpile grow.

### Quality / luck on craft
Crit-craft chance for higher-quality outputs:
```lua
local critChance = 0.1 + profile.attributes.craftLuck * 0.01
if math.random() < critChance then
    -- Crit — output is a +1 quality version
    Inventory.Add(profile.inventory, recipe.outputs[1].itemId .. "+1", 1)
else
    Inventory.Add(profile.inventory, recipe.outputs[1].itemId, recipe.outputs[1].count)
end
```

### Auto-craft / repeatable buttons
For idle-game style "auto-smelt while you do other things," tag the queue as repeating and re-enqueue after each completion until cancelled or ingredients run out.

### Crafting XP
Grant XP per craft to a "Crafting" sub-skill (separate from combat XP):
```lua
profile.craftingXp = (profile.craftingXp or 0) + recipe.craftXp
-- Recompute crafting level via the same Progression.GrantXp pattern
```

## Pitfalls

- **Trusting client-supplied ingredient list.** "I'm crafting with 0 of each" → server must look up the recipe canonically and validate ingredients server-side.
- **Granting outputs before consuming ingredients.** A failure between (e.g., inventory-full on output) leaves the player with both. Consume first; if grant fails, the ingredients are gone (acceptable) or refunded (designer choice).
- **Inventory full when outputs exceed capacity.** Either reject the craft, drop overflow as loot, or queue the outputs in an "overflow" buffer for later pickup. Pick one and stick with it.
- **Per-frame craft trigger.** Player spamming a "craft" button N times per frame fires the remote N times. Throttle (one craft per N seconds, or a single-craft-in-flight lock).
- **Craft-station drift.** Player moves away from the station mid-craft. Either lock them in place during craft, or accept it (most games do).
- **Recipe lookup case sensitivity.** "ironSword" ≠ "IronSword". Use a canonical id and compare with `==` not lowercased lookups.
- **Quantity = 0 or negative.** A buggy quantity field grants nothing or removes inventory. Clamp `quantity = math.max(1, ...)`.
- **No level / unlock check.** Early-game players craft endgame items by knowing the id. Always gate.
- **Crafting completion without player.** Player left during the timer; the spawned outputs go nowhere (or worse, into a nil profile). Check `player.Parent` before granting.
- **Recipes that take 0 ingredients but produce items.** Free-money exploit; usually a designer error. Validate at recipe-load time.
- **Discovery-based crafting reveals recipes via brute force.** Add cooldowns or ingredient costs to failed attempts.
- **Saving the crafting queue but not validating it on resume.** A queue of "100x MysticBlade" survives a server restart but the player no longer has the ingredients. Re-validate on resume; cancel invalid queue entries.

## See also
- [inventory.md](inventory.md) — the underlying item store crafting consumes/produces
- [interactions.md](interactions.md) — workbench ProximityPrompt opens the crafting UI
- [security.md](security.md) — server-side validation discipline
- [datastores.md](datastores.md) — persisting `unlockedRecipes` and any pending queue
- [progression.md](progression.md) — crafting XP, level gates
- [ui-system.md](ui-system.md) — crafting menu (recipe list, ingredient checklist, "Craft" button)
- [tweens-animation.md](tweens-animation.md) — craft progress bar
- [npcs.md](npcs.md) — vendor NPC alternative for "buy what you can't craft"
- [community-libraries.md](community-libraries.md) — `t` for runtime payload validation
