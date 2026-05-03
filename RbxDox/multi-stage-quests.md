# Multi-Stage Quests

> Source: distilled from RPG / story-game multi-act quest patterns

## Purpose
`quests.md` covers **flat** quest objectives — a list, all parallel, complete-them-all-then-turn-in. Real RPG questlines are richer: **sequential stages** that unlock one at a time, **per-stage world state** (this NPC appears now; that door opens), **dynamic objectives** that materialize after the previous one resolves, **branching paths** ("save the prince OR betray him"), **dialog interludes between stages**, and **quest chains** stitching multiple quest definitions into multi-hour storylines. This card extends `quests.md` with those patterns. Read `quests.md` first; this card assumes its data shape and adds a `stages` layer on top.

## Stage data shape

Each stage is a self-contained mini-quest with its own objectives, dialog, and world-state hooks:

```lua
--!strict
export type StageObjective = {
    id: string,                                      -- unique within the stage
    description: string,
    type: "Kill" | "Collect" | "TalkTo" | "Reach" | "Use" | "Custom",
    targetId: string,
    count: number,
    optional: boolean?,                              -- if true, completing all required is enough
}

export type Stage = {
    id: string,
    name: string,
    intro: string?,                                  -- dialog shown when this stage begins
    objectives: {StageObjective},
    onEnter: ((player: Player) -> ())?,              -- called when the stage activates
    onExit: ((player: Player, completed: boolean) -> ())?,
    nextStageId: string?,                            -- linear; nil = quest ends after this stage
    branches: {{label: string, nextStageId: string, condition: ((player: Player) -> boolean)?}}?,  -- choices
    rewards: QuestReward?,                           -- per-stage rewards (optional)
}

export type MultiStageQuestDef = {
    id: string,
    name: string,
    description: string,
    giverId: string?,
    requiresLevel: number?,
    requiresCompleted: {string}?,
    startStageId: string,
    stages: {[string]: Stage},                       -- keyed by stage id
    finalRewards: QuestReward,                       -- granted when the whole questline completes
}
```

## Example: a 4-stage questline with branching

```lua
local Quests = {}

Quests.LostPrince = {
    id = "LostPrince",
    name = "The Lost Prince",
    description = "The prince has gone missing in the haunted forest.",
    giverId = "QueenNpc",
    requiresLevel = 8,
    startStageId = "investigate",

    stages = {
        investigate = {
            id = "investigate", name = "Investigate the Forest",
            intro = "Search the forest for clues about the prince's fate.",
            objectives = {
                {id = "talk_to_woodsman", description = "Talk to the Old Woodsman",
                 type = "TalkTo", targetId = "WoodsmanNpc", count = 1},
                {id = "find_torn_cloak", description = "Find the prince's torn cloak",
                 type = "Collect", targetId = "TornCloak", count = 1},
            },
            onEnter = function(player)
                spawnQuestNpc("WoodsmanNpc", forestGladePosition)
                spawnQuestItem("TornCloak", witchHutPosition)
            end,
            onExit = function(player) despawnQuestNpc("WoodsmanNpc") end,
            nextStageId = "find_witch",
        },

        find_witch = {
            id = "find_witch", name = "Confront the Witch",
            intro = "The cloak is hers. Find her hut deep in the forest.",
            objectives = {
                {id = "reach_witch_hut", description = "Reach the Witch's Hut",
                 type = "Reach", targetId = "WitchHutRegion", count = 1},
                {id = "talk_to_witch", description = "Speak with the Witch",
                 type = "TalkTo", targetId = "WitchNpc", count = 1},
            },
            onEnter = function(player)
                spawnQuestNpc("WitchNpc", witchHutPosition)
                openSecretPath("WitchHutPath")
            end,
            -- BRANCHING: player chooses to fight or pay her
            branches = {
                {label = "Fight the Witch", nextStageId = "fight_witch"},
                {label = "Pay her gold (500 coins)", nextStageId = "pay_witch",
                 condition = function(p) return getProfile(p).coins >= 500 end},
            },
        },

        fight_witch = {
            id = "fight_witch", name = "Defeat the Witch",
            intro = "She refuses. There's no other way.",
            objectives = {
                {id = "kill_witch", description = "Defeat the Witch",
                 type = "Kill", targetId = "WitchNpc", count = 1},
            },
            onEnter = function(player) makeNpcHostile("WitchNpc", player) end,
            nextStageId = "rescue_prince",
        },

        pay_witch = {
            id = "pay_witch", name = "Pay the Witch",
            intro = "You hand over the gold. She nods.",
            objectives = {},                          -- no objectives; auto-advance after onEnter
            onEnter = function(player)
                getProfile(player).coins -= 500
                despawnQuestNpc("WitchNpc")
            end,
            nextStageId = "rescue_prince",
        },

        rescue_prince = {
            id = "rescue_prince", name = "Free the Prince",
            intro = "The cage is in her cellar. Free him.",
            objectives = {
                {id = "talk_to_prince", description = "Free the prince and talk to him",
                 type = "TalkTo", targetId = "PrinceNpc", count = 1},
                {id = "escort_to_castle", description = "Escort him back to the castle",
                 type = "Reach", targetId = "CastleGateRegion", count = 1, optional = true},
            },
            onEnter = function(player)
                spawnQuestNpc("PrinceNpc", witchCellarPosition)
                openCage("WitchCage")
            end,
        },
    },

    finalRewards = {coins = 1000, xp = 800, items = {"RoyalSeal", "PrinceCape"}},
}

return Quests
```

