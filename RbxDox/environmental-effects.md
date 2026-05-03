# Environmental Effect Blocks

> Source: distilled from obby / platformer / sandbox / RPG environmental-interaction patterns

## Purpose
A grab-bag of touch-triggered world objects that affect any character that contacts them: **kill blocks** (lava, void, instant death), **damage blocks** (slow damage over time), **heal blocks** (restore health), **buff/debuff blocks** (apply status effect), **currency pickups** (gain coins on touch), **speed pads** (temporary boost), **float/gravity zones** (zero-gravity / low-gravity), **invisibility blocks** (hide for N seconds), **teleport pads** (jump to another part), **checkpoints** (update respawn point). All share the same shape — a tagged BasePart with attributes that one designer-driven script handles. This card is the consolidated catalog. `world-mechanics.md` covers structural map mechanics (jump pads, conveyors, moving platforms); this card covers the touch-triggered state-changes.

## Common shape

Every effect block follows the same pattern:

```lua
-- Designer drops a Part in the world, sets attributes:
block:AddTag("EffectBlock")
block:SetAttribute("EffectType", "Damage")          -- which effect to apply
block:SetAttribute("EffectMagnitude", 5)             -- amount (varies by effect)
block:SetAttribute("EffectDuration", 0)              -- seconds (varies by effect)
block:SetAttribute("CooldownPerPlayer", 1)           -- seconds before same player can re-trigger
block:SetAttribute("OneShot", false)                 -- if true, destroys after first trigger

-- Optional appearance:
block.Material = Enum.Material.Neon
block.BrickColor = BrickColor.new("Bright red")
block.Transparency = 0.3
```

One server script handles every tagged effect block:

```lua
local CollectionService = game:GetService("CollectionService")
local Players = game:GetService("Players")

local EFFECT_HANDLERS: {[string]: (player: Player, block: BasePart) -> ()} = {}

local cooldowns: {[BasePart]: {[Player]: number}} = {}    -- per-block, per-player

local function setupEffectBlock(block: BasePart)
    cooldowns[block] = {}

    block.Touched:Connect(function(hit)
        local model = hit:FindFirstAncestorOfClass("Model")
        local hum = model and model:FindFirstChildOfClass("Humanoid")
        local player = hum and Players:GetPlayerFromCharacter(model)
        if not player or not hum or hum.Health <= 0 then return end

        -- Cooldown check
        local cooldownPeriod = block:GetAttribute("CooldownPerPlayer") or 1
        local lastTriggered = cooldowns[block][player] or 0
        if os.clock() - lastTriggered < cooldownPeriod then return end
        cooldowns[block][player] = os.clock()

        -- Dispatch by EffectType
        local effectType = block:GetAttribute("EffectType")
        local handler = EFFECT_HANDLERS[effectType]
        if handler then handler(player, block) end

        if block:GetAttribute("OneShot") then
            block:Destroy()
        end
    end)
end

for _, b in CollectionService:GetTagged("EffectBlock") do setupEffectBlock(b) end
CollectionService:GetInstanceAddedSignal("EffectBlock"):Connect(setupEffectBlock)
CollectionService:GetInstanceRemovedSignal("EffectBlock"):Connect(function(b) cooldowns[b] = nil end)
```

Then register handlers per effect type. The whole catalog below registers into this one dispatcher.

## The catalog

### Kill block (insta-death)
```lua
EFFECT_HANDLERS.Kill = function(player, block)
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    if hum then hum.Health = 0 end
end
```

Designer authoring: `EffectType = "Kill"`. Common materials: Lava (red Neon glow), water-with-deadly-flag, void (transparent below the world), spike pits.

For lava that **looks like lava**, set `Material = Enum.Material.CrackedLava`, `BrickColor = "Neon orange"`. Pair with [vfx.md](vfx.md) `Fire` Instance and [sound.md](sound.md) ambient hiss.

### Damage block (damage over time on contact)
```lua
EFFECT_HANDLERS.Damage = function(player, block)
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    local amount = block:GetAttribute("EffectMagnitude") or 5
    hum:TakeDamage(amount)
end
```

`CooldownPerPlayer = 1` means standing on a damage tile takes 5 dmg/s. Tune for the genre.

