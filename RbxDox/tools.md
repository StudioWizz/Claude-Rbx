# Tools (equippable items)

> Upstream: <https://create.roblox.com/docs/reference/engine/classes/Tool>
> Source: distilled from the Tool API reference and idiomatic Roblox tool patterns

## Purpose
A `Tool` is the standard equippable item in Roblox — sword, gun, wand, fishing rod, food, key. The engine handles the equip/unequip lifecycle, parents the tool under the player's `Character` while equipped, and exposes events for swing/use. Tools are the dominant gameplay primitive in most Roblox games; getting their structure right unlocks half the genre.

## Anatomy

```
Tool                            -- the item the player equips
├── Handle (BasePart)           -- REQUIRED if Tool.RequiresHandle = true; the visible part
├── (other BaseParts welded to Handle for visual detail)
├── Script  (server logic)      -- damage, cooldowns, server-authoritative effects
├── LocalScript (client logic)  -- mouse/aim, predicted FX, sounds (or Script with RunContext)
├── Animation (asset)           -- swing/idle animations to play on Equipped
└── Sound (under Handle)        -- 3D sound that plays from the tool
```

The `Handle` part must be **named exactly "Handle"** when `RequiresHandle = true` (default). Welded children of Handle move with it.

## Where Tools live

| Container | Behavior |
|---|---|
| `StarterPack` | Copied into every player's `Backpack` on spawn — every player gets one |
| `StarterCharacterScripts` (rare for tools) | Auto-equip-on-spawn pattern; not standard |
| `ServerStorage` (templates) | Server clones from here when granting a tool at runtime |
| `ReplicatedStorage` (templates) | OK if both sides need to read structure; clients see the source |
| `Player.Backpack` (runtime) | Tool is in inventory, equippable but not currently equipped |
| `Player.Character` (runtime) | Tool is currently equipped — engine moves it here automatically |

To grant a tool at runtime: `tool:Clone(); clone.Parent = player.Backpack`. To force-equip immediately: also call `humanoid:EquipTool(clone)`.

## Key API

```lua
-- Properties
tool.Handle                 -- the BasePart named "Handle"
tool.RequiresHandle         -- bool, default true
tool.CanBeDropped           -- bool, default true (can the player Backspace to drop?)
tool.ManualActivationOnly   -- bool, default false (suppress auto-activation on click)
tool.Enabled                -- bool, default true (set false to disable Activated)
tool.GripPos, GripUp, GripRight, GripForward  -- positioning in the player's hand
tool.ToolTip                -- text shown when hovering in inventory
tool.TextureId              -- icon image in the inventory UI

-- Methods
tool:GetHandle()            -- returns Handle (errors if missing)

-- Events
tool.Equipped:Connect(function(mouse: Mouse?) end)    -- mouse only on client
tool.Unequipped:Connect(function() end)
tool.Activated:Connect(function() end)                 -- left-click / tap / gamepad button
tool.Deactivated:Connect(function() end)               -- release
```

