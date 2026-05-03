# Quest Systems

> Source: distilled from RPG quest patterns and the standard Roblox state-machine + DataStore + interactions composition

## Purpose
A quest system tracks **what the player is currently working on**, **what they've completed**, and **what triggers progress** for each active objective. Like shops, NPCs, and save slots, it isn't a single Roblox class — it's a composition: a quest definition catalog, per-player quest state stored in the DataStore profile, an event bus that listens for "things that might progress a quest" (kills, pickups, location entry, NPC interactions), a quest-giver UI, and a quest log UI. This card is the canonical assembly.

## Quest data shapes

### Quest definition (static catalog)
```lua
--!strict
export type ObjectiveType =
    "Kill" |              -- kill N of target
    "Collect" |           -- pick up N of item
    "TalkTo" |            -- talk to NPC X
    "Reach" |             -- enter region X
    "Use"                 -- use item / interact with object

export type Objective = {
    type: ObjectiveType,
    targetId: string,     -- "Wolf", "HealthPotion", "ApothecaryNpc", "MountainPeak"
    count: number,        -- required count (1 for Talk/Reach)
    description: string,
}

export type QuestReward = {
    coins: number?,
    xp: number?,
    items: {string}?,     -- item ids granted on completion
}

export type QuestDef = {
    id: string,
    name: string,
    description: string,
    giverId: string?,     -- NPC who offers it
    turnInId: string?,    -- NPC who completes it (often == giverId)
    requiresLevel: number?,
    requiresCompleted: {string}?,   -- prereq quest ids
    objectives: {Objective},
    rewards: QuestReward,
    repeatable: boolean?,
}

local Quests = {}

Quests.WOLF_HUNT = {
    id = "WolfHunt",
    name = "Trouble With Wolves",
    description = "The shepherd's flock is being picked off. Help her out.",
    giverId = "ShepherdNpc",
    turnInId = "ShepherdNpc",
    objectives = {
        {type = "Kill", targetId = "Wolf", count = 5,
         description = "Slay 5 wolves"},
    },
    rewards = {coins = 200, xp = 100, items = {"WoolCloak"}},
}

Quests.LOST_LETTER = {
    id = "LostLetter",
    name = "The Lost Letter",
    description = "Find the courier's missing letter and deliver it.",
    giverId = "CourierNpc",
    turnInId = "MayorNpc",
    requiresLevel = 5,
    objectives = {
        {type = "Collect", targetId = "MissingLetter", count = 1, description = "Find the letter"},
        {type = "TalkTo", targetId = "MayorNpc", count = 1, description = "Deliver to the mayor"},
    },
    rewards = {coins = 500, xp = 300},
}

return Quests
```

Catalog lives in a `ModuleScript` in `ReplicatedStorage` so client (UI) and server (logic) can both read it.

### Per-player quest state (in profile / DataStore)
```lua
export type ObjectiveProgress = {progress: number, completed: boolean}

export type ActiveQuest = {
    id: string,
    objectives: {ObjectiveProgress},   -- parallel to QuestDef.objectives
    startedAt: number,                 -- os.time()
}

export type CompletedQuest = {
    id: string,
    completedAt: number,
}

-- Stored in profile:
profile.quests = {
    active = {} :: {ActiveQuest},
    completed = {} :: {[string]: CompletedQuest},     -- keyed by quest id for O(1) lookup
}
```

## QuestSystem module — server-side core

