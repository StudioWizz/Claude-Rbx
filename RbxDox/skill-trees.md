# Skill Trees

> Source: distilled from RPG/MMO skill-tree patterns

## Purpose
A **skill tree** is a directed graph of unlockable nodes — each node grants a passive bonus, an active ability, or a stat boost. Players spend **skill points** (granted by leveling — see [progression.md](progression.md)) to unlock nodes, but only along edges from already-unlocked nodes (the "tree" structure forces commitment to a build path). This card covers the data shape, allocation/respec mechanics, and the canonical UI rendering. For the underlying XP/level loop that grants points, see [progression.md](progression.md). For the active-ability execution that nodes might unlock, see [combat.md](combat.md).

## Tree data

```lua
--!strict
export type SkillNode = {
    id: string,
    name: string,
    description: string,
    iconId: string?,
    cost: number,                          -- skill points required
    maxRanks: number?,                     -- nil = 1 rank only; >1 = stackable node

    -- Position in the tree (for UI rendering)
    x: number,
    y: number,

    -- Prerequisites: must have at least one of these unlocked
    requires: {string}?,
    requiresAll: boolean?,                 -- true = AND; false (default) = OR

    -- What it grants
    onUnlock: ((profile: PlayerProfile, rank: number) -> ())?,
    statBonus: {[string]: number}?,        -- additive, per rank
}

export type SkillTree = {
    id: string,
    name: string,
    nodes: {[string]: SkillNode},
    rootId: string,                        -- starting node (always free; or first in chain)
}

local SkillTrees = {}

SkillTrees.Warrior = {
    id = "Warrior", name = "Warrior",
    rootId = "warrior_root",
    nodes = {
        warrior_root = {
            id = "warrior_root", name = "Warrior", description = "Begin the warrior path.",
            cost = 0, x = 0, y = 0,
        },
        toughness = {
            id = "toughness", name = "Toughness", description = "+10 max HP per rank.",
            cost = 1, maxRanks = 5, x = -1, y = 1,
            requires = {"warrior_root"},
            statBonus = {maxHealth = 10},
        },
        cleave = {
            id = "cleave", name = "Cleave", description = "Unlocks the Cleave ability (AOE attack).",
            cost = 2, x = 0, y = 1,
            requires = {"warrior_root"},
            onUnlock = function(profile, _) profile.unlockedAbilities.Cleave = true end,
        },
        berserker = {
            id = "berserker", name = "Berserker", description = "+20% damage when below 50% HP.",
            cost = 3, x = 0, y = 2,
            requires = {"toughness", "cleave"},
            requiresAll = true,
        },
    },
}

return SkillTrees
```

The `x`/`y` fields aren't gameplay — they tell the UI where to draw each node. Edges are inferred from `requires`.

## Profile shape

```lua
profile.skillPoints = 0                                   -- granted by leveling
profile.unlockedNodes = {} :: {[string]: number}          -- nodeId → rank (1 for single-rank, 1..maxRanks for stackable)
profile.unlockedAbilities = {} :: {[string]: boolean}     -- side-effect of nodes that grant abilities
```

Skill points are granted on level up:
```lua
-- In progression.md onLevelUp:
profile.skillPoints += SKILL_POINTS_PER_LEVEL
```

## Allocate a point

