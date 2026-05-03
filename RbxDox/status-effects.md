# Status Effects (Buffs & Debuffs)

> Source: distilled from RPG/fighting/survival status-effect patterns

## Purpose
A **status effect** is a time-limited modification applied to a Humanoid: poison ticking damage, stun preventing actions, slow reducing speed, regen restoring HP, "on fire" doing both DoT and visual smoke. The pattern is consistent: a registry of effect definitions, per-character active-effect state, periodic ticking, stack/refresh rules, and a visual indicator. This card covers the canonical assembly. Combat hits apply effects (`combat.md`), and the multiplier-stack discipline from `progression.md` extends naturally to stat modifiers.

## Effect data

```lua
--!strict
export type EffectKind = "Buff" | "Debuff"
export type StackRule = "Refresh" | "Stack" | "Independent"

export type EffectDef = {
    id: string,
    name: string,
    kind: EffectKind,
    stackRule: StackRule,         -- how multiple applications combine
    maxStacks: number?,           -- for Stack rule
    duration: number,             -- seconds
    tickInterval: number?,        -- for DoT/HoT; nil = no tick
    iconId: string?,
    description: string,

    -- Hooks
    onApply: ((target: Humanoid, stacks: number) -> ())?,
    onTick: ((target: Humanoid, stacks: number) -> ())?,
    onRemove: ((target: Humanoid) -> ())?,

    -- Stat modifiers (additive or multiplicative)
    walkSpeedMult: number?,       -- 0.5 = halve walkspeed
    damageDealtMult: number?,
    damageTakenMult: number?,
}
```

| Stack rule | Behavior |
|---|---|
| **Refresh** | Re-applying resets the duration; only one instance exists |
| **Stack** | Re-applying adds a stack (capped at `maxStacks`); duration may refresh |
| **Independent** | Each application is its own instance with its own duration |

## Effect registry

```lua
local Effects = {}

Effects.Burn = {
    id = "Burn", name = "Burning", kind = "Debuff",
    stackRule = "Refresh", duration = 5, tickInterval = 1,
    iconId = "rbxassetid://...",
    description = "Take 5 damage per second.",
    onTick = function(target, _) target:TakeDamage(5) end,
}

Effects.Stun = {
    id = "Stun", name = "Stunned", kind = "Debuff",
    stackRule = "Refresh", duration = 1.5,
    description = "Cannot move or act.",
    walkSpeedMult = 0,
    onApply = function(target, _) target.PlatformStand = true end,
    onRemove = function(target) target.PlatformStand = false end,
}

Effects.Speed = {
    id = "Speed", name = "Hasted", kind = "Buff",
    stackRule = "Stack", maxStacks = 5, duration = 10,
    description = "+10% speed per stack.",
    walkSpeedMult = 1.1,    -- per stack; aggregator multiplies stacks times
}

Effects.Regen = {
    id = "Regen", name = "Regenerating", kind = "Buff",
    stackRule = "Refresh", duration = 8, tickInterval = 1,
    onTick = function(target, _) target.Health = math.min(target.MaxHealth, target.Health + 5) end,
}

return Effects
```

## Active state

```lua
type ActiveEffect = {
    effectId: string,
    expiresAt: number,            -- os.clock()
    stacks: number,
    instanceId: string?,          -- for Independent rule
    nextTickAt: number?,
}

-- Per-Humanoid (use a weak table; see luau-basics.md weak tables)
local active: {[Humanoid]: {ActiveEffect}} = {}
```

## Apply