Two paths through the questline:
- `investigate` → `find_witch` → **`fight_witch`** → `rescue_prince` (combat path)
- `investigate` → `find_witch` → **`pay_witch`** → `rescue_prince` (peaceful path)

Both converge at the final stage. Different outcomes (XP, faction reputation, cosmetics) could vary based on the path taken — track via `branchHistory` on the quest's active state.

## Per-player state

Extending the active-quest shape from `quests.md`:

```lua
export type ActiveMultiStageQuest = {
    id: string,
    currentStageId: string,
    objectiveProgress: {[string]: {progress: number, completed: boolean}},  -- keyed by objective id within the stage
    branchHistory: {string},                          -- choices made; for outcome tracking
    startedAt: number,
}

profile.quests.active = {} :: {ActiveMultiStageQuest}
profile.quests.completed = {} :: {[string]: {id: string, completedAt: number, branchHistory: {string}}}
```

The flat `objectives` array from `quests.md` is replaced with `currentStageId` + per-stage `objectiveProgress`. As stages advance, the stage id changes; only the current stage's objectives are tracked.

## Stage advancement

```lua
local Quests = require(game.ReplicatedStorage.Quests)
local QuestSystem = {}

function QuestSystem.IsStageComplete(active: ActiveMultiStageQuest): boolean
    local def = Quests[active.id]; if not def then return false end
    local stage = def.stages[active.currentStageId]; if not stage then return false end

    for _, obj in stage.objectives do
        if obj.optional then continue end
        local p = active.objectiveProgress[obj.id]
        if not p or not p.completed then return false end
    end
    return true
end

function QuestSystem.AdvanceStage(player: Player, active: ActiveMultiStageQuest, chosenBranchId: string?)
    local def = Quests[active.id]; if not def then return end
    local currentStage = def.stages[active.currentStageId]; if not currentStage then return end

    -- Run exit hook for current stage
    if currentStage.onExit then currentStage.onExit(player, true) end

    -- Per-stage rewards
    if currentStage.rewards then grantRewards(player, currentStage.rewards) end

    -- Determine next stage
    local nextStageId = currentStage.nextStageId
    if currentStage.branches and chosenBranchId then
        for _, b in currentStage.branches do
            if b.label == chosenBranchId or b.nextStageId == chosenBranchId then
                if not b.condition or b.condition(player) then
                    nextStageId = b.nextStageId
                    table.insert(active.branchHistory, chosenBranchId)
                    break
                end
            end
        end
    end

    if not nextStageId then
        -- End of questline
        return QuestSystem.CompleteQuest(player, active)
    end

    -- Activate next stage
    active.currentStageId = nextStageId
    active.objectiveProgress = {}
    local nextStage = def.stages[nextStageId]

    if nextStage.onEnter then nextStage.onEnter(player) end
    QuestUpdatedRemote:FireClient(player, profile.quests)

    -- Show intro dialog if any
    if nextStage.intro then
        QuestDialogRemote:FireClient(player, {kind = "StageIntro", text = nextStage.intro})
    end
end

function QuestSystem.CompleteQuest(player: Player, active: ActiveMultiStageQuest)
    local def = Quests[active.id]
    grantRewards(player, def.finalRewards)

    local profile = PlayerData.Get(player)
    -- Remove from active
    for i, q in profile.quests.active do
        if q.id == active.id then table.remove(profile.quests.active, i); break end
    end
    -- Add to completed
    profile.quests.completed[active.id] = {
        id = active.id,
        completedAt = os.time(),
        branchHistory = active.branchHistory,
    }
    QuestUpdatedRemote:FireClient(player, profile.quests)
end
```

