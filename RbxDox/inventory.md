# Inventory System

> Source: distilled from RPG/survival/sandbox inventory patterns and the standard Roblox profile-stored inventory shape

## Purpose
An **inventory** is the player's owned items — separate from their `Backpack` (which is the engine's equipped-Tools container) and separate from their `leaderstats` (which is just a display). Inventory holds counts of items with stack support, sometimes per-slot positioning, sometimes equipped-from-inventory state. This card covers the canonical data shape, server-authoritative mutations, and the typical UI pattern. For equipping items into specific gear slots (head/chest/weapon), see [equipment.md](equipment.md). For consuming items in recipes, see [crafting.md](crafting.md).

## Backpack vs Inventory — disambiguation

- **`Backpack`** is the Roblox-engine container of `Tool` instances the player has access to. Walking up to a Tool or being granted one lands it here. Tools in Backpack appear in the bottom hotbar and are equippable. See [tools.md](tools.md).
- **Inventory** (your data) is everything else the player owns: stackable consumables, materials, key items, eggs, loot. Most items aren't Tools; they're abstract item ids with counts.

These can be **bridged**: an inventory entry like "Sword × 1" might, when "equipped," clone a Tool template into Backpack. The inventory remains the source of truth; the Backpack is a transient view.

## Data shape

```lua
--!strict
export type InventoryEntry = {
    itemId: string,        -- "HealthPotion", "IronOre", "WoolCloak"
    count: number,         -- stack size (1 for unstackable items like a unique sword)
    metadata: {[string]: any}?,  -- per-instance state: durability, enchantments, custom name, color
}

export type Inventory = {
    items: {[string]: InventoryEntry},   -- keyed by itemId for stackables; unique key for non-stackables
    capacity: number?,                   -- nil = unlimited
}

-- Stored on the player's profile (see datastores.md)
profile.inventory = {
    items = {
        ["HealthPotion"]      = {itemId = "HealthPotion", count = 5},
        ["IronOre"]           = {itemId = "IronOre", count = 23},
        ["MysticSword#a8d2"]  = {itemId = "MysticSword", count = 1, metadata = {durability = 100, enchantment = "Fire"}},
    },
    capacity = 50,
}
```

**Stackable** items use the itemId as the key (one entry per type). **Unique** items (each instance has its own state) use a generated id (`"MysticSword#a8d2"`) as the key with the itemId in the entry.

The capacity check counts entries (slots used), not total item count — so 5 health potions take 1 slot, but 5 unique swords take 5 slots.

## Item catalog

A static catalog defines every possible item: stackable, max stack size, icon, display name, description. Same idea as [shops.md](shops.md):

```lua
--!strict
export type ItemDef = {
    id: string,
    name: string,
    description: string,
    iconId: string,
    stackable: boolean,
    maxStack: number?,         -- per stack; nil = unlimited within capacity
    rarity: "Common" | "Rare" | "Epic" | "Legendary"?,
    consumable: boolean?,      -- can it be used to trigger an effect?
    equipSlot: string?,        -- "Weapon", "Head", "Chest", etc.; nil = not equippable
}

local Items = {}

Items.HealthPotion = {
    id = "HealthPotion", name = "Health Potion",
    description = "Restores 50 HP.",
    iconId = "rbxassetid://...",
    stackable = true, maxStack = 99,
    rarity = "Common", consumable = true,
}

Items.MysticSword = {
    id = "MysticSword", name = "Mystic Sword",
    description = "A blade of myth.",
    iconId = "rbxassetid://...",
    stackable = false, rarity = "Epic", equipSlot = "Weapon",
}

return Items
```

Catalog lives in `ReplicatedStorage` so server (validation) and client (UI) both see it.

## Inventory module — server-side core

