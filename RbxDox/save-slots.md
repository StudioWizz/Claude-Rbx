# Save Slots (Multi-Character / Multi-Save)

> Source: distilled from the multi-key DataStore pattern used by RPG, sandbox, and life-sim Roblox experiences

## Purpose
Some games have one save per player (most platformers, shooters, idle games — see [datastores.md](datastores.md) for that pattern). Others let players have multiple **save slots**: pick a slot at the title screen, each with independent currency, inventory, character class, story progress. RPGs, life sims, and "create-a-character" games need this. The DataStoreService API doesn't have a built-in slot concept — you implement it as a key-naming convention on top.

## Key shape

The convention: one **slot index** key per (player, slot), plus optionally one **metadata** key per player for the slot list.

```
Player_<userId>_Slot1   → full save data for slot 1
Player_<userId>_Slot2   → full save data for slot 2
Player_<userId>_Slot3   → ...
Player_<userId>_Meta    → {slots = {1, 2, 3}, lastUsedSlot = 2, slotMetadata = {...}}
```

Each slot is its own DataStore key — so each slot has its own session lock, its own quota budget, its own retry semantics. Slots are independent saves, not partitions of a single save.

## Slot count limits

Pick a fixed maximum (3–5 is typical) and hard-cap. Each new slot is a new DataStore key, costs quota, and complicates the picker UI. Unlimited slots is a pit trap — don't.

## Pattern: slot manager

```lua
--!strict
local DataStoreService = game:GetService("DataStoreService")
local store = DataStoreService:GetDataStore("PlayerData_v1")

local MAX_SLOTS = 3

export type SlotMetadata = {
    name: string?,         -- character/save name
    classId: string?,      -- visible to slot picker without loading the full save
    level: number?,
    lastPlayed: number?,   -- os.time() when last saved
    playtimeSeconds: number?,
}

export type SlotData = {
    -- whatever your full save shape is
    coins: number,
    inventory: {[string]: number},
    questProgress: {[string]: any},
    metadata: SlotMetadata,
}

local SaveSlots = {}

local function key(userId: number, slot: number): string
    assert(slot >= 1 and slot <= MAX_SLOTS, "invalid slot index")
    return string.format("Player_%d_Slot%d", userId, slot)
end

local function metaKey(userId: number): string
    return string.format("Player_%d_Meta", userId)
end

-- Load a single slot
function SaveSlots.LoadSlotAsync(userId: number, slot: number): SlotData?
    local ok, data = pcall(function()
        return store:GetAsync(key(userId, slot))
    end)
    if not ok then warn("LoadSlot failed:", data); return nil end
    return data
end

-- Save a single slot
function SaveSlots.SaveSlotAsync(userId: number, slot: number, data: SlotData): boolean
    data.metadata = data.metadata or {}
    data.metadata.lastPlayed = os.time()

    local ok, err = pcall(function()
        store:UpdateAsync(key(userId, slot), function(_) return data, {userId} end)
    end)
    if not ok then warn("SaveSlot failed:", err); return false end

    -- Update meta to know which slots exist (for the picker)
    SaveSlots._UpdateMeta(userId, slot, data.metadata)
    return ok
end

-- Get all slot summaries (for the slot picker, without loading full data)
function SaveSlots.ListSlotsAsync(userId: number): {[number]: SlotMetadata}
    local ok, meta = pcall(function() return store:GetAsync(metaKey(userId)) end)
    if not ok or not meta then return {} end
    return meta.slotMetadata or {}
end

function SaveSlots._UpdateMeta(userId: number, slot: number, metadata: SlotMetadata)
    pcall(function()
        store:UpdateAsync(metaKey(userId), function(old)
            old = old or {slotMetadata = {}, lastUsedSlot = nil}
            old.slotMetadata = old.slotMetadata or {}
            old.slotMetadata[slot] = metadata
            old.lastUsedSlot = slot
            return old
        end)
    end)
end

-- Delete a slot
function SaveSlots.DeleteSlotAsync(userId: number, slot: number): boolean
    local ok = pcall(function() store:RemoveAsync(key(userId, slot)) end)
    pcall(function()
        store:UpdateAsync(metaKey(userId), function(old)
            if old and old.slotMetadata then old.slotMetadata[slot] = nil end
            return old
        end)
    end)
    return ok
end

-- Copy / clone a slot to another index
function SaveSlots.CopySlotAsync(userId: number, fromSlot: number, toSlot: number): boolean
    local data = SaveSlots.LoadSlotAsync(userId, fromSlot)
    if not data then return false end
    return SaveSlots.SaveSlotAsync(userId, toSlot, data)
end

return SaveSlots
```

## Slot picker UI flow

1. Player joins → server reads `ListSlotsAsync` → sends summaries to client.
2. Client shows a slot picker UI: each slot shows class/level/lastPlayed, or "Empty" with a "+" button.
3. Player picks a slot → fires `SelectSlot` RemoteEvent.
4. Server validates the slot index, calls `LoadSlotAsync`, holds the slot's data in memory under the player's session, spawns the character.
5. From then on, the active slot is used for all saves until the player explicitly returns to the slot picker.