## Objective progression hook

The same shape as `quests.md` — events fire `QuestSystem.Progress(player, type, targetId)` and the system finds the matching active quest's *current stage* and ticks the objective:

```lua
function QuestSystem.Progress(player: Player, objType: string, targetId: string, amount: number?)
    amount = amount or 1
    local profile = PlayerData.Get(player); if not profile then return end

    for _, active in profile.quests.active do
        local def = Quests[active.id]; if not def then continue end
        local stage = def.stages[active.currentStageId]; if not stage then continue end

        for _, obj in stage.objectives do
            if obj.type ~= objType or obj.targetId ~= targetId then continue end
            local prog = active.objectiveProgress[obj.id] or {progress = 0, completed = false}
            if prog.completed then continue end

            prog.progress = math.min(obj.count, prog.progress + amount)
            if prog.progress >= obj.count then prog.completed = true end
            active.objectiveProgress[obj.id] = prog
        end

        -- Check if the stage is now complete (no branches → auto-advance)
        if QuestSystem.IsStageComplete(active) then
            local currentStage = def.stages[active.currentStageId]
            if currentStage.branches then
                -- Wait for player choice via UI
                BranchChoiceRemote:FireClient(player, {
                    questId = active.id,
                    choices = currentStage.branches,
                })
            else
                QuestSystem.AdvanceStage(player, active)
            end
        end
    end

    QuestUpdatedRemote:FireClient(player, profile.quests)
end
```

For branching stages, the system pauses and waits for the player to choose. The client shows a choice UI (modal dialog with branch buttons), then fires the chosen branch back:

```lua
BranchChosenRemote.OnServerEvent:Connect(function(player, questId, branchLabel)
    if typeof(questId) ~= "string" or typeof(branchLabel) ~= "string" then return end
    local profile = PlayerData.Get(player); if not profile then return end
    for _, active in profile.quests.active do
        if active.id == questId then
            QuestSystem.AdvanceStage(player, active, branchLabel)
            return
        end
    end
end)
```

## Per-stage world state

The `onEnter` and `onExit` hooks let each stage modify the world: spawn an NPC, open a door, change a region's lighting, enable a previously-hidden quest item.

Two patterns for "world state changes per stage":

### Pattern A: Per-player instanced
Each player has their *own copy* of the world's quest state (their NPC spawned just for them, visible only to them). Use a per-player `Folder` under `workspace.PlayerInstances.<UserId>` and parent the stage's spawned content there. Achieved by Filtering the `Parent` on a per-player basis with custom replication, OR by using `:SetAttribute` flags the client filters on.

Best for: questlines where multiple players doing the same quest shouldn't interfere.

### Pattern B: Server-authoritative shared
The world has one canonical state; quest changes affect everyone. The first player to start the quest sees the witch spawn; everyone in the server can interact. Subsequent players entering the area see the witch already there.

