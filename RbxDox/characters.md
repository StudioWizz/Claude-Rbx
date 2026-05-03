# Characters & Humanoids

> Upstream: <https://create.roblox.com/docs/characters>
> Source: `content/en-us/characters/index.md`, `appearance.md`, `pathfinding.md`

## Purpose
A "character" is a `Model` with a `Humanoid` inside, plus body parts joined by `Motor6D`s. Roblox auto-spawns one for each player and respawns it when the Humanoid dies. NPCs use exactly the same structure — anything the player character does, an NPC can do.

## Anatomy

```
Character (Model)
├── Humanoid
├── HumanoidRootPart    (the invisible physics anchor)
├── Head, Torso/UpperTorso, LowerTorso, ...   (BaseParts)
├── Animator (under Humanoid)
├── Animate (LocalScript that plays movement animations)
├── Motor6Ds            (one per joint, parented under the part being moved)
└── BodyColors / Pants / Shirt / Accessory(s)  (appearance)
```

Two rig types: **R6** (6 body parts, simple) and **R15** (15 parts, with separate upper/lower limbs and per-finger expressivity). R15 is the modern default.

## Humanoid

The brain of the character. Key properties:

- `Health: number`, `MaxHealth: number`
- `WalkSpeed: number` (default 16)
- `JumpHeight: number` (default 7.2) or `JumpPower` (legacy)
- `HipHeight: number` — distance from feet to HumanoidRootPart center
- `AutoRotate: bool` — character faces movement direction
- `RigType: Enum.HumanoidRigType` — R6 or R15
- `MoveDirection: Vector3` (read-only, world-space)
- `FloorMaterial: Enum.Material` (read-only)
- `PlatformStand: bool` — disables locomotion (good for ragdoll)
- `Sit: bool`, `SeatPart`

Methods:
- `:TakeDamage(n)` — respects ForceField
- `:Move(direction, relativeToCamera?)` — manual control
- `:MoveTo(position, part?)` — pathless walk to a world point
- `:LoadAnimation(anim)` — returns an `AnimationTrack`
- `:GetAccessories()`
- `:UnequipTools()`, `:EquipTool(tool)`

Events:
- `Died`, `HealthChanged`, `Jumping`, `Climbing`, `Running`, `Seated`, `FreeFalling`, `StateChanged`, `Touched`

## Spawning and respawning

```lua
local Players = game:GetService("Players")

Players.PlayerAdded:Connect(function(player)
    player.CharacterAdded:Connect(function(character)
        local hum = character:WaitForChild("Humanoid")
        hum.Health = 100
        -- per-life setup
    end)
end)
```

`Players.CharacterAutoLoads = false` if you want to control spawning manually; then call `player:LoadCharacter()`.

## Appearance

Appearance is set by:
- `Pants`, `Shirt`, `ShirtGraphic` (legacy decals on Torso)
- `BodyColors` (per-part Color3)
- `Accessory` Instances (hats, faces, gear)
- `Clothing` (modern layered)
- `HumanoidDescription` (the modern bundle): a single Instance describing the avatar — body sizes, accessories, animations, clothing.

```lua
local desc = humanoid:GetAppliedDescription()
desc.Head = 13070796818  -- assetId
desc.Torso = 86500008
humanoid:ApplyDescription(desc)
```

`Players:GetHumanoidDescriptionFromUserId(userId)` fetches a player's avatar from the platform.

## Pathfinding for NPCs

```lua
local PathfindingService = game:GetService("PathfindingService")

local path = PathfindingService:CreatePath({
    AgentRadius = 2,
    AgentHeight = 5,
    AgentCanJump = true,
    AgentJumpHeight = 8,
    AgentMaxSlope = 45,
    Costs = {Water = 10, Lava = math.huge},
})

path:ComputeAsync(npc.HumanoidRootPart.Position, target.Position)

if path.Status == Enum.PathStatus.Success then
    for _, waypoint in path:GetWaypoints() do
        humanoid:MoveTo(waypoint.Position)
        if waypoint.Action == Enum.PathWaypointAction.Jump then
            humanoid.Jump = true
        end
        humanoid.MoveToFinished:Wait()
    end
end
```

`path.Blocked` fires if the route becomes obstructed mid-walk; recompute.

## Animations

```lua
local anim = Instance.new("Animation")
anim.AnimationId = "rbxassetid://1234567890"
local track = humanoid:LoadAnimation(anim)
-- (Newer: humanoid.Animator:LoadAnimation(anim) — preferred)
track:Play()
track.Looped = true
track:AdjustSpeed(1.5)
track:Stop()
```

Modern code should `LoadAnimation` on the `Animator` Instance under the Humanoid (Animator is required and exists on every Humanoid since 2018).

### Authoring an animation

Animations are not written in code — they're keyframed in Studio's **Animation Editor** (Avatar tab → Animation Editor). Workflow:

1. In Studio, select a rig (a Model with a Humanoid; use `Avatar → Rig Builder` to spawn a test rig if you don't have one).
2. Open the Animation Editor with the rig selected.
3. Create a new animation, set keyframes (move the rig's joints, scrub time, set a key).
4. Save → Export → Publish to Roblox. You get an asset ID.
5. In code: `anim.AnimationId = "rbxassetid://<published-id>"` and `LoadAnimation` as above.

You can only play an animation owned by you OR by the experience's owner (group or individual). Animations published under a different account won't load — re-author or re-publish under the correct account.

For non-character rigs (custom Models with Motor6Ds — vehicles, doors, animated props), put an `AnimationController` Instance in the model with an `Animator` child, and use the same `LoadAnimation` API on that Animator.

## Custom locomotion

The default Humanoid handles walking, jumping, falling, and basic swimming. For anything else — flying, sprinting, climbing, dashing, double-jump, jetpack — you override or augment.

### Sprint (the simplest case)
Just toggle `WalkSpeed` from a server-side handler that the client requests via RemoteEvent or ContextActionService:
```lua
-- ContextActionService on the client; mirror change to server via RemoteEvent if other systems care
CAS:BindAction("Sprint", function(_, state)
    local hum = char:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    hum.WalkSpeed = if state == Enum.UserInputState.Begin then 24 else 16
end, false, Enum.KeyCode.LeftShift, Enum.KeyCode.ButtonL3)
```
For server-authoritative speed (no client speed-hacking), keep speed server-side and apply via RemoteEvent.

### Flying — gravity-disabled style
The cleanest "no-clip fly" pattern: zero the character's gravity influence and drive position with `BodyVelocity` / `LinearVelocity` from the camera direction.

```lua
-- Server function (exposed via RemoteEvent or run server-side from a tool)
local function startFly(character: Model)
    local hum = character:FindFirstChildOfClass("Humanoid")
    local hrp = character:FindFirstChild("HumanoidRootPart") :: BasePart
    if not hum or not hrp then return end

    hum:ChangeState(Enum.HumanoidStateType.Physics)   -- bypasses default state machine
    hum.PlatformStand = true                          -- disables locomotion controllers

    local bv = Instance.new("BodyVelocity")           -- legacy Mover; LinearVelocity constraint also works
    bv.Name = "FlyVelocity"
    bv.MaxForce = Vector3.new(1, 1, 1) * 1e6
    bv.Velocity = Vector3.zero
    bv.Parent = hrp

    local bg = Instance.new("BodyGyro")               -- keeps the character upright / facing
    bg.Name = "FlyGyro"
    bg.MaxTorque = Vector3.new(1, 1, 1) * 1e6
    bg.D = 100; bg.P = 1e4
    bg.CFrame = hrp.CFrame
    bg.Parent = hrp
end

local function stopFly(character: Model)
    local hum = character:FindFirstChildOfClass("Humanoid")
    local hrp = character:FindFirstChild("HumanoidRootPart")
    if not hum or not hrp then return end

    hum.PlatformStand = false
    hum:ChangeState(Enum.HumanoidStateType.GettingUp)

    local bv = hrp:FindFirstChild("FlyVelocity"); if bv then bv:Destroy() end
    local bg = hrp:FindFirstChild("FlyGyro"); if bg then bg:Destroy() end
end
```

On the client (or server, via RemoteEvent), drive the velocity each frame from the camera direction + WASD input:
```lua
RunService.RenderStepped:Connect(function()
    local bv = hrp:FindFirstChild("FlyVelocity")
    local bg = hrp:FindFirstChild("FlyGyro")
    if not bv or not bg then return end

    local move = humanoid.MoveDirection                 -- WASD relative to camera
    bv.Velocity = move * FLY_SPEED + Vector3.new(0, verticalInput * FLY_SPEED, 0)
    bg.CFrame = workspace.CurrentCamera.CFrame          -- face camera direction
end)
```

The modern preferred approach uses `LinearVelocity` and `AlignOrientation` constraints instead of `BodyVelocity`/`BodyGyro` (which are deprecated but still work). See [physics-constraints.md](physics-constraints.md).

### Flying — jetpack with thrust (physics-based)
Player still walks normally; pressing the jetpack key adds an upward impulse via `VectorForce` while falling. Lets gravity behave normally; the jetpack just resists / accelerates upward.

```lua
local thrust = Instance.new("VectorForce")
thrust.Force = Vector3.new(0, 0, 0)
thrust.RelativeTo = Enum.ActuatorRelativeTo.World
thrust.Attachment0 = bodyAttachment
thrust.Parent = hrp

CAS:BindAction("Jetpack", function(_, state)
    if state == Enum.UserInputState.Begin then
        thrust.Force = Vector3.new(0, hrp.AssemblyMass * workspace.Gravity * 1.5, 0)   -- 1.5x gravity = ascend
    elseif state == Enum.UserInputState.End then
        thrust.Force = Vector3.zero
    end
end, false, Enum.KeyCode.Space)
```
Multiplying by `AssemblyMass * workspace.Gravity` gives a thrust that's force-correct regardless of player size.

### Climbing
Set `humanoid.AutoRotate = false` while climbing, then drive `hrp.CFrame` along the wall surface. Or detect a "climbable" surface via raycast in front of the character and switch to a custom state. Roblox has built-in climbing on truss parts and certain mesh edges — but custom climbing requires a state machine.

### Swim
Built-in for Terrain water. For custom water (lakes, oceans made of meshes), detect overlap with a "Water" tag and modify `humanoid.WalkSpeed` / `humanoid.JumpHeight`, or change `humanoid:ChangeState` to `Swimming`.

### Dash / dodge
One-shot impulse pattern:
```lua
CAS:BindAction("Dash", function(_, state)
    if state == Enum.UserInputState.Begin then
        local dir = humanoid.MoveDirection
        if dir.Magnitude < 0.1 then dir = hrp.CFrame.LookVector end
        hrp:ApplyImpulse(dir * hrp.AssemblyMass * 60)   -- mass-scaled
    end
end, false, Enum.KeyCode.Q)
```

### Double jump
Track jump count in a state variable, listen for `humanoid.StateChanged` to reset on landing:
```lua
local jumps = 0
local MAX_JUMPS = 2

humanoid.StateChanged:Connect(function(_, new)
    if new == Enum.HumanoidStateType.Landed then jumps = 0 end
end)

CAS:BindAction("DoubleJump", function(_, state)
    if state ~= Enum.UserInputState.Begin then return end
    if jumps < MAX_JUMPS then
        humanoid:ChangeState(Enum.HumanoidStateType.Jumping)
        jumps += 1
    end
end, false, Enum.KeyCode.Space)
```

### Locomotion state pitfalls
- **`humanoid:ChangeState` to `Physics`** bypasses the locomotion controller — useful for fly/dash, but you're responsible for re-enabling normal movement (`ChangeState(GettingUp)` works).
- **`PlatformStand = true`** disables Jump and movement entirely; it's the on/off switch for default locomotion.
- **Network ownership during fly/dash** — the client owns the character by default. If the player is using fly to clip out of bounds, validate server-side or set `SetNetworkOwner(nil)` on the HumanoidRootPart during the ability.
- **Mover Instances** (`BodyVelocity`, `BodyGyro`, `BodyForce`) are **deprecated** in favor of constraint-based equivalents (`LinearVelocity`, `AlignOrientation`, `VectorForce`). Both still work; new code should use constraints.

## Patterns

### React to death
```lua
humanoid.Died:Connect(function()
    -- drop loot, broadcast
end)
```

### Knockback
```lua
local hrp = character:WaitForChild("HumanoidRootPart")
hrp.AssemblyLinearVelocity = direction * 50
```

### Force-field on spawn
```lua
local ff = Instance.new("ForceField")
ff.Parent = character
ff.Visible = true
task.delay(5, function() ff:Destroy() end)
```

## Pitfalls

- **Touching `character` before children replicate.** Use `:WaitForChild` for `Humanoid`, `HumanoidRootPart`, etc.
- **Setting `WalkSpeed` on the client.** It sticks locally but the server canonical value rules. For game effects, set on server.
- **Loading animations on Humanoid (deprecated path).** Use `humanoid.Animator:LoadAnimation(anim)`.
- **Network ownership of NPCs.** Default goes to nearest player → they can manipulate NPC physics. `npc.HumanoidRootPart:SetNetworkOwner(nil)` for hostile NPCs.
- **`MoveTo` timing out at 8 seconds.** If the NPC can't reach in 8 s, the call resolves. Loop with shorter waypoints.
- **`humanoid:LoadAnimation` on a Humanoid that hasn't replicated yet** errors. Wait for the Animator.

## See also
- [avatar-customization.md](avatar-customization.md) — `HumanoidDescription`, accessories, clothing, emotes (deeper than this card's appearance section)
- [player-lifecycle.md](player-lifecycle.md) — PlayerAdded/CharacterAdded events, AFK, reconnect (the "when does this Humanoid exist?" picture)
- [npcs.md](npcs.md) — NPCs are characters too — the consolidated friendly + hostile composition
- [project-structure.md](project-structure.md) — Player vs Character
- [physics-constraints.md](physics-constraints.md) — Motor6D for joints; LinearVelocity/VectorForce for custom locomotion
- [tweens-animation.md](tweens-animation.md) — non-character animation
- [tools.md](tools.md) — Tools equipped to a Character
- [vehicles.md](vehicles.md) — Seat / VehicleSeat for sitting and driving
- [combat.md](combat.md) — Humanoid as the damage target; Health, status integration
- [progression.md](progression.md) — XP and level granted by combat tied to characters
- [spawn-mechanics.md](spawn-mechanics.md) — manual spawn flow, spawn protection, loadout-at-spawn
- [resource-bars.md](resource-bars.md) — health/stamina/mana pool patterns and HUD bars
- [player-badges.md](player-badges.md) — above-head BillboardGui badges parented to the character's Head
- [raycasting.md](raycasting.md) — line-of-sight, hit detection for NPCs
- [workspace-hierarchy.md](workspace-hierarchy.md) — Character is parented under Workspace as a Model