```lua
--!strict
local Quests = require(game.ReplicatedStorage.Quests)
local PlayerData = require(game.ServerStorage.PlayerData)

local QuestSystem = {}

-- Check if a quest is available (level + prereqs + not already active/completed)
function QuestSystem.IsAvailable(player: Player, questId: string): boolean
    local def = Quests[questId]; if not def then return false end
    local profile = PlayerData.Get(player); if not profile then return false end

    if def.requiresLevel and profile.level < def.requiresLevel then return false end
    if def.requiresCompleted then
        for _, prereq in def.requiresCompleted do
            if not profile.quests.completed[prereq] then return false end
        end
    end

    -- Already active?
    for _, q in profile.quests.active do
        if q.id == questId then return false end
    end

    -- Already completed and not repeatable?
    if profile.quests.completed[questId] and not def.repeatable then return false end

    return true
end

-- Start a quest
function QuestSystem.Start(player: Player, questId: string): boolean
    if not QuestSystem.IsAvailable(player, questId) then return false end
    local def = Quests[questId]
    local profile = PlayerData.Get(player)

    local objectives = {}
    for _ in def.objectives do
        table.insert(objectives, {progress = 0, completed = false})
    end

    table.insert(profile.quests.active, {
        id = questId,
        objectives = objectives,
        startedAt = os.time(),
    })

    QuestUpdatedRemote:FireClient(player, profile.quests)
    return true
end

-- Progress an objective. Called from event handlers (kill, pickup, talk, reach).
function QuestSystem.Progress(player: Player, objType: string, targetId: string, amount: number?)
    amount = amount or 1
    local profile = PlayerData.Get(player); if not profile then return end

    local changed = false
    for _, active in profile.quests.active do
        local def = Quests[active.id]; if not def then continue end
        for i, obj in def.objectives do
            local prog = active.objectives[i]
            if not prog.completed and obj.type == objType and obj.targetId == targetId then
                prog.progress = math.min(prog.progress + amount, obj.count)
                if prog.progress >= obj.count then prog.completed = true end
                changed = true
            end
        end
    end

    if changed then
        QuestUpdatedRemote:FireClient(player, profile.quests)
    end
end

-- Check if all objectives complete (ready to turn in)
function QuestSystem.IsComplete(player: Player, questId: string): boolean
    local profile = PlayerData.Get(player); if not profile then return false end
    for _, active in profile.quests.active do
        if active.id == questId then
            for _, obj in active.objectives do
                if not obj.completed then return false end
            end
            return true
        end
    end
    return false
end

-- Turn in (grant rewards, mark completed, remove from active)
function QuestSystem.TurnIn(player: Player, questId: string): boolean
    if not QuestSystem.IsComplete(player, questId) then return false end

    local def = Quests[questId]
    local profile = PlayerData.Get(player)

    -- Grant rewards
    if def.rewards.coins then profile.coins += def.rewards.coins end
    if def.rewards.xp then grantXp(player, def.rewards.xp) end
    if def.rewards.items then
        for _, itemId in def.rewards.items do grantItem(player, itemId) end
    end

    -- Move active → completed
    for i, active in profile.quests.active do
        if active.id == questId then
            table.remove(profile.quests.active, i)
            break
        end
    end
    profile.quests.completed[questId] = {id = questId, completedAt = os.time()}

    QuestUpdatedRemote:FireClient(player, profile.quests)
    return true
end

-- Abandon (remove from active without completing)
function QuestSystem.Abandon(player: Player, questId: string): boolean
    local profile = PlayerData.Get(player); if not profile then return false end
    for i, active in profile.quests.active do
        if active.id == questId then
            table.remove(profile.quests.active, i)
            QuestUpdatedRemote:FireClient(player, profile.quests)
            return true
        end
    end
    return false
end

return QuestSystem
```

## Hooking into game events

Quest progression hooks into the systems that already exist. Wire up a thin layer that calls `QuestSystem.Progress` from the relevant signals:

```lua
-- Kill objectives — listen to NPC death (see npcs.md)
EnemyDied:Connect(function(killer: Player, enemyType: string)
    QuestSystem.Progress(killer, "Kill", enemyType, 1)
end)

-- Collect objectives — listen to pickup events (see world-mechanics.md)
PickupCollected:Connect(function(player: Player, itemId: string)
    QuestSystem.Progress(player, "Collect", itemId, 1)
end)

-- TalkTo objectives — quest NPCs fire on Triggered (see npcs.md, interactions.md)
NpcTalkedTo:Connect(function(player: Player, npcId: string)
    QuestSystem.Progress(player, "TalkTo", npcId, 1)
end)

-- Reach objectives — region triggers (see collisions.md)
RegionEntered:Connect(function(player: Player, regionId: string)
    QuestSystem.Progress(player, "Reach", regionId, 1)
end)
```

The QuestSystem doesn't subscribe to game events directly — it exposes `Progress(...)` and trusts callers to invoke it. That keeps it decoupled from any specific damage system, pickup system, etc.

## Quest-giver NPC integration

```lua
-- NPC server script (see npcs.md)
prompt.Triggered:Connect(function(player)
    local npcId = npc:GetAttribute("NpcId")    -- "ShepherdNpc"

    -- Is this NPC turning in a quest?
    for _, active in PlayerData.Get(player).quests.active do
        local def = Quests[active.id]
        if def.turnInId == npcId and QuestSystem.IsComplete(player, active.id) then
            TurnInOfferedRemote:FireClient(player, def, active)
            return
        end
    end

    -- Is this NPC offering a new quest?
    local available = {}
    for questId, def in Quests do
        if def.giverId == npcId and QuestSystem.IsAvailable(player, questId) then
            table.insert(available, def)
        end
    end
    if #available > 0 then
        QuestOfferedRemote:FireClient(player, available)
        return
    end

    -- Otherwise just dialog
    DialogRemote:FireClient(player, defaultDialogFor(npcId))
end)
```

The client receives offers/turn-ins and shows the appropriate UI; player accepts/turns-in via remotes that call `QuestSystem.Start` or `QuestSystem.TurnIn` server-side.

## Quest log UI

A standard quest log shows the player's `profile.quests.active` list, with progress bars per objective. The server pushes updates via the `QuestUpdatedRemote` so the client reflects every progress change in real time.

```lua
-- Client
QuestUpdatedRemote.OnClientEvent:Connect(function(quests)
    QuestLogUI.Render(quests.active, quests.completed)
end)

-- Render shows for each active quest:
-- - Name
-- - Each objective's description + "X / Y" progress + checkbox if complete
-- - "Track" button (highlight on map / minimap)
-- - "Abandon" button
```