```lua
local SkillTrees = require(game.ReplicatedStorage.SkillTrees)

local Skills = {}

function Skills.CanAllocate(profile: PlayerProfile, treeId: string, nodeId: string): (boolean, string?)
    local tree = SkillTrees[treeId]
    if not tree then return false, "no-such-tree" end
    local node = tree.nodes[nodeId]
    if not node then return false, "no-such-node" end

    local currentRank = profile.unlockedNodes[nodeId] or 0
    local maxRanks = node.maxRanks or 1
    if currentRank >= maxRanks then return false, "max-rank" end

    if profile.skillPoints < node.cost then return false, "not-enough-points" end

    -- Prereq check
    if node.requires then
        local satisfied = if node.requiresAll then allReqs(node.requires, profile) else anyReq(node.requires, profile)
        if not satisfied then return false, "prereqs-missing" end
    end

    return true, nil
end

function Skills.Allocate(profile: PlayerProfile, treeId: string, nodeId: string): (boolean, string?)
    local ok, err = Skills.CanAllocate(profile, treeId, nodeId)
    if not ok then return false, err end

    local tree = SkillTrees[treeId]
    local node = tree.nodes[nodeId]

    profile.skillPoints -= node.cost
    profile.unlockedNodes[nodeId] = (profile.unlockedNodes[nodeId] or 0) + 1

    if node.onUnlock then
        node.onUnlock(profile, profile.unlockedNodes[nodeId])
    end

    -- Re-aggregate stats (for statBonus contributions)
    Equipment.RecomputeStats(profile)
    return true, nil
end

local function anyReq(reqs: {string}, profile: PlayerProfile): boolean
    for _, r in reqs do if (profile.unlockedNodes[r] or 0) > 0 then return true end end
    return false
end
local function allReqs(reqs: {string}, profile: PlayerProfile): boolean
    for _, r in reqs do if (profile.unlockedNodes[r] or 0) == 0 then return false end end
    return true
end

return Skills
```

The remote handler:

```lua
AllocateSkillRemote.OnServerEvent:Connect(function(player, treeId, nodeId)
    if typeof(treeId) ~= "string" or typeof(nodeId) ~= "string" then return end
    local profile = PlayerData.Get(player); if not profile then return end
    local ok, err = Skills.Allocate(profile, treeId, nodeId)
    SkillResultRemote:FireClient(player, {ok = ok, err = err, nodeId = nodeId})
end)
```

## Stat aggregation from skill nodes

Add a per-node statBonus into the player's derived stats. This integrates with the same `Equipment.RecomputeStats` aggregation pattern:

```lua
function Skills.AggregateBonuses(profile: PlayerProfile): {[string]: number}
    local agg: {[string]: number} = {}
    for treeId, tree in SkillTrees do
        for nodeId, node in tree.nodes do
            local rank = profile.unlockedNodes[nodeId] or 0
            if rank > 0 and node.statBonus then
                for stat, value in node.statBonus do
                    agg[stat] = (agg[stat] or 0) + value * rank
                end
            end
        end
    end
    return agg
end
```

In `RecomputeStats`, sum gear bonuses + skill bonuses + base attributes.

## Respec (refund all points)

```lua
function Skills.Respec(profile: PlayerProfile, treeId: string?, costToRespec: number?)
    if costToRespec and profile.coins < costToRespec then return false end
    if costToRespec then profile.coins -= costToRespec end

    local refund = 0
    for nodeId, rank in profile.unlockedNodes do
        local tree = nil
        for _, t in SkillTrees do
            if t.nodes[nodeId] then tree = t; break end
        end
        if tree and (not treeId or tree.id == treeId) then
            local node = tree.nodes[nodeId]
            refund += node.cost * rank
            profile.unlockedNodes[nodeId] = nil

            -- If the node granted an ability, revoke it
            if node.onUnlock then
                -- Run a corresponding revoke if you defined one
                local revoke = node.onRevoke
                if revoke then revoke(profile) end
            end
        end
    end

    profile.skillPoints += refund
    Equipment.RecomputeStats(profile)
    return true
end
```

For idempotent re-allocation, store `onRevoke` per-node OR re-derive abilities from the unlocked-nodes set on every change rather than maintaining a separate `unlockedAbilities` table.

## UI rendering

The skill tree UI is the trickiest part — drawing nodes at their (x, y) positions with edges between connected nodes:

- Use a `ScrollingFrame` (or `Frame` with custom pan/zoom) with absolute pixel positioning.
- For each node, place a `Frame` at `UDim2.fromOffset(node.x * SCALE, node.y * SCALE)`.
- For each edge (`require` connection), draw a line — easiest with a thin rotated `Frame`, harder with a `UIStroke` or curve.
- Color nodes by state: locked (gray), available (highlighted), unlocked (lit-up).

