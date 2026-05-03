# NPCs (Non-Player Characters)

> Source: distilled from Humanoid + Pathfinding + interaction patterns and standard NPC composition idioms

## Purpose
An "NPC" isn't a single Roblox class — it's a composition of primitives: a `Model` containing a `Humanoid` (so it can move and take damage like a player character), plus interaction (`ProximityPrompt` for dialog/vendor, or AI scripts for hostile behavior), plus visual identity (avatar from `HumanoidDescription`, or a custom mesh/model). This card pulls together the standard friendly-NPC and hostile-NPC compositions so you don't have to reassemble them from `characters.md` + `interactions.md` + `oop-patterns.md` every time.

## Anatomy of a friendly NPC (vendor, quest-giver, dialog)

```
ShopkeeperNPC (Model, PrimaryPart = HumanoidRootPart)
├── HumanoidRootPart (BasePart, anchored OR network-owned by server)
├── Head, Torso, ...          (BaseParts; can be a player rig OR a custom mesh)
├── Humanoid                   (so they look "alive" — animator, name display)
│   └── Animator
├── ProximityPrompt            (parented under HumanoidRootPart or Head)
│   └── (ActionText = "Talk", HoldDuration = 0)
├── Server Script              (handles Triggered → opens dialog UI on player)
└── Configuration (Folder)     (designer-tunable — see Attributes pattern)
```

For a vendor: replace the dialog logic with shop-open. For a quest-giver: connect to the quest system.

## Anatomy of a hostile NPC (enemy, mob)

```
ZombieNPC (Model, PrimaryPart = HumanoidRootPart)
├── HumanoidRootPart           (server-owned via :SetNetworkOwner(nil))
├── Head, Torso, ...
├── Humanoid                   (server-driven — pathfinding, attacks)
│   └── Animator
├── Server Script              (state machine: Idle → Patrol → Chase → Attack → Dead)
└── Configuration              (Damage, ChaseRadius, AttackRange attributes)
```