See [ui-system.md](ui-system.md) for the UI primitives.

## Patterns

### Branching outcomes
After turn-in, fire a `QuestCompleted` BindableEvent with the quest id. Other systems (story progression, world state changes, NPC dialog updates) listen to it.

### Time-limited quests
Add `expiresAt: number?` to ActiveQuest. Periodically (`Heartbeat` at 1Hz) sweep active quests; expired ones move to a `failed` bucket or auto-abandon.

### Daily quests
Reset a subset of completed quests every UTC day, similar to the daily login pattern in [offline-processing.md](offline-processing.md):
```lua
local today = os.date("!*t", os.time()).yday
if profile.lastDailyResetYday ~= today then
    for questId in pairs(profile.quests.completed) do
        if Quests[questId] and Quests[questId].daily then
            profile.quests.completed[questId] = nil
        end
    end
    profile.lastDailyResetYday = today
end
```

### Achievements (one-shot completed-only quests)
Use the same data model with `objectives = {{type = "Custom", targetId = "DefeatedFinalBoss", count = 1}}` and never offer them as active quests — instead, fire `QuestSystem.Progress` from wherever the achievement triggers, and auto-complete + grant when filled. Or fork the system into a separate AchievementSystem that shares the data shape.

### Quest chains
Use `requiresCompleted` to gate quest N on quest N-1. The same NPC offers next-in-chain when previous is in `completed`.

### Hidden objectives
Set `obj.description = ""` (UI hides empty objectives) until a separate trigger reveals it. The server can rewrite the description on a `QuestUpdated` push.

### Multiple concurrent quests on the same target
A player has both "Kill 5 wolves" (WolfHunt) and "Kill 3 wolves" (DailyHunt) active. Both progress simultaneously when a wolf dies. The `QuestSystem.Progress` loop iterates all active quests — single kill ticks both. This is the right behavior; designers control overlap by quest design.

## Pitfalls

- **Quest state stored client-side.** A player can claim "I completed this quest" — the server must own progress and turn-ins.
- **Granting rewards before persisting state change.** If grant succeeds and DataStore save fails, player has the rewards AND the quest still active — they can turn it in again on next save. Save-then-grant or use the `UpdateAsync` callback pattern from [datastores.md](datastores.md).
- **Progress beyond required count.** `prog.progress + amount` could overshoot. `math.min(..., obj.count)` clamps; without it, "5 / 3 wolves" looks broken.
- **Removing from active list while iterating** — classic table-mutation-during-iteration bug. Either iterate backwards (`for i = #t, 1, -1`), or collect indices to remove and process after.
- **Repeated quest accumulating in `completed`** — for repeatable quests, store `completionCount` instead of a single completed entry, or move them to a separate `history` bucket.
- **Quest catalog changed after a player started a quest** — if you delete or rename a quest id, players holding it in `active` get nil lookups. Either keep deprecated quest defs around with `removed = true`, or sweep `active` on load and abandon-with-refund the missing ones.
- **NPC-id mismatch.** Quest definition's `giverId` and the NPC's `NpcId` attribute must match exactly (case-sensitive). Use a constants module to centralize NPC ids.
- **No catalog validation.** A typo in `objectives = {{type = "Kil", ...}}` (missing 'l') means the objective never progresses. Add a startup-time check that walks every quest def and warns on unknown objective types.
- **Per-frame `QuestUpdatedRemote:FireClient`** during a kill streak fires N times. Throttle to ~10 Hz or coalesce.
- **Quest progress lost on respawn or rejoin** — store quest state in the persisted profile, not in per-character state. See [datastores.md](datastores.md).
- **Reach/region objectives via `Touched`** — fires every step a contact exists. Always debounce per-quest-per-player.

## See also
- [multi-stage-quests.md](multi-stage-quests.md) — sequential stages, branching, per-stage world state, quest chains (extends this card's flat-objectives model)
- [datastores.md](datastores.md) — persisting `profile.quests`
- [save-slots.md](save-slots.md) — quest progress is per-slot, not per-player
- [npcs.md](npcs.md) — quest-giver and turn-in NPC composition
- [interactions.md](interactions.md) — ProximityPrompt-driven turn-in
- [shops.md](shops.md) — sister system; both compose with NPCs
- [oop-patterns.md](oop-patterns.md) — state machine for quest progression flow
- [tags.md](tags.md), [attributes.md](attributes.md) — designer-driven quest binding (NpcId, RegionId attributes)
- [collisions.md](collisions.md), [world-mechanics.md](world-mechanics.md) — Reach objective implementation
- [ui-system.md](ui-system.md) — quest log, quest offer modal, turn-in UI
- [security.md](security.md) — server-side quest validation
- [community-libraries.md](community-libraries.md) — ProfileService for the persisted profile