```lua
-- Pseudo-code for rendering one node
local function renderNode(node: SkillNode, profile: PlayerProfile, parent: ScrollingFrame)
    local frame = Instance.new("Frame", parent)
    frame.Position = UDim2.fromOffset(node.x * 80, node.y * 80)
    frame.Size = UDim2.fromOffset(64, 64)

    local rank = profile.unlockedNodes[node.id] or 0
    local canAllocate = Skills.CanAllocate(profile, treeId, node.id)
    if rank > 0 then frame.BackgroundColor3 = Color3.new(0.2, 0.8, 0.3)
    elseif canAllocate then frame.BackgroundColor3 = Color3.new(0.5, 0.5, 0.9)
    else frame.BackgroundColor3 = Color3.new(0.3, 0.3, 0.3) end

    -- Click → AllocateSkillRemote:FireServer(treeId, node.id)
end
```

For node icons, name, current rank, hover-tooltip with description and statBonus preview, see [ui-system.md](ui-system.md).

## Patterns

### Tier locks
"Tier 2 nodes require N points spent in this tree." Easy implementation: count total ranks in the tree, check >= threshold.

### Mutually exclusive choices
"Pick exactly one of these three at this depth." Allocate one → mark the others as `excluded` (a runtime-computed flag the UI shows as red).

### Multiple trees per player
A player has access to several trees (Warrior + Mage + Crafting). Allocate from a shared point pool, OR one pool per tree (gain different point types from different XP sources).

### Auto-loadout per tree
Saved skill loadouts (similar to equipment loadouts in [equipment.md](equipment.md)) — switch between "PvE build" and "PvP build" with a click. Limit swap frequency to prevent abuse.

## Pitfalls

- **Allocating a node and the prereq check passes but the prereq is at rank 0** (because `unlockedNodes[r] = 0` reads as falsy in some checks). Always compare `>= 1` explicitly.
- **Refund spans tree but you can't tell which tree a node belongs to.** Index nodes by tree at load (a `nodeIdToTreeId` lookup) instead of scanning every tree on every refund.
- **Stat aggregation re-walks the whole tree per Heartbeat.** Recompute on allocate/respec only. Cache in `profile.derivedStats`.
- **`onUnlock` side-effects (e.g., add an ability) not reversed on respec.** Either define an `onRevoke` per node, or re-derive all unlocks from `unlockedNodes` on every change (slower but simpler).
- **Stackable node maxRanks not enforced.** Player allocates 100 ranks in "Toughness" → +1000 HP. Cap.
- **Skill-tree data sent to client unchanged each open.** Catalogs in `ReplicatedStorage` replicate once on join; don't re-send on every UI open.
- **Respec without re-aggregating gear stats.** The respec strips skill bonuses but leaves stale `derivedStats` until the next equip event. Always Recompute after respec.
- **Skill points becoming negative.** A bug in cost subtraction → negative balance. Floor at 0 with a warn.
- **No protection against allocating during combat.** Live-respec mid-fight is exploit territory. Lock allocate to safe zones / out-of-combat.
- **Hard-coded UI positions in code.** Ugly to update. Keep `x`/`y` in the data and let the UI compute layout.
- **No save versioning.** When you change a node's cost or remove a node, players' saves point to non-existent entries. Migrate on load (drop unknown nodes, refund their points).

## See also
- [progression.md](progression.md) — XP and level grants the skill points this card spends
- [equipment.md](equipment.md) — same stat-aggregation pattern
- [combat.md](combat.md) — abilities unlocked by nodes execute via combat
- [datastores.md](datastores.md) — persisting `unlockedNodes` and `skillPoints`
- [ui-system.md](ui-system.md) — tree rendering with positioned nodes and connecting lines
- [security.md](security.md) — server-validated allocate
- [save-slots.md](save-slots.md) — skill trees are per active slot
- [community-libraries.md](community-libraries.md) — `t` for runtime payload validation