### Heal block
```lua
EFFECT_HANDLERS.Heal = function(player, block)
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    local amount = block:GetAttribute("EffectMagnitude") or 25
    hum.Health = math.min(hum.MaxHealth, hum.Health + amount)
end
```

For "full heal at base" pads: set magnitude to 9999 (clamped at MaxHealth) with cooldown 30s.

### Status effect block (buff / debuff)
```lua
EFFECT_HANDLERS.StatusEffect = function(player, block)
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    local effectId = block:GetAttribute("EffectMagnitude")    -- string here; could rename attribute
    local duration = block:GetAttribute("EffectDuration") or 5
    StatusEffects.Apply(hum, effectId, duration)
end
```

Designer sets `EffectMagnitude = "Burn"` with `EffectDuration = 8`. See [status-effects.md](status-effects.md) for the status system.

Variants: `EffectType = "Buff"` for known-good effects (Speed, Regen, Strength), `EffectType = "Debuff"` for known-bad (Poison, Slow, Stun). Same handler; just designer convention.

### Currency pickup (coin)
```lua
EFFECT_HANDLERS.Currency = function(player, block)
    local profile = PlayerData.Get(player); if not profile then return end
    local amount = block:GetAttribute("EffectMagnitude") or 1
    local currency = block:GetAttribute("Currency") or "Coins"
    profile[currency:lower()] = (profile[currency:lower()] or 0) + amount

    -- Visual feedback: floating "+1 coin" number (see progression.md floating XP pattern)
    showFloatingNumber(player, "+" .. amount .. " " .. currency, block.Position, Color3.fromRGB(255, 215, 0))

    -- Sound
    playPickupSound(player, block.Position)

    -- Always one-shot for pickups
    block:Destroy()
end
```

Designer authoring: `EffectType = "Currency"`, `EffectMagnitude = 5`, `Currency = "Gems"`, `OneShot = true`.

Couple with [world-mechanics.md](world-mechanics.md) Pickups pattern — coin pickups are basically the simplest pickup case.

### Speed pad (temporary boost)
```lua
EFFECT_HANDLERS.Speed = function(player, block)
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    local mult = block:GetAttribute("EffectMagnitude") or 2
    local duration = block:GetAttribute("EffectDuration") or 5

    StatusEffects.Apply(hum, "SpeedBoost", duration)
    -- Or directly modify WalkSpeed and reset after duration
end
```

For arrow-shaped boost pads (Sonic-style), pair with directional impulse:
```lua
local hrp = player.Character.HumanoidRootPart
hrp.AssemblyLinearVelocity += block.CFrame.LookVector * 50
```

### Float / low-gravity zone
```lua
EFFECT_HANDLERS.Float = function(player, block)
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    if not hum then return end
    local duration = block:GetAttribute("EffectDuration") or 10

    -- Apply BodyVelocity pointing up to counter gravity (or use modern LinearVelocity constraint)
    local hrp = player.Character:FindFirstChild("HumanoidRootPart") :: BasePart
    local existing = hrp:FindFirstChild("FloatVel")
    if existing then existing:Destroy() end

    local bv = Instance.new("BodyVelocity")
    bv.Name = "FloatVel"
    bv.MaxForce = Vector3.new(0, 1e6, 0)
    bv.Velocity = Vector3.new(0, 0, 0)        -- counter gravity, don't push up
    bv.Parent = hrp

    -- Or, lower the character's gravity: hum:ChangeState — see characters.md custom locomotion
    task.delay(duration, function()
        if bv and bv.Parent then bv:Destroy() end
    end)
end
```