The `Mouse` argument to `Equipped` is **only passed to LocalScripts** (it's a client concept). Server-side `Tool.Equipped` runs without it.

## Patterns

### Damage on swing (server-authoritative)
```lua
-- ServerStorage/Tools/Sword/Script (Script inside the Tool)
local tool = script.Parent
local handle = tool:WaitForChild("Handle")

local DAMAGE = 25
local COOLDOWN = 0.5
local lastSwing = 0

local function onActivated()
    local now = os.clock()
    if now - lastSwing < COOLDOWN then return end
    lastSwing = now

    local hits = {}   -- per-swing dedup
    local conn
    conn = handle.Touched:Connect(function(hit)
        local model = hit:FindFirstAncestorOfClass("Model")
        if not model or model == tool.Parent then return end   -- skip wielder
        if hits[model] then return end
        hits[model] = true

        local hum = model:FindFirstChildOfClass("Humanoid")
        if hum then hum:TakeDamage(DAMAGE) end
    end)

    task.wait(0.4)
    conn:Disconnect()
end

tool.Activated:Connect(onActivated)
```

Damage logic lives in a server `Script` *inside the Tool*. When the tool is equipped, the Script runs as a child of the Character. When the tool is back in Backpack, the Script is still alive but the player can't trigger Activated.

### Equip animations
```lua
-- LocalScript inside the Tool
local tool = script.Parent
local anim = tool:WaitForChild("SwingAnim") :: Animation

local equippedTrack: AnimationTrack? = nil

tool.Equipped:Connect(function()
    local char = tool.Parent
    local hum = char:FindFirstChildOfClass("Humanoid") :: Humanoid
    equippedTrack = hum.Animator:LoadAnimation(anim)
    equippedTrack.Looped = false
end)

tool.Activated:Connect(function()
    if equippedTrack then equippedTrack:Play() end
end)

tool.Unequipped:Connect(function()
    if equippedTrack then equippedTrack:Stop(); equippedTrack:Destroy(); equippedTrack = nil end
end)
```
See [characters.md](characters.md) and [tweens-animation.md](tweens-animation.md) for the Animator API.

### Grip (where the hand holds the Handle)
The `Tool.Grip` is a CFrame composed from `GripPos` + `GripUp/Right/Forward`. Easiest to set with the **Tool Grip Editor** plugin (see [community-libraries.md](community-libraries.md)). For code:
```lua
tool.Grip = CFrame.new(0, 0, -1) * CFrame.Angles(math.rad(-90), 0, 0)
```

### Granting a tool from the server
```lua
local Players = game:GetService("Players")
local ServerStorage = game:GetService("ServerStorage")

local function giveSword(player: Player)
    local sword = ServerStorage.Tools.Sword:Clone()
    sword.Parent = player.Backpack
end

Players.PlayerAdded:Connect(function(p)
    p.CharacterAdded:Connect(function() giveSword(p) end)
end)
```

### Server-side click validation (gun-like tools)
```lua
-- LocalScript: tells server where the player aimed
tool.Activated:Connect(function()
    local mouse = Players.LocalPlayer:GetMouse()
    fireRemote:FireServer(mouse.Hit.Position)
end)

-- Script (server): re-raycast and apply damage
fireRemote.OnServerEvent:Connect(function(player, claimedTarget)
    -- Validate: is the player holding this tool? Is target plausibly in line of sight?
    if tool.Parent ~= player.Character then return end
    -- Re-raycast on server, never trust the client's claim of what was hit
    -- ...
end)
```
See [security.md](security.md) for the validation discipline.

## Pitfalls

- **`Handle` not named exactly `Handle`** with `RequiresHandle = true` — the Tool fails to equip silently. Either rename the part or set `RequiresHandle = false`.
- **`Activated` only fires on the side a Script is running.** A `Script` in the Tool gets it server-side; a `LocalScript` gets it client-side. If both are present, both fire — design accordingly.
- **`Equipped` Mouse arg is client-only.** Server-side handlers should not expect it.
- **Tools you grant via `:Clone()` need to go to `Backpack`**, not Character. Equipping is the engine's job.
- **`Tool.Equipped` can fire before the Character is fully replicated** on the client. Use `WaitForChild` for any character descendant you reach.
- **Don't put gameplay logic in the Tool template that you'll need to update.** Many copies exist in players' Backpacks — template changes won't propagate. Put logic in a ModuleScript in `ReplicatedStorage` and have the Tool's Script `require` it.
- **Touched-based hit detection is unreliable** for fast-moving weapons (the part may pass through the target between physics steps). Use `Raycast` (or `Blockcast` for swept volume) on the server during the active swing window. See [raycasting.md](raycasting.md).
- **Forgetting `tool.Enabled = false` during cooldown** leaves Activated firing — your debounce works but the player sees the swing animation play with no effect. Disable the tool for the cooldown duration.
- **Tool dropped (Backspace) by a player** triggers `Unequipped` and reparents the Tool back to Workspace as a pickup. If you don't want this, set `CanBeDropped = false`.
- **Multiple tools in Backpack with the same `Name`** all show in the hotbar but the engine indexes them by the inventory order — confusing. Keep names unique.

## See also
- [project-structure.md](project-structure.md) — StarterPack vs Backpack vs Character
- [characters.md](characters.md) — Humanoid, EquipTool, Animator
- [security.md](security.md) — server-authoritative damage, never trust client's hit claim
- [events-signals.md](events-signals.md) — RemoteEvent for client → server actions
- [raycasting.md](raycasting.md) — proper hit detection for ranged/fast weapons
- [combat.md](combat.md) — central damage application, combos, blocking, knockback
- [guns-fps.md](guns-fps.md) — gun-specific patterns (ammo, reload, recoil, ADS) built on tools
- [sound.md](sound.md) — 3D sounds parented under Handle
- [interactions.md](interactions.md) — alternative for non-equippable interactions (doors, vendors)
- [equipment.md](equipment.md) — Weapon-slot integration with stat aggregation