```lua
--!strict
local HttpService = game:GetService("HttpService")
local Items = require(game.ReplicatedStorage.Items)

local Inventory = {}

local function newId(itemId: string): string
    return itemId .. "#" .. HttpService:GenerateGUID(false):sub(1, 4)
end

-- Add N of an item to inventory; returns leftover (0 if all added)
function Inventory.Add(inv: Inventory, itemId: string, amount: number, metadata: {[string]: any}?): number
    amount = math.max(0, amount)
    local def = Items[itemId]; if not def then return amount end

    if def.stackable then
        local entry = inv.items[itemId]
        if entry then
            local cap = def.maxStack or math.huge
            local space = cap - entry.count
            local toAdd = math.min(amount, space)
            entry.count += toAdd
            return amount - toAdd
        else
            if inv.capacity and countSlots(inv) >= inv.capacity then return amount end
            local cap = def.maxStack or math.huge
            local toAdd = math.min(amount, cap)
            inv.items[itemId] = {itemId = itemId, count = toAdd, metadata = metadata}
            return amount - toAdd
        end
    else
        local added = 0
        for _ = 1, amount do
            if inv.capacity and countSlots(inv) >= inv.capacity then break end
            local key = newId(itemId)
            inv.items[key] = {itemId = itemId, count = 1, metadata = metadata}
            added += 1
        end
        return amount - added
    end
end

-- Remove N of a stackable item; returns true if removed in full
function Inventory.Remove(inv: Inventory, itemId: string, amount: number): boolean
    local entry = inv.items[itemId]
    if not entry then return false end
    if entry.count < amount then return false end
    entry.count -= amount
    if entry.count <= 0 then inv.items[itemId] = nil end
    return true
end

-- Remove a specific unique item by key (for non-stackables)
function Inventory.RemoveByKey(inv: Inventory, key: string): boolean
    if not inv.items[key] then return false end
    inv.items[key] = nil
    return true
end

function Inventory.Count(inv: Inventory, itemId: string): number
    -- For stackables, look up directly
    local entry = inv.items[itemId]
    if entry then return entry.count end
    -- For non-stackables, count keys with matching itemId
    local n = 0
    for _, e in inv.items do if e.itemId == itemId then n += 1 end end
    return n
end

function countSlots(inv: Inventory): number
    local n = 0; for _ in inv.items do n += 1 end; return n
end

return Inventory
```

## Server-authoritative mutations

Players send "use this item" intents; the server checks ownership and applies effects:

```lua
local UseItemRemote = game.ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("UseItem")

UseItemRemote.OnServerEvent:Connect(function(player, itemKey)
    if typeof(itemKey) ~= "string" then return end

    local profile = PlayerData.Get(player); if not profile then return end
    local inv = profile.inventory

    local entry = inv.items[itemKey]; if not entry then return end
    local def = Items[entry.itemId]; if not def then return end
    if not def.consumable then return end

    -- Apply effect (consume one)
    if entry.itemId == "HealthPotion" then
        local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
        if hum then hum.Health = math.min(hum.MaxHealth, hum.Health + 50) end
        Inventory.Remove(inv, entry.itemId, 1)
    end
    -- ... per-item handlers

    InventoryUpdatedRemote:FireClient(player, inv)
end)
```

Push the updated inventory to the client after every change so the UI reflects ground truth.

## Inventory UI pattern

A simple inventory UI is a grid of slots:
- Each slot shows the item icon, count badge if > 1, rarity-tinted border.
- Hover/tap shows description tooltip.
- Click activates a context menu: Use, Equip, Drop, Inspect.

Use [ui-system.md](ui-system.md) `UIGridLayout` for the grid; build slot-frames from a template. Update by re-rendering when `InventoryUpdatedRemote` fires, or by mounting a Roact-style declarative component if you've adopted [Roact](community-libraries.md).

## Patterns

### Pickups
A pickup part triggers `Inventory.Add` on Touched (see [world-mechanics.md](world-mechanics.md)). Notify the player with a toast (see [live-ops.md](live-ops.md)).