Best for: dungeon-style content; co-op quests; one-quest-per-server.

For most game styles, **Pattern A is the safer default** — quest-specific NPCs and items shouldn't bleed into other players' experiences.

For instanced spawns:
```lua
local function spawnQuestNpc(npcId, position, owner: Player)
    local npc = NPC_TEMPLATES[npcId]:Clone()
    npc:PivotTo(CFrame.new(position))
    npc:SetAttribute("VisibleTo", owner.UserId)
    npc.Parent = workspace.PlayerInstances:FindFirstChild(tostring(owner.UserId))
        or createPlayerInstanceFolder(owner.UserId)
end
```

The client filters: when iterating world NPCs, skip any with `VisibleTo` ≠ LocalPlayer.UserId. (Or use Roblox's per-player replication features if available.)

## Dialog interludes

A stage's `intro` field is a single dialog line. For longer cinematic interludes (multiple lines, auto-advance, NPC speaking with character animation), define a richer dialog format:

```lua
type DialogScript = {
    {speaker: string, text: string, durationSec: number?, animation: string?},
    -- ...
}

stage.dialog = {
    {speaker = "Queen", text = "My son... has been gone for three days.", durationSec = 4},
    {speaker = "Queen", text = "Find him. Bring him home. I beg you.", durationSec = 4, animation = "Plead"},
}
```

On `onEnter`, fire the dialog script to the client; the client plays it line-by-line with timing. See [npcs.md](npcs.md) dialog patterns and [menu-systems.md](menu-systems.md) for the dialog UI.

For full cutscenes (camera control, character motion), trigger a separate cutscene system from `onEnter`. See [camera.md](camera.md) for camera CFrame control.

## Mid-quest persistence

Quest state must survive disconnect → rejoin. The `profile.quests.active` table already covers this (saved with the rest of the profile via `datastores.md`). When the player rejoins:

```lua
Players.PlayerAdded:Connect(function(player)
    local profile = PlayerData.Get(player)
    -- Re-trigger onEnter for each active quest's current stage
    for _, active in profile.quests.active do
        local def = Quests[active.id]
        local stage = def and def.stages[active.currentStageId]
        if stage and stage.onEnter then
            stage.onEnter(player)
        end
    end
end)
```

This re-spawns quest NPCs, re-opens doors, etc. — restoring the world state the player needs to continue.

For quests that involve other players (escort the prince — but only if you're playing alone, or the prince is your responsibility), persist who's the "owner" and re-spawn into their personal world state.

## Quest chains across multiple definitions

A multi-stage quest is one definition with multiple stages. A **quest chain** is multiple definitions linked by prerequisites. Use `requiresCompleted`:

```lua
Quests.LostPrincePartTwo = {
    id = "LostPrincePartTwo",
    name = "The Prince's Secret",
    requiresCompleted = {"LostPrince"},
    -- ...
}
```

The questgiver only offers `LostPrincePartTwo` to players who have `LostPrince` in their `completed` table. See `quests.md` Patterns section.

For very long chains (10+ quests), wrap in a "saga" concept:
```lua
local SAGAS = {
    LostPrinceSaga = {
        ordered = {"LostPrince", "LostPrincePartTwo", "LostPrinceFinale"},
        sagaCompleteRewards = {coins = 5000, xp = 5000, items = {"PrinceTitle"}},
    },
}
```

Track saga completion as the last quest in the saga becoming `completed` — grant additional rewards then.

## UI implications

The standard quest log (from `quests.md`) needs a few additions:
- **Stage progress** instead of objective list — show "Stage 3 of 5: Confront the Witch"
- **Per-stage objectives** — only the current stage's objectives are visible
- **Branching choice modal** — pops up when the player must choose
- **Stage history** (optional) — let the player review what they've done so far
- **Saga progress bar** — for chained quests, show "3/5 quests complete"

See [menu-systems.md](menu-systems.md) for the UI shell and [ui-system.md](ui-system.md) for the primitives.

## Patterns

### Time-limited stages
A stage that auto-fails if not completed in N minutes. Track `stageStartedAt` in active state; sweep on Heartbeat and fail-advance to a "failed" stage if expired.

### Failed stages
A separate stage path for "you took too long" or "the prince died." Often a shorter alternative ending with reduced rewards.

### Multi-objective gating within a stage
A stage with several objectives where some are required (kill the witch) and some are optional (free the cat too — bonus reward). Use `optional = true` on the optional ones; complete only the required ones to advance.

### Permanent world changes after questline
Some quests should change the world *permanently* (the witch's hut burns down; the prince becomes a recurring NPC at the castle). Persist via global DataStore key, not per-player. Other quests might offer to "restart from a save" — store snapshots.

### Replayable / repeatable questlines
For repeatable storylines (daily dungeons, weekly raids), set `repeatable = true` on the def. On `Complete`, immediately re-add to available, but track `completionCount` separately so rewards diminish.

## Pitfalls

- **Stage objectives keep progressing across stages.** A "Kill 5 wolves" objective in Stage 1 keeps counting after advancing to Stage 2 (since wolves are still being killed). Reset `objectiveProgress = {}` on advance.
- **Branch chosen by client without server validation.** Player picks a "free 1000 gold" branch they shouldn't have access to. Server validates `condition` before honoring.
- **`onEnter` runs before client UI is ready.** Stage-intro dialog fires but the player's quest log hasn't loaded the new stage. Send the stage update first; defer `onEnter` dialog fire until the next frame, or send both atomically.
- **Quest NPC spawned in shared world but visible to wrong player.** Use Pattern A (per-player instanced) for quest-specific spawns.
- **Re-spawning quest NPCs on rejoin without checking the current stage.** Player on Stage 3 rejoins → onEnter for Stage 1 fires (re-spawning the wrong NPC). Only fire `onEnter` for the *current* stage.
- **Branching paths with no UI cue.** Player completes the stage; nothing happens; they're confused. Always pop a branch choice modal explicitly.
- **Quest NPC despawned but the player is still mid-dialog with them.** Order matters in `onExit` — wait for any active dialog to finish before despawning.
- **Per-stage rewards granted but quest ends abruptly.** A stage with `nextStageId = nil` should call `CompleteQuest` for `finalRewards`. Don't double-grant by also having `rewards` on the final stage unless intentional.
- **Quest chain prerequisites becoming circular.** Quest A requires B; B requires A. Validate at quest-load time.
- **Saving the entire world's quest state to one DataStore key.** Bloats. Per-player quest progress in profile; per-quest "world state" in its own keys if shared.
- **`onEnter` hooks that yield (yield in a hook delays stage activation).** Don't block; `task.spawn` if you need to do async work.
- **Stage auto-advance on `Reach` objective with a single-step trigger.** Player triggers the gate, advances; gate fires Touched again from a different part of the character; double-advances. Use the `crossedProgress` dedupe pattern from `time-trials.md`.

## See also
- [quests.md](quests.md) — flat-objectives quest system this card extends
- [npcs.md](npcs.md) — quest NPCs spawned per stage; dialog interludes
- [interactions.md](interactions.md) — ProximityPrompt for quest-NPC dialog
- [datastores.md](datastores.md) — persisting `profile.quests.active` mid-quest
- [menu-systems.md](menu-systems.md) — quest log UI, branch choice modal
- [ui-system.md](ui-system.md) — quest UI primitives
- [camera.md](camera.md) — cinematic interludes
- [tweens-animation.md](tweens-animation.md) — dialog text animation
- [tags.md](tags.md), [attributes.md](attributes.md) — designer-driven quest item / region tagging
- [time-trials.md](time-trials.md) — dedupe pattern for trigger-driven objective progress
- [areas-and-biomes.md](areas-and-biomes.md) — Reach objectives and per-area quest content
- [save-slots.md](save-slots.md) — quest progress is per active slot
- [security.md](security.md) — server-authoritative branch selection