The key difference: hostile NPCs need **server network ownership** (so the player can't manipulate them), and a state machine to drive behavior. Friendly NPCs are typically anchored or stationary and don't need ownership concerns.

## Server-side anchoring vs network ownership

| Style | Setup | Behavior |
|---|---|---|
| **Anchored stationary NPC** (vendor, quest-giver) | `HumanoidRootPart.Anchored = true` | Doesn't move, can't be pushed, no physics cost. Ideal for shopkeepers. |
| **Server-owned moving NPC** (patrol, enemy) | `HumanoidRootPart.Anchored = false` + `:SetNetworkOwner(nil)` | Server simulates physics. Slightly higher cost, but trustworthy. **Required for damage-dealing or movement-critical NPCs.** |
| **Client-owned NPC** (default for unanchored) | nothing — engine assigns nearest player | Smooth for that player but they can manipulate it. **Almost never what you want.** |

## Friendly NPC: dialog pattern (linear)

Linear dialog is the simplest: list of lines, advance with E, close at end.

```lua
-- Server Script under the NPC model
local CollectionService = game:GetService("CollectionService")

local npc = script.Parent
local prompt = npc:WaitForChild("HumanoidRootPart"):WaitForChild("ProximityPrompt")
local dialogRemote = game.ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("Dialog")

local LINES = {
    "Welcome, traveler!",
    "I run the local apothecary.",
    "Have a look at my wares.",
}

prompt.Triggered:Connect(function(player)
    -- Don't trust client to handle line advancement; just push the whole script.
    dialogRemote:FireClient(player, {
        npcName = "Apothecary Mira",
        lines = LINES,
        endAction = "openShop",   -- optional: tells the client what to do when dialog ends
    })
end)
```

The client receives the lines, walks the player through them with a UI panel (TextLabel + click-to-continue), then dispatches the `endAction` (open shop, accept quest, etc.).

## Friendly NPC: dialog pattern (branching with choices)

Branching dialog has nodes with multiple outgoing edges. Best authored as a data structure (table-of-tables) that the server interprets:

```lua
local DIALOG_TREE = {
    start = {
        text = "Greetings, hero. What brings you?",
        choices = {
            {label = "Tell me about this place", next = "lore"},
            {label = "Have any work?", next = "quest"},
            {label = "Goodbye", next = nil},
        },
    },
    lore = {
        text = "This village has stood for a thousand years...",
        choices = {{label = "Interesting.", next = "start"}},
    },
    quest = {
        text = "Aye, the wolves have been bothering my flock. Slay 5 and I'll reward you.",
        choices = {
            {label = "Accept", next = nil, action = "acceptQuest:WolfHunt"},
            {label = "Maybe later", next = nil},
        },
    },
}

prompt.Triggered:Connect(function(player)
    dialogRemote:InvokeClient(player, "start", DIALOG_TREE)
    -- Or send only the start node and walk forward via more remote calls
end)
```

Designer-friendly approach: instead of authoring trees in Lua, store them in a ModuleScript per NPC, or even as JSON in `attributes`. See [attributes.md](attributes.md), [tags.md](tags.md).

## Friendly NPC: vendor (calls into shops.md)

```lua
prompt.Triggered:Connect(function(player)
    openShopRemote:FireClient(player, {
        shopId = "apothecary",
        catalog = AppoShopCatalog,
    })
end)
```

The shop UI is opened on the client; purchase intents come back to the server for validation. See [shops.md](shops.md).

## Friendly NPC: quest-giver (calls into quests.md)

```lua
prompt.Triggered:Connect(function(player)
    local availableQuests = QuestSystem.GetAvailable(player, "ApothecaryNpcId")
    if #availableQuests == 0 then
        dialogRemote:FireClient(player, {lines = {"Nothing for you today."}})
        return
    end
    questOfferRemote:FireClient(player, availableQuests)
end)
```

See [quests.md](quests.md).

## Hostile NPC: state machine

Use the pattern from [oop-patterns.md](oop-patterns.md):

```lua
--!strict
local NPC = {}
NPC.__index = NPC

type State = "Idle" | "Patrol" | "Chase" | "Attack" | "Dead"

function NPC.new(model: Model)
    local self = setmetatable({
        model = model,
        humanoid = model:FindFirstChildOfClass("Humanoid") :: Humanoid,
        hrp = model:FindFirstChild("HumanoidRootPart") :: BasePart,
        state = "Idle" :: State,
        target = nil :: Player?,
        lastAttack = 0,
    }, NPC)

    self.hrp:SetNetworkOwner(nil)   -- server owns this NPC's physics

    self.humanoid.Died:Connect(function() self:Transition("Dead") end)
    return self
end

local CHASE_RADIUS = 30
local ATTACK_RANGE = 5
local DAMAGE = 10
local ATTACK_COOLDOWN = 1

local handlers = {
    Idle = function(self, dt)
        local nearest, dist = findNearestPlayer(self.hrp.Position, CHASE_RADIUS)
        if nearest then self.target = nearest; self:Transition("Chase") end
    end,

    Chase = function(self, dt)
        if not self.target or not self.target.Character then self:Transition("Idle"); return end
        local targetHrp = self.target.Character:FindFirstChild("HumanoidRootPart")
        if not targetHrp then self:Transition("Idle"); return end

        local dist = (targetHrp.Position - self.hrp.Position).Magnitude
        if dist <= ATTACK_RANGE then self:Transition("Attack"); return end
        if dist > CHASE_RADIUS * 1.5 then self:Transition("Idle"); return end

        self.humanoid:MoveTo(targetHrp.Position)
    end,

    Attack = function(self, dt)
        if not self.target or not self.target.Character then self:Transition("Idle"); return end
        local now = os.clock()
        if now - self.lastAttack < ATTACK_COOLDOWN then return end
        self.lastAttack = now

        local hum = self.target.Character:FindFirstChildOfClass("Humanoid")
        if hum and (self.target.Character.HumanoidRootPart.Position - self.hrp.Position).Magnitude <= ATTACK_RANGE then
            hum:TakeDamage(DAMAGE)
        else
            self:Transition("Chase")
        end
    end,

    Dead = function(self, dt) end,
}

function NPC:Tick(dt: number) handlers[self.state](self, dt) end
function NPC:Transition(to: State) self.state = to end

function NPC:Destroy()
    self.model:Destroy()
end

return NPC
```

Drive every NPC's Tick from one Heartbeat loop:
```lua
local npcs: {[Model]: NPC} = {}

for _, model in CollectionService:GetTagged("Enemy") do
    npcs[model] = NPC.new(model)
end
CollectionService:GetInstanceAddedSignal("Enemy"):Connect(function(model)
    npcs[model] = NPC.new(model)
end)
CollectionService:GetInstanceRemovedSignal("Enemy"):Connect(function(model)
    if npcs[model] then npcs[model]:Destroy(); npcs[model] = nil end
end)

RunService.Heartbeat:Connect(function(dt)
    for _, npc in npcs do npc:Tick(dt) end
end)
```

For pathfinding around obstacles (not just MoveTo), use `PathfindingService` — see [characters.md](characters.md).

## NPC appearance — copy a player avatar

```lua
-- Make the NPC look like a specific user
local Players = game:GetService("Players")
local desc = Players:GetHumanoidDescriptionFromUserId(123456789)
npc.Humanoid:ApplyDescription(desc)
```

Or use a custom rig built in Studio with the standard R15 / R6 rig from `Avatar Tab → Rig Builder`. See [avatar-customization.md](avatar-customization.md).

## NPC names — overhead labels

Configure the Humanoid's `DisplayName`, `NameDisplayDistance`, and `HealthDisplayType`:
```lua
npc.Humanoid.DisplayName = "Apothecary Mira"
npc.Humanoid.NameDisplayDistance = 50
npc.Humanoid.HealthDisplayType = Enum.HumanoidHealthDisplayType.AlwaysOff   -- or AlwaysOn for enemies
```

For richer labels (icons, color, "QUEST" tag), use a `BillboardGui` parented to the Head with a TextLabel inside. See [ui-system.md](ui-system.md).

## Designer-driven NPC instantiation

Tag NPCs and use Attributes for their tunables:
- Tag `"FriendlyNPC"` — designer drops a model with this tag, your script auto-wires the prompt
- Attributes `DialogId`, `ShopId`, `QuestId` — designer fills in which content to attach

```lua
for _, model in CollectionService:GetTagged("FriendlyNPC") do setupFriendly(model) end
CollectionService:GetInstanceAddedSignal("FriendlyNPC"):Connect(setupFriendly)

local function setupFriendly(model)
    local prompt = createPromptFor(model)
    prompt.Triggered:Connect(function(player)
        local dialogId = model:GetAttribute("DialogId")
        local shopId = model:GetAttribute("ShopId")
        local questId = model:GetAttribute("QuestId")
        if dialogId then runDialog(player, dialogId) end
        if shopId then openShop(player, shopId) end
        if questId then offerQuest(player, questId) end
    end)
end
```

Designers tag the NPC and fill attributes — no per-NPC scripts. See [tags.md](tags.md), [attributes.md](attributes.md).

## Pitfalls

- **Forgetting `:SetNetworkOwner(nil)`** on hostile NPCs → players manipulate their physics.
- **Anchored hostile NPC** → can't move (Humanoid.MoveTo silently does nothing on anchored HumanoidRootPart).
- **`Humanoid:MoveTo` with no `MoveToFinished` wait** → next call cancels the previous; loop with the wait or use Pathfinding waypoints.
- **`MoveTo` 8-second timeout** — if the NPC can't reach in 8s, the call resolves silently. For long paths, use Pathfinding with shorter waypoint legs.
- **NPC stuck on geometry** → use `PathfindingService:CreatePath` with `AgentRadius`/`AgentHeight` set to the NPC's actual size.
- **Touched-based attacks** → unreliable for fast NPCs; use raycasts or overlap queries on attack.
- **All NPCs in one hot loop without batching** → 100 NPCs each running pathfinding per Heartbeat will hammer the server. Tick AI at 5–10Hz, not 60Hz; pathfind only when target moves significantly.
- **Despawned NPC's connections leak** → connect via the NPC class's `:Destroy()` and disconnect there; or use a Trove (see [community-libraries.md](community-libraries.md)).
- **Spawned NPC clones share connections from the template** — if you connected events on the template, every clone fires them. Connect in `:Setup` after cloning, not on the template.
- **Friendly NPC ProximityPrompt firing for combat actions** — separate prompts for "Talk" (opens dialog) vs "Attack" (initiates combat); use Exclusivity to prevent overlap.

## See also
- [enemy-spawning.md](enemy-spawning.md) — spawner Instances, waves, budgets, despawn rules for hostile NPCs
- [enemy-difficulty.md](enemy-difficulty.md) — applying difficulty stat multipliers per spawn
- [characters.md](characters.md) — Humanoid, pathfinding, the rig anatomy NPCs share with player characters
- [interactions.md](interactions.md) — ProximityPrompt for dialog/vendor activation
- [oop-patterns.md](oop-patterns.md) — class pattern + state machine
- [tags.md](tags.md) — designer-driven NPC instantiation
- [attributes.md](attributes.md) — per-NPC tunables
- [shops.md](shops.md) — vendor NPCs open shops
- [quests.md](quests.md) — quest-giver NPCs offer quests
- [avatar-customization.md](avatar-customization.md) — NPC appearance via HumanoidDescription
- [security.md](security.md) — server network ownership for hostile NPCs
- [raycasting.md](raycasting.md) — line-of-sight, attack hit detection