### Loot tables
```lua
local LOOT = {
    {itemId = "Coin", weight = 100, count = 5},
    {itemId = "HealthPotion", weight = 30, count = 1},
    {itemId = "MysticSword", weight = 1, count = 1},
}

local function rollLoot()
    local total = 0
    for _, l in LOOT do total += l.weight end
    local roll = math.random() * total
    local accum = 0
    for _, l in LOOT do accum += l.weight; if roll <= accum then return l end end
end
```

### Drop/discard from inventory
```lua
DropRemote.OnServerEvent:Connect(function(player, itemKey, amount)
    -- ... validate ...
    if Inventory.Remove(inv, itemKey, amount) then
        spawnLootBag(player.Character.HumanoidRootPart.Position, itemKey, amount)
    end
end)
```

### Capacity overflow
When `Inventory.Add` returns leftover > 0, either:
- Reject the pickup and leave it in the world (player must drop something first).
- Show "inventory full" toast and discard the leftover.
- Auto-drop the leftover into a loot bag at the player's feet.

### Trading between players
```lua
TradeOfferRemote.OnServerEvent:Connect(function(playerA, playerB, offerA, offerB)
    -- Validate both inventories have what they're offering
    -- Atomically: remove from both, add to both
    -- Send TradeAcceptedRemote to both clients
end)
```
Trading is **highly exploitable** — both sides need to confirm via UI; server validates at the moment of swap; never allow client-driven inventory mutation. See [security.md](security.md).

## Pitfalls

- **Inventory not persisted.** The `profile.inventory` table must be saved with the rest of the profile (see [datastores.md](datastores.md)) — it's not magical state.
- **Mutating inventory client-side.** A client claim of "I now have 1000 swords" must be ignored. All mutations server-side, then push the update.
- **No capacity check on grant.** `Inventory.Add` should respect `capacity`; if you skip it, infinite inventories leak memory and DataStore byte limits.
- **Stackable item with `count = 0` not removed.** The `Remove` must clear the entry when count hits zero, otherwise UI shows "Health Potion × 0".
- **Unique item key collisions.** `newId` uses GUID prefix — short prefixes collide more often than you'd think. Use the full GUID or check before assigning.
- **Inventory passed by reference and mutated.** Lua tables are reference types — passing `profile.inventory` to a helper that mutates it changes the original. Useful (avoids copy cost), dangerous (action-at-a-distance bugs).
- **Item count overflow.** Don't let stackables exceed `2^31`. Even with `maxStack`, a buggy grant loop could overflow. Clamp.
- **Saving inventory in `Heartbeat`.** Don't. Save on PlayerRemoving + BindToClose + periodic autosave (see [player-lifecycle.md](player-lifecycle.md)).
- **No itemId catalog validation.** A bug or exploit grants `"FreeBillionDollars"` — your code happily stores the entry. On grant, always check `Items[itemId]` exists.
- **Bridging Backpack and Inventory inconsistently.** If equipping moves an item from inventory to Backpack but unequipping doesn't return it, the item vanishes. Define one direction as canonical (usually inventory) and treat Backpack as a derived view.
- **Showing inventory data the player shouldn't see.** Other players inspecting your character shouldn't see your full inventory unless you opt in (e.g., trade view). Only push the player's own inventory to their own client.

## See also
- [datastores.md](datastores.md) — persisting `profile.inventory`
- [trading-and-gifting.md](trading-and-gifting.md) — full mutual trade + gifting flow built on inventory transfers
- [tools.md](tools.md) — Backpack vs inventory; bridging "equip" → Tool clone
- [equipment.md](equipment.md) — slot-based gear with stat aggregation
- [crafting.md](crafting.md) — consuming inventory items as recipe ingredients
- [shops.md](shops.md) — granting items into inventory on purchase
- [world-mechanics.md](world-mechanics.md) — pickup tag pattern that calls `Inventory.Add`
- [ui-system.md](ui-system.md) — inventory grid UI
- [security.md](security.md) — server-authoritative mutations; trading exploits
- [save-slots.md](save-slots.md) — inventory is per active slot
- [community-libraries.md](community-libraries.md) — `t` for runtime payload validation