```lua
function StatusEffects.Apply(target: Humanoid, effectId: string, durationOverride: number?)
    local def = Effects[effectId]; if not def then return end

    active[target] = active[target] or {}
    local list = active[target]

    local duration = durationOverride or def.duration
    local now = os.clock()

    if def.stackRule == "Refresh" then
        -- Find existing; refresh duration
        for _, e in list do
            if e.effectId == effectId then
                e.expiresAt = now + duration
                return
            end
        end
        -- Otherwise add new
        local e: ActiveEffect = {
            effectId = effectId, expiresAt = now + duration, stacks = 1,
            nextTickAt = def.tickInterval and (now + def.tickInterval) or nil,
        }
        table.insert(list, e)
        if def.onApply then def.onApply(target, 1) end

    elseif def.stackRule == "Stack" then
        for _, e in list do
            if e.effectId == effectId then
                e.stacks = math.min(e.stacks + 1, def.maxStacks or math.huge)
                e.expiresAt = now + duration       -- refresh on stack-add
                if def.onApply then def.onApply(target, e.stacks) end
                return
            end
        end
        local e: ActiveEffect = {effectId = effectId, expiresAt = now + duration, stacks = 1,
                                  nextTickAt = def.tickInterval and (now + def.tickInterval) or nil}
        table.insert(list, e)
        if def.onApply then def.onApply(target, 1) end

    else    -- Independent
        local e: ActiveEffect = {effectId = effectId, expiresAt = now + duration, stacks = 1,
                                  instanceId = HttpService:GenerateGUID(false):sub(1, 8),
                                  nextTickAt = def.tickInterval and (now + def.tickInterval) or nil}
        table.insert(list, e)
        if def.onApply then def.onApply(target, 1) end
    end

    recomputeStatModifiers(target)
    pushVisualUpdate(target)
end
```

## Remove

```lua
function StatusEffects.Remove(target: Humanoid, effectId: string)
    local list = active[target]; if not list then return end
    for i = #list, 1, -1 do
        if list[i].effectId == effectId then
            local e = list[i]
            local def = Effects[e.effectId]
            if def and def.onRemove then def.onRemove(target) end
            table.remove(list, i)
        end
    end
    recomputeStatModifiers(target)
    pushVisualUpdate(target)
end

function StatusEffects.Clear(target: Humanoid)
    local list = active[target]; if not list then return end
    for _, e in list do
        local def = Effects[e.effectId]
        if def and def.onRemove then def.onRemove(target) end
    end
    active[target] = nil
    recomputeStatModifiers(target)
    pushVisualUpdate(target)
end
```

## Periodic tick + expiry sweep

One Heartbeat-driven sweep handles every active effect across every character:

```lua
RunService.Heartbeat:Connect(function()
    local now = os.clock()
    for hum, list in active do
        if not hum.Parent then active[hum] = nil; continue end

        for i = #list, 1, -1 do
            local e = list[i]
            local def = Effects[e.effectId]
            if not def then table.remove(list, i); continue end

            -- Tick
            if e.nextTickAt and now >= e.nextTickAt then
                if def.onTick then def.onTick(hum, e.stacks) end
                e.nextTickAt = now + (def.tickInterval or 1)
            end

            -- Expire
            if now >= e.expiresAt then
                if def.onRemove then def.onRemove(hum) end
                table.remove(list, i)
                recomputeStatModifiers(hum)
            end
        end

        if #list == 0 then active[hum] = nil end
    end
end)
```

## Stat modifier aggregation

When multiple effects modify the same stat (WalkSpeed, damage), aggregate **multiplicatively** (consistent with the progression.md multiplier discipline):

```lua
local function recomputeStatModifiers(hum: Humanoid)
    local list = active[hum] or {}
    local walkSpeedMult = 1
    local dmgDealtMult = 1
    local dmgTakenMult = 1

    for _, e in list do
        local def = Effects[e.effectId]; if not def then continue end
        if def.walkSpeedMult then
            walkSpeedMult *= def.walkSpeedMult ^ e.stacks       -- N stacks of 0.9 = 0.9^N
        end
        if def.damageDealtMult then dmgDealtMult *= def.damageDealtMult ^ e.stacks end
        if def.damageTakenMult then dmgTakenMult *= def.damageTakenMult ^ e.stacks end
    end

    hum.WalkSpeed = (hum:GetAttribute("BaseWalkSpeed") or 16) * walkSpeedMult
    hum:SetAttribute("DamageDealtMult", dmgDealtMult)
    hum:SetAttribute("DamageTakenMult", dmgTakenMult)
end
```

`combat.md` `Combat.ApplyDamage` reads `DamageDealtMult` from the attacker and `DamageTakenMult` from the victim during the calculation.

## Visual indicators

Push the active list to the player's client; render small icons above their character or in a HUD bar:

```lua
local function pushVisualUpdate(hum: Humanoid)
    local player = Players:GetPlayerFromCharacter(hum.Parent)
    if not player then return end
    StatusEffectsRemote:FireClient(player, active[hum] or {})
end
```

