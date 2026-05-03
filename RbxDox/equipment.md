# Equipment Slots & Loadouts

> Source: distilled from RPG/FPS gear-slot patterns

## Purpose
Equipment is the subset of inventory that the player **wears in fixed slots** — head, chest, legs, weapon, ring, etc. Each slot accepts one item, items contribute aggregated stats, sets of items grant bonuses, and players save loadout presets to swap quickly. Distinct from `inventory.md` (which holds *all* owned items including equipment when not worn) and from `tools.md` (which is the engine's Backpack — equippable hand-tools). Equipment is structurally similar to `pets.md` (slot-limited equipping with multipliers), but for gear with stat aggregation.

## Slot definitions

```lua
--!strict
export type EquipSlot = "Head" | "Chest" | "Legs" | "Hands" | "Feet" | "Weapon" | "Offhand" | "Ring1" | "Ring2" | "Necklace"

local SLOTS: {EquipSlot} = {"Head", "Chest", "Legs", "Hands", "Feet", "Weapon", "Offhand", "Ring1", "Ring2", "Necklace"}
```

Each item declares which slot it fits via the catalog (extending `inventory.md`):

```lua
Items.IronHelm = {
    id = "IronHelm", name = "Iron Helm", stackable = false,
    equipSlot = "Head",
    stats = {defense = 5, vitality = 1},
    setId = "IronArmor",          -- members of a set for set-bonus calculation
}

Items.MysticSword = {
    id = "MysticSword", name = "Mystic Sword", stackable = false,
    equipSlot = "Weapon",
    stats = {damage = 25, magicDamage = 10},
}
```

## Profile shape

```lua
profile.equipment = {
    slots = {} :: {[EquipSlot]: string?},      -- slot → inventory item key (uid)
    presets = {} :: {[string]: {[EquipSlot]: string?}},   -- saved loadouts by name
}

-- Example state:
profile.equipment.slots = {
    Head = "IronHelm#a8d2",
    Weapon = "MysticSword#x91f",
    -- Chest, Legs, ... = nil (empty)
}
```

The slot value is the **inventory uid** of the equipped item, not a copy of the item — equipment slots are *references* into inventory. This way the item is in exactly one place at a time, and selling/dropping logic only has to look at one source of truth.

## Equip / unequip

```lua
local Equipment = {}

function Equipment.Equip(profile: PlayerProfile, slot: EquipSlot, itemKey: string): boolean
    local entry = profile.inventory.items[itemKey]
    if not entry then return false end
    local def = Items[entry.itemId]
    if not def or def.equipSlot ~= slot then return false end

    -- Unequip whatever's currently in that slot (returns to inventory implicitly — it's a reference)
    profile.equipment.slots[slot] = itemKey
    Equipment.RecomputeStats(profile)
    return true
end

function Equipment.Unequip(profile: PlayerProfile, slot: EquipSlot)
    profile.equipment.slots[slot] = nil
    Equipment.RecomputeStats(profile)
end

function Equipment.GetEquipped(profile: PlayerProfile, slot: EquipSlot): InventoryEntry?
    local key = profile.equipment.slots[slot]
    if not key then return nil end
    return profile.inventory.items[key]
end

return Equipment
```

Note: when equipment is a reference into inventory, **selling an equipped item must unequip it first**. A helper `Inventory.Remove` should check the equipment slots and clear them when destroying an entry.

## Stat aggregation

Walk all equipped slots, sum (or multiply) stats:

```lua
function Equipment.AggregateStats(profile: PlayerProfile): {[string]: number}
    local agg: {[string]: number} = {}

    for slot, itemKey in profile.equipment.slots do
        if not itemKey then continue end
        local entry = profile.inventory.items[itemKey]
        local def = entry and Items[entry.itemId]
        if not def or not def.stats then continue end

        for stat, value in def.stats do
            agg[stat] = (agg[stat] or 0) + value
        end

        -- Per-instance metadata can override / boost
        if entry.metadata and entry.metadata.bonusStats then
            for stat, value in entry.metadata.bonusStats do
                agg[stat] = (agg[stat] or 0) + value
            end
        end
    end

    -- Apply set bonuses
    local setCounts = countSets(profile)
    for setId, count in setCounts do
        local bonus = SET_BONUSES[setId] and SET_BONUSES[setId][count]
        if bonus then
            for stat, value in bonus do
                agg[stat] = (agg[stat] or 0) + value
            end
        end
    end

    return agg
end
```

`AggregateStats` returns the player's effective stats from gear; merge with their base stats (from `progression.md` attributes) for the final damage / defense / max-HP values.

## Set bonuses

Items with the same `setId` form a set; equipping multiple grants increasing bonuses:

```lua
local SET_BONUSES = {
    IronArmor = {
        [2] = {defense = 5},                              -- 2 pieces: +5 defense
        [4] = {defense = 15, vitality = 5},               -- 4 pieces: +15 defense, +5 vitality
        [5] = {defense = 30, vitality = 10, criticalHit = 0.05},  -- full set
    },
}

function countSets(profile: PlayerProfile): {[string]: number}
    local counts: {[string]: number} = {}
    for slot, itemKey in profile.equipment.slots do
        if not itemKey then continue end
        local entry = profile.inventory.items[itemKey]
        local def = entry and Items[entry.itemId]
        if def and def.setId then
            counts[def.setId] = (counts[def.setId] or 0) + 1
        end
    end
    return counts
end
```

The aggregator applies the largest tier achieved (2-piece bonus + 4-piece bonus accumulate; 5-piece doesn't *replace*, it *adds* on top).

## RecomputeStats — one entry point

Centralize re-aggregation so callers don't forget:

```lua
function Equipment.RecomputeStats(profile: PlayerProfile)
    local gear = Equipment.AggregateStats(profile)
    local base = profile.attributes      -- from progression.md

    profile.derivedStats = {
        maxHealth = 100 + (base.vitality or 0) * 10 + (gear.vitality or 0) * 10 + (gear.maxHealth or 0),
        damage = 10 + (base.strength or 0) * 2 + (gear.damage or 0),
        defense = (base.endurance or 0) + (gear.defense or 0),
        magicDamage = (base.intelligence or 0) * 1.5 + (gear.magicDamage or 0),
        critChance = 0.05 + (gear.critChance or 0),
        -- ...
    }

    -- Push to character (e.g., update Humanoid.MaxHealth)
    local hum = getHumanoidFromProfile(profile)
    if hum then
        hum.MaxHealth = profile.derivedStats.maxHealth
        hum.Health = math.min(hum.Health, hum.MaxHealth)    -- clamp on max-HP decrease
    end
end
```

Every equip/unequip/inventory-change calls `RecomputeStats`. Combat reads from `profile.derivedStats` (never from raw item stats — they may not be equipped).

## Visual reflection

When a player equips armor, their character should look armored. Two patterns:

### A. Tool-based (equipment = visual Tool in Backpack)
Each gear item has a corresponding Tool clone-template; equipping adds the Tool to Backpack and `:EquipTool`s it. See [tools.md](tools.md). Works for visible weapons (sword in hand).

### B. Accessory-based (equipment = HumanoidDescription update)
Update the player's `HumanoidDescription` with the gear's accessory ids. See [avatar-customization.md](avatar-customization.md). Works for cosmetic armor pieces, hats, capes.

```lua
function Equipment.ApplyVisuals(player: Player, profile: PlayerProfile)
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    if not hum then return end

    local desc = hum:GetAppliedDescription()
    desc.HatAccessory = ""
    desc.BackAccessory = ""
    -- ... clear other slots

    for slot, itemKey in profile.equipment.slots do
        local entry = profile.inventory.items[itemKey]
        local def = entry and Items[entry.itemId]
        if def and def.visualAccessoryId then
            -- Map slot to the right accessory field
            if slot == "Head" then desc.HatAccessory = tostring(def.visualAccessoryId) end
            -- ... etc
        end
    end

    hum:ApplyDescription(desc)
end
```

Hybrid: `Weapon` slot uses Tool-based; armor slots use accessory-based. Best of both.

## Loadouts (presets)

Save the current slot config under a name; restore it later:

```lua
function Equipment.SavePreset(profile: PlayerProfile, name: string)
    profile.equipment.presets[name] = table.clone(profile.equipment.slots)
end

function Equipment.LoadPreset(profile: PlayerProfile, name: string): boolean
    local preset = profile.equipment.presets[name]
    if not preset then return false end

    for slot in SLOTS do
        local itemKey = preset[slot]
        if itemKey and profile.inventory.items[itemKey] then
            -- Item still exists — equip
            profile.equipment.slots[slot] = itemKey
        else
            profile.equipment.slots[slot] = nil    -- item gone (sold/used) — slot empty
        end
    end

    Equipment.RecomputeStats(profile)
    Equipment.ApplyVisuals(getPlayer(profile), profile)
    return true
end
```

Each preset is a snapshot of `slots`; loading re-applies. If items in a preset have been sold or destroyed, those slots end up empty (graceful degrade).

## Patterns

### Quick-swap loadouts in combat
Bind preset-load to keys (1, 2, 3 for "Tank / DPS / Stealth"). Useful for builds that swap mid-combat. Add a swap cooldown to prevent abuse.

### Auto-equip best
For new players or lazy ones, equip the highest-stat item in each slot:
```lua
function Equipment.AutoEquipBest(profile: PlayerProfile, scoreFn: ((InventoryEntry) -> number)?)
    scoreFn = scoreFn or function(entry)
        local def = Items[entry.itemId]
        local s = 0
        for stat, v in def.stats or {} do s += v end
        return s
    end

    for _, slot in SLOTS do
        local best, bestScore = nil, -math.huge
        for key, entry in profile.inventory.items do
            local def = Items[entry.itemId]
            if def and def.equipSlot == slot then
                local score = scoreFn(entry)
                if score > bestScore then best, bestScore = key, score end
            end
        end
        profile.equipment.slots[slot] = best
    end
    Equipment.RecomputeStats(profile)
end
```

### Item comparison tooltip
On hover-inventory-item, show "would equip" stat delta vs currently-equipped:
```lua
local cur = Equipment.GetEquipped(profile, def.equipSlot)
local delta = computeDelta(cur, hoveredItem)
-- Render "Damage +5 → +10"
```

### Required-level equipment
Cap equip by level:
```lua
if def.requiresLevel and profile.level < def.requiresLevel then return false end
```

### Two-handed weapons disable Offhand
If equipping a `twoHanded` weapon, also unequip Offhand:
```lua
if def.twoHanded then profile.equipment.slots.Offhand = nil end
```

## Pitfalls

- **Equipped item also drop-able from inventory.** A naive sell/drop on inventory removes the item but the slot still references it → broken. Always check equipment references when removing inventory entries.
- **Slot mismatch.** Trying to equip a Helm in the Weapon slot. Validate `def.equipSlot == slot`.
- **Stats applied per-frame instead of on equip.** Recompute on equip events only. Reading from `derivedStats` is per-frame fine.
- **Set bonus double-counted.** Walking the set-counts and adding multiple tiers naïvely (2 + 4 + 5) can over-grant. Either accumulate (intended; bonuses stack as you wear more) or override (replace lower-tier with higher) — pick one and document.
- **MaxHealth decrease without health clamp.** Player's current HP exceeds new MaxHealth → engine clamps but it looks weird; clamp explicitly and animate the bar.
- **Visual not updating after equip.** ApplyVisuals not called, or HumanoidDescription change didn't take effect because the character was nil. Re-apply on `CharacterAdded` too.
- **Loadout preset references items the player no longer owns.** Always check `inventory.items[key]` before applying.
- **Server doesn't validate the slot move.** A client requesting "equip Helm in Weapon slot" can pass through if the server doesn't check `equipSlot`. Validate.
- **`table.clone` is shallow.** Cloning `slots` is fine because values are strings (uids). Cloning a struct with nested tables needs deep-copy.
- **Two-handed weapon equipped without freeing Offhand.** Glitch state where both are set but the rules say only one. Define and enforce slot conflict rules.
- **Set bonus calculator scales with sets-in-game.** A growing game with hundreds of sets becomes slow. Use per-set lookup instead of iterating all.
- **Items with custom enchantments not surviving save/load.** `metadata` (durability, enchantment) is item-instance-specific; persist with each entry, not aggregated on the def.

## See also
- [inventory.md](inventory.md) — the underlying item store; equipment slots are references into it
- [progression.md](progression.md) — base attributes that combine with gear stats into derivedStats
- [combat.md](combat.md) — reads `profile.derivedStats` for damage/defense calculations
- [tools.md](tools.md) — equipping a Weapon-slot item often spawns a Tool in Backpack
- [avatar-customization.md](avatar-customization.md) — accessory-based visual reflection of equipped gear
- [datastores.md](datastores.md) — persisting `profile.equipment.slots` and presets
- [pets.md](pets.md) — structurally similar (slot-equipping with multipliers)
- [crafting.md](crafting.md) — common source of new equipment
- [shops.md](shops.md) — common source of new equipment
- [ui-system.md](ui-system.md) — equipment UI: paper-doll layout, slot frames, item comparison
- [security.md](security.md) — server-validated equip operations