For a sustained zone (you float as long as you're in it, fall when you leave), use Touched + TouchEnded to manage the BodyVelocity lifecycle.

### Invisibility block
```lua
EFFECT_HANDLERS.Invisibility = function(player, block)
    local char = player.Character; if not char then return end
    local duration = block:GetAttribute("EffectDuration") or 8

    -- Save original transparencies, set all to 1
    local saved = {}
    for _, p in char:GetDescendants() do
        if p:IsA("BasePart") then
            saved[p] = p.Transparency
            p.Transparency = 1
        end
    end

    task.delay(duration, function()
        for p, t in saved do
            if p and p.Parent then p.Transparency = t end
        end
    end)
end
```

For full invisibility (other players also can't see you), this is server-side and replicates. For self-only invisibility (camouflage stealth), do it client-side with custom replication.

Couple with disabling the player's Humanoid name display (`hum.NameDisplayDistance = 0`) for the duration.

### Teleport pad
```lua
EFFECT_HANDLERS.Teleport = function(player, block)
    local destPath = block:GetAttribute("DestinationPath")    -- e.g., "workspace.Pads.Destination"
    local dest = destPath and resolveByFullName(destPath)
    if not dest or not dest:IsA("BasePart") then return end

    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if hrp then hrp.CFrame = dest.CFrame + Vector3.new(0, 5, 0) end
end
```

Designer pairs two pads via `DestinationPath` attribute — touch one, teleport to the other.

For cross-place teleporting, see [teleport.md](teleport.md) `TeleportService` (different system).

### Checkpoint block
```lua
EFFECT_HANDLERS.Checkpoint = function(player, block)
    local stageNumber = block:GetAttribute("StageNumber") or 0
    local profile = PlayerData.Get(player); if not profile then return end

    if stageNumber > (profile.maxStageReached or 0) then
        profile.maxStageReached = stageNumber
        profile.currentStage = stageNumber
        notifyPlayer(player, "Checkpoint reached: Stage " .. stageNumber)
    end

    -- Set respawn location
    if block:IsA("SpawnLocation") then
        player.RespawnLocation = block
    end
end
```

For obby-style stage progression, see [areas-and-biomes.md](areas-and-biomes.md) discrete-stages section. The checkpoint block is the trigger; the stage-tracking is in the profile.

### Force-field / safe zone
```lua
EFFECT_HANDLERS.SafeZone = function(player, block)
    local char = player.Character; if not char then return end
    local existing = char:FindFirstChildOfClass("ForceField")
    if existing then return end       -- already in safe zone

    local ff = Instance.new("ForceField")
    ff.Parent = char
end
```

Pair with `TouchEnded` to remove on exit:
```lua
block.TouchEnded:Connect(function(hit)
    local model = hit:FindFirstAncestorOfClass("Model")
    local ff = model and model:FindFirstChildOfClass("ForceField")
    if ff then ff:Destroy() end
end)
```

## Combining effects

A "lava lake" might be both Damage and Slow:
- Either set the block's `EffectType = "LavaCombo"` and write a custom handler that applies both
- Or stack two overlapping invisible parts each with a single effect

For complex stacked effects (poison + on-fire + slowed), the **status-effects approach** is cleaner — one effect block applies a "BurningGround" status that itself contains multiple modifiers. See [status-effects.md](status-effects.md).

## Visual conventions players expect

- **Red glow** = damage / kill
- **Green glow** = heal
- **Yellow / gold particles** = currency
- **Blue arrow / pad shape** = speed boost
- **Purple particles + transparent** = invisibility / phase
- **Spinning ring** = teleport pad
- **Flag / checkered** = checkpoint
- **Hum / shimmer audio** = magical / status effect

Use [vfx.md](vfx.md) `ParticleEmitter` to add ambient particles. Use [sound.md](sound.md) for hum/sting on touch.

## Patterns

### Per-player one-shot (collected by you, still visible to others)
A coin pickup that disappears for the player who collected it but stays visible to others (so multiple players can each collect their own copy of all pickups). Don't `:Destroy()`; instead, track per-player collected:

```lua
local collected: {[Player]: {[BasePart]: true}} = setmetatable({}, {__mode = "k"})

EFFECT_HANDLERS.PerPlayerCoin = function(player, block)
    collected[player] = collected[player] or {}
    if collected[player][block] then return end
    collected[player][block] = true

    grantCoin(player, block:GetAttribute("EffectMagnitude") or 1)
    -- Hide the block for this player only (transparency client-side via per-player ScreenGui? Or accept that all players see it)
end
```

For per-player visibility, you'd render a clone in PlayerGui as a BillboardGui — fiddly. Accept the visual gloss and use a global one-shot for most games.

### Tag-driven setup at runtime (procgen)
Procedurally placed effect blocks in a procgen dungeon — same script picks them up via the `GetInstanceAddedSignal`. The L-system in [procedural-construction.md](procedural-construction.md) can spawn them; tagging happens at generation time.

### Effect block immunity by tag
"Bosses don't trigger checkpoints"; "Friendly NPCs don't trigger damage blocks." Add a tag check in the dispatcher:
```lua
if model:HasTag("EffectImmune") then return end
```

### Multi-step trap (combo block)
A "trap" that triggers a sequence: tilt the floor → drop spikes → release acid. Implement as a custom handler with `task.spawn` + `task.wait` for the sequence; use `OneShot = true` so a single trigger fires the whole show.

### "Hot floor" mechanic
Damage block tied to a TouchEnded sweep — players take damage as long as they stand on it, recover the moment they leave. Use a per-Heartbeat sweep over `workspace:GetPartBoundsInBox(block.CFrame, block.Size, op)` instead of relying on `Touched` (which fires sporadically).

### Conditional effect blocks
"Only triggers if you have the Stealth Cloak equipped." Add a check in the handler against the player's profile:
```lua
EFFECT_HANDLERS.SecretDoor = function(player, block)
    local profile = PlayerData.Get(player)
    if not profile or not Equipment.HasItemEquipped(profile, "StealthCloak") then return end
    -- Open the door only if cloak equipped
end
```

## Pitfalls

- **No cooldown.** `Touched` fires every physics step a contact exists; a damage block at 5 dmg per touch kills the player in one second of standing. Always cooldown.
- **Effect block triggered by non-character parts.** A loose Part rolling onto the trigger fires the handler. Filter to characters via `Players:GetPlayerFromCharacter`.
- **Effect handler errors crash the dispatcher.** One bad handler (typo in attribute name) errors → no further blocks fire. `pcall(handler, player, block)` to isolate.
- **Currency pickups granted client-side.** Trivial dupe exploit. Server is the only place that grants.
- **Float/Invisibility duration timer not cleaned up on death.** Player dies, respawns, but the float velocity object is still attached to the new character (or the timer fires on the new character). Hook to `humanoid.Died` to abort lingering effects.
- **Teleport pad to a destination that no longer exists.** `resolveByFullName` returns nil; player stays in place. Show an error or fall through gracefully.
- **Per-block cooldown table grown unbounded.** If players come and go but the table isn't cleaned, the `cooldowns[block]` map keeps every player who ever touched it. Periodically sweep, OR use a weak table (`__mode = "k"`) — but watch the Roblox-instance-not-weak gotcha (see [luau-basics.md](luau-basics.md)).
- **OneShot block destroyed before all triggers process.** A `:Destroy()` between Touched events for multiple players — only one player gets credit. Either `OneShot = false` for shared-credit pickups, OR use `Debris:AddItem(block, 0.1)` so it lingers briefly.
- **Touch detection on a CanCollide=false block.** Works; just confirm `CanTouch = true` (default true). Make the block `CanQuery = false` if you don't want raycasts hitting it.
- **Players bypassing kill blocks via lag.** A teleporting/lagging player can sometimes pass through Touched volumes. For critical kill zones (the void below the map), pair with a per-Heartbeat overlap query as a backup.
- **Effect blocks in shared world but per-player state.** A "checkpoint" should advance the player who touched it, not all players. The dispatcher above does this correctly; verify if you customize.
- **Stacking incompatible effects.** Player on a healing pad while burning — heal cancels burn, or burn outpaces heal? Test combinations; prefer explicit precedence rules.

## See also
- [world-mechanics.md](world-mechanics.md) — adjacent: jump pads, conveyors, moving platforms, trigger volumes (the structural-mechanic side)
- [collisions.md](collisions.md) — Touched, CanCollide / CanQuery / CanTouch flags
- [status-effects.md](status-effects.md) — buff/debuff blocks delegate to the status-effect system
- [characters.md](characters.md) — Humanoid health, custom locomotion (float zones use the same patterns)
- [resource-bars.md](resource-bars.md) — heal blocks restore the health pool
- [areas-and-biomes.md](areas-and-biomes.md) — per-area effect block rules; checkpoint integration with discrete stages
- [tags.md](tags.md), [attributes.md](attributes.md) — designer-driven block setup
- [vfx.md](vfx.md), [sound.md](sound.md) — visual + audio polish for effect blocks
- [progression.md](progression.md) — floating-number pattern for "+X coins" pickup feedback
- [security.md](security.md) — server-side validation; never trust client touch claims