```lua
-- Server
local activeSlot: {[Player]: number} = {}
local activeData: {[Player]: SlotData} = {}

selectSlotRemote.OnServerEvent:Connect(function(player, slotIndex)
    if typeof(slotIndex) ~= "number" then return end
    if slotIndex < 1 or slotIndex > MAX_SLOTS then return end

    local data = SaveSlots.LoadSlotAsync(player.UserId, slotIndex)
    if not data then
        data = createFreshSave()    -- new game on empty slot
    end
    activeSlot[player] = slotIndex
    activeData[player] = data

    -- Now spawn the character with this data, etc.
    player:LoadCharacter()
end)
```

## Saving the active slot on lifecycle events

```lua
local Players = game:GetService("Players")

local function saveActive(player: Player)
    local slot = activeSlot[player]; if not slot then return end
    local data = activeData[player]; if not data then return end
    SaveSlots.SaveSlotAsync(player.UserId, slot, data)
end

Players.PlayerRemoving:Connect(saveActive)

game:BindToClose(function()
    for _, p in Players:GetPlayers() do task.spawn(saveActive, p) end
end)

-- Periodic autosave
task.spawn(function()
    while true do
        task.wait(120)
        for _, p in Players:GetPlayers() do task.spawn(saveActive, p) end
    end
end)
```

See [player-lifecycle.md](player-lifecycle.md) for the full save-on-lifecycle discipline.

## Returning to the slot picker mid-session

If the player can switch slots without leaving the server (rare, but possible):
1. Fire `LeaveSlot` RemoteEvent.
2. Server saves current slot, clears `activeSlot[player]` and `activeData[player]`.
3. Despawns the character (`player.Character:Destroy()` after disabling auto-respawn).
4. Re-shows the slot picker UI.

## Slot metadata for picker previews

The picker shouldn't need to load every slot's full data to draw the previews. Store a small `metadata` table separately (in the Meta key, or as a thin field on each slot):
- Character name and class
- Level / progress %
- Last played timestamp
- A small thumbnail (asset id, not data)
- Total playtime

Reads of the Meta key are one DataStore call total, even with 5 slots.

## ProfileService integration

If you're using ProfileService (recommended — see [community-libraries.md](community-libraries.md)), each slot is its own profile:

```lua
local ProfileService = require(...)

local profileStore = ProfileService.GetProfileStore("PlayerData_v1", {
    coins = 0, inventory = {}, ...
})

local function loadSlot(player, slot)
    local profile = profileStore:LoadProfileAsync(string.format("Player_%d_Slot%d", player.UserId, slot))
    if not profile then player:Kick("Slot busy on another server."); return end
    profile:AddUserId(player.UserId)
    profile:Reconcile()
    return profile
end
```

Each slot gets its own session lock — switching slots requires releasing the old profile and loading the new one. ProfileService handles the lock-busy case naturally.

## Pitfalls

- **Quota multiplication.** Each slot costs DataStore quota independently. Three slots × every-2-min autosave = 90 saves per player per hour. For high-population servers this adds up. Throttle autosaves; only save the *active* slot regularly.
- **Slot index validation.** Always range-check `slot` server-side (`1 <= slot <= MAX_SLOTS`). A bad client could request slot 99999 and create exotic key names.
- **Cross-slot data leak.** Don't share mutable in-memory tables between slots. After switching slots, fully rebuild per-slot state.
- **Character respawn during slot transition.** If you `:LoadCharacter` before the slot data is loaded, the player spawns with default stats. Wait for the data, then spawn.
- **Slot deletion without confirmation UI.** Players will mis-click and lose hours of progress. Always confirm; ideally back up to a `*_Trash` key for 24 hours.
- **Listing slots without the Meta key.** Calling `LoadSlotAsync` for every slot just to draw the picker triples the read quota and yields N times. Use a Meta key.
- **Slot copy with shared sub-tables.** `:CopySlot` should deep-clone before saving — otherwise mutations to one slot affect the other in memory until next save/load.
- **Stored ProfileService session lock survives across slots.** Switching slots without releasing the previous profile leaves it locked from this server's view; another server can't load it until lock expires (~5 min). Always `:Release()` before switching.
- **Versioning / migration.** When you change the save shape, every slot needs to migrate independently. Version each slot's data and migrate on load (`if data.version < 2 then ... end`).

## See also
- [datastores.md](datastores.md) — single-save DataStore patterns this builds on
- [player-lifecycle.md](player-lifecycle.md) — when to save (PlayerRemoving + BindToClose + autosave)
- [offline-processing.md](offline-processing.md) — slot-aware offline accumulation (reward only the slot the player was last using)
- [ui-system.md](ui-system.md) — building the slot picker
- [menu-systems.md](menu-systems.md) — slot picker is a major menu before character spawn
- [community-libraries.md](community-libraries.md) — **ProfileService** for session-locked per-slot profiles
- [security.md](security.md) — server-side slot index validation