For visual flair (burning fire particle, frozen blue tint), pair `onApply` and `onRemove` with [vfx.md](vfx.md):
```lua
Effects.Burn.onApply = function(target, _)
    local fire = Instance.new("Fire")
    fire.Size = 5
    fire.Parent = target.RootPart
end
Effects.Burn.onRemove = function(target)
    local fire = target.RootPart:FindFirstChildOfClass("Fire")
    if fire then fire:Destroy() end
end
```

## Patterns

### Crowd control immunity
After being stunned, immune to stun for N seconds (prevents stun-locking):
```lua
Effects.StunImmunity = {
    id = "StunImmunity", duration = 3, ...
}

Effects.Stun.onApply = function(target)
    target.PlatformStand = true
    -- After stun ends, apply immunity
end
Effects.Stun.onRemove = function(target)
    target.PlatformStand = false
    StatusEffects.Apply(target, "StunImmunity")
end

-- In Apply, check for immunity:
if effectId == "Stun" and hasEffect(target, "StunImmunity") then return end
```

### Cleanse / dispel
A "cleanse" spell removes all debuffs:
```lua
function cleanse(target: Humanoid)
    local list = active[target] or {}
    for i = #list, 1, -1 do
        local def = Effects[list[i].effectId]
        if def and def.kind == "Debuff" then
            StatusEffects.Remove(target, list[i].effectId)
        end
    end
end
```

### Effect stacking with diminishing returns
Each stack is less effective than the previous:
```lua
walkSpeedMult = 1 - (1 - 0.9) / e.stacks   -- diminishing
```
Override the simple `^stacks` aggregation per-effect for designer-tuned curves.

## Pitfalls

- **Effects on a Humanoid that's destroyed.** The active table holds a dangling reference. Sweep on Heartbeat (`if not hum.Parent then active[hum] = nil`).
- **Tick fires faster than expected.** Setting `nextTickAt = now + tickInterval` after the tick is correct; setting it on every Heartbeat resets it. Update only when the tick fires.
- **Stun set via WalkSpeed = 0** doesn't prevent jumping or attacks — only locomotion. Use `humanoid.PlatformStand = true` for full lockdown, or also set `humanoid.JumpHeight = 0` for partial.
- **Stat modifier permanently sticks.** If you forget to `recomputeStatModifiers` on Remove, the WalkSpeed never recovers. Always recompute in Apply, Remove, and Clear.
- **Adding modifiers additively when they should be multiplicative.** Two 50% slows shouldn't = 100% (full stop) — they should compound to 25%. Multiply (0.5 × 0.5).
- **Saving status effects to DataStore.** Effects are ephemeral combat state; they shouldn't persist across sessions. Clear on PlayerRemoving / CharacterAdded.
- **Effect set in `:Apply` reads stale stacks.** When `Refresh` rule re-applies, `e.stacks` is still 1, but `Stack` rule increments it before calling `onApply`. Check the rule.
- **Recompute hits every Heartbeat instead of only on change.** Recompute when applying, removing, or stack-changing — not every tick. Use a "dirty" flag if needed.
- **Visual particles attached to a destroyed RootPart.** `:Destroy()` the visual on Remove, not on parent destruction.
- **Effect that buffs damage applied to attacker but read on victim.** Track which way each modifier flows. `damageDealtMult` reads attacker; `damageTakenMult` reads victim. Don't mix.
- **Server-side state, client-side UI lying.** UI shows "Burning 3s" but server shows "Burning 5s". Push updates from server on every change so they stay in sync.

## See also
- [combat.md](combat.md) — applies status effects on hit; reads damage modifiers from active effects
- [characters.md](characters.md) — Humanoid is the target; PlatformStand/WalkSpeed are common modifiers
- [progression.md](progression.md) — multiplier-stack discipline (effects extend the same idea)
- [vfx.md](vfx.md), [sound.md](sound.md) — visual/audio indicators on apply/remove
- [ui-system.md](ui-system.md) — status icon HUD
- [tweens-animation.md](tweens-animation.md) — duration bar animations
- [tags.md](tags.md) — could tag immune characters (e.g., "BossNoStun") for easy gating
- [luau-basics.md](luau-basics.md) — weak tables for the active map
