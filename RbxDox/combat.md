# Combat System

> Source: distilled from PvP and PvE combat patterns across fighting / RPG / FPS Roblox games

## Purpose
A combat system is more than "deal damage." It's the integration of **hit detection** (hitbox vs hitscan), **damage rules** (types, resistances, crits), **resource management** (HP, mana/stamina), **action timing** (combos, cooldowns, blocking, parries), and **feedback** (knockback, status effects, VFX, screen shake). `tools.md` covers the equippable layer; `security.md` covers the validation discipline; `raycasting.md` covers ray queries. This card pulls them together into a working combat surface.

## Hit detection — pick the right primitive

| Method | Use when | Trade-off |
|---|---|---|
| **Hitscan (raycast)** | Guns, lightning, lasers — instantaneous range attacks | Cheap, instant, no projectile travel |
| **Projectile** | Arrows, fireballs, grenades — visible travel | Looks better, dodgeable, more network state |
| **Hitbox** (Touched / overlap query) | Melee swings, AOE explosions | Server-validated easily; needs careful timing |
| **Region query** (`workspace:GetPartBoundsInBox`) | Ground-targeted AOE, cone attacks | Atomic check; no per-frame Touched listener |

For melee, prefer **overlap queries during the swing window** over `Touched`:
```lua
-- During an attack animation's "active window"
local function checkSwingHit(weaponPart: BasePart, attacker: Player)
    local op = OverlapParams.new()
    op.FilterType = Enum.RaycastFilterType.Exclude
    op.FilterDescendantsInstances = {attacker.Character}

    local hits = workspace:GetPartBoundsInBox(
        weaponPart.CFrame * CFrame.new(0, 0, -2),
        Vector3.new(2, 2, 4),       -- swing volume
        op
    )

    local hit = {}     -- dedupe per-character
    for _, p in hits do
        local model = p:FindFirstAncestorOfClass("Model")
        local hum = model and model:FindFirstChildOfClass("Humanoid")
        if hum and not hit[hum] then
            hit[hum] = true
            applyHit(attacker, hum)
        end
    end
end
```

For hitscan guns and abilities, see [raycasting.md](raycasting.md) — the canonical pattern.

For projectiles, simulate server-side and check overlap each tick (or use `FastCastRedux` from [community-libraries.md](community-libraries.md)).

## Damage application — central function

Always route damage through one function so resistances, crits, mitigation, and events fire consistently:

```lua
--!strict
export type DamageType = "Physical" | "Fire" | "Ice" | "Lightning" | "Magic" | "True"

export type DamageContext = {
    attacker: Player?,           -- nil for environmental damage
    victim: Humanoid,
    amount: number,
    damageType: DamageType,
    crit: boolean?,
    knockback: Vector3?,
    statusEffect: string?,       -- apply on hit, e.g., "Burn", "Stun"
}

local Combat = {}

function Combat.ApplyDamage(ctx: DamageContext)
    -- 1. Mitigation by resistances
    local victimProfile = getProfileFromHumanoid(ctx.victim)
    local resist = victimProfile and victimProfile.resistances[ctx.damageType] or 0
    local mitigated = ctx.amount * (1 - math.clamp(resist, 0, 0.9))

    -- 2. Crit
    if ctx.crit then mitigated *= 2 end

    -- 3. Block / parry check (see below)
    if isBlocking(ctx.victim) then
        mitigated *= 0.3
        drainBlock(ctx.victim, ctx.amount)
    end
    if isParrying(ctx.victim, ctx.attacker) then
        mitigated = 0
        applyStun(ctx.attacker, 1.0)    -- punish attacker
    end

    -- 4. Apply
    ctx.victim:TakeDamage(mitigated)

    -- 5. Knockback
    if ctx.knockback then
        local hrp = ctx.victim.RootPart
        if hrp then hrp.AssemblyLinearVelocity += ctx.knockback end
    end

    -- 6. Status effect
    if ctx.statusEffect then
        StatusEffects.Apply(ctx.victim, ctx.statusEffect, 5)
    end

    -- 7. Fire damage event for UI / quest progress / XP
    DamageDealt:Fire(ctx)
end

return Combat
```

Every damage source (sword, gun, environmental hazard, status DoT) constructs a `DamageContext` and calls `Combat.ApplyDamage`. Centralized = consistent.

## HP, stamina, and resource costs

`Humanoid.Health` is the engine's HP. Stamina/mana are your own attributes:

```lua
-- Per-character state, tracked server-side
type CombatState = {
    stamina: number, maxStamina: number,
    mana: number, maxMana: number,
    blocking: boolean, blockMeter: number,
    parryWindow: number?,    -- timestamp until which parry counts
    cooldowns: {[string]: number},   -- ability id → next-ready timestamp
}

local function tryUseAbility(player: Player, abilityId: string): boolean
    local state = getState(player); if not state then return false end
    local ability = ABILITIES[abilityId]; if not ability then return false end

    if state.cooldowns[abilityId] and os.clock() < state.cooldowns[abilityId] then return false end
    if state.stamina < ability.staminaCost then return false end
    if state.mana < ability.manaCost then return false end

    state.stamina -= ability.staminaCost
    state.mana -= ability.manaCost
    state.cooldowns[abilityId] = os.clock() + ability.cooldown

    ability.execute(player)
    return true
end
```

Regenerate via Heartbeat:
```lua
RunService.Heartbeat:Connect(function(dt)
    for player, state in playerStates do
        state.stamina = math.min(state.maxStamina, state.stamina + STAMINA_REGEN * dt)
        state.mana = math.min(state.maxMana, state.mana + MANA_REGEN * dt)
    end
end)
```

## Combos

A combo is a chain of attacks where each consecutive hit (within a window) deals more damage or unlocks the next move:

```lua
type ComboState = {
    chain: number,          -- current combo count
    lastHitAt: number,      -- os.clock()
}

local COMBO_WINDOW = 1.5   -- seconds between hits to maintain combo
local COMBO_DMG_MULT = {1, 1.1, 1.25, 1.5, 2.0}    -- per chain step

local function progressCombo(state: ComboState): number
    local now = os.clock()
    if now - state.lastHitAt > COMBO_WINDOW then state.chain = 0 end
    state.chain += 1
    state.lastHitAt = now
    return COMBO_DMG_MULT[math.min(state.chain, #COMBO_DMG_MULT)]
end
```

For **named combo strings** ("light, light, heavy"), check the input sequence:

```lua
local COMBOS = {
    {input = {"L", "L", "H"}, name = "Smash"},
    {input = {"L", "H", "L"}, name = "Sweep"},
}

local function checkCombo(inputs: {string}): string?
    for _, c in COMBOS do
        if matchesEnding(inputs, c.input) then return c.name end
    end
    return nil
end
```

## Blocking and parrying

Blocking reduces damage but drains a meter; parrying within a tight window reflects damage back:

```lua
local PARRY_WINDOW = 0.2   -- seconds after block-press where parry counts

blockRemote.OnServerEvent:Connect(function(player, action)
    local state = getState(player); if not state then return end
    if action == "start" then
        state.blocking = true
        state.parryWindow = os.clock() + PARRY_WINDOW
    elseif action == "stop" then
        state.blocking = false
        state.parryWindow = nil
    end
end)

local function isBlocking(humanoid: Humanoid): boolean
    local state = getStateFromHumanoid(humanoid)
    return state and state.blocking and state.blockMeter > 0
end

local function isParrying(humanoid: Humanoid, attacker: Player?): boolean
    local state = getStateFromHumanoid(humanoid)
    return state and state.parryWindow and os.clock() < state.parryWindow
end
```

The block meter recovers when not blocking; depletes on hits taken while blocking. When it's empty, the player is "guard broken" — animation cancels, briefly stunned.

## Knockback

A linear velocity impulse:
```lua
ctx.knockback = (victim.RootPart.Position - attacker.Character.HumanoidRootPart.Position).Unit * 30
```

Direction = away-from-attacker; magnitude = "knockback strength" tuned per ability. For an "uppercut" knockup, swap horizontal for vertical: `Vector3.new(0, 50, 0)`.

For physics-correct knockback (heavy enemies move less), scale by inverse mass:
```lua
local massFactor = 5 / victim.RootPart.AssemblyMass
ctx.knockback *= massFactor
```

## Damage types and resistances

Damage types let you build elemental rock-paper-scissors:

```lua
profile.resistances = {Fire = 0.5, Ice = -0.25}    -- 50% resist fire, 25% weak to ice
```

`-0.25` is *vulnerability* (takes 125%). Cap at the call site (`math.clamp(resist, -0.5, 0.9)`) so no resistance trivializes a damage type entirely.

`"True"` damage type bypasses all resistances — useful for unblockable mechanics (boss execute, environmental fall damage).

## Crits

Roll on the attacker's crit chance:
```lua
local function rollCrit(attacker: Player): boolean
    local profile = PlayerData.Get(attacker)
    return math.random() < (profile and profile.critChance or 0.05)
end
```

## Network ownership during combat

For PvP fairness, **don't let attackers own physics of their projectiles or hitboxes**:
```lua
projectile:SetNetworkOwner(nil)   -- server simulates
```

For player characters in PvP, the engine's default (player owns own character) is normally fine — combat *outcomes* (damage application) are server-validated even if movement is client-owned. See [security.md](security.md).

## Pitfalls

- **Trusting client hit reports.** "I hit them with my sword" → ignore. The server runs the overlap/raycast; the client sends only intent ("I swung at time T").
- **Touched-based melee.** Touched is unreliable for fast swings — uses physics step granularity, can miss between steps, doesn't fire on anchored-anchored contact. Use overlap queries during the active window.
- **No per-swing dedupe.** A single swing's overlap query can hit the same Humanoid twice if its hitbox overlaps the swing volume in two parts. Track `hit[hum] = true` per swing.
- **Damage applied client-side.** Players get free godmode by simply not running your "deal damage to me" path. Server is the only authority.
- **Cooldown stored client-side.** Same exploit. Server stores cooldowns; client UI shows a *display* of the cooldown which the server pushes.
- **Friendly-fire.** Forget to skip teammates and you get griefing. Filter by Team or by attacker-victim ownership.
- **Self-damage from your own AOE.** Filter the attacker's own character out of the overlap query.
- **Knockback that pushes the player into geometry.** Cap velocity magnitude; ideally raycast from victim along the knockback direction and clip distance to first wall hit.
- **Combos persisting across deaths.** Reset combo state on `humanoid.Died` and on respawn.
- **Crit roll on the client.** Roll on server. The client's UI reflecting "CRIT!" is a server-pushed flag.
- **Stamina/mana negative.** Floor at 0 on every change.
- **Damage event leaks information.** Firing "you took 23 damage" to other clients reveals defender stats — for competitive games, only fire to involved parties.
- **Kill credit ambiguity.** Multiple players hit one enemy — who gets the kill XP/loot? Define one rule (last hit, most damage, etc.) and stick to it.

## See also
- [tools.md](tools.md) — equippable weapons that drive Combat.ApplyDamage
- [raycasting.md](raycasting.md) — hitscan and overlap queries
- [security.md](security.md) — server-authoritative validation; never trust client damage claims
- [status-effects.md](status-effects.md) — buffs/debuffs/DoT that combat hits apply
- [progression.md](progression.md) — XP grants on kill
- [characters.md](characters.md) — Humanoid, Health, custom locomotion (dash mid-combat)
- [resource-bars.md](resource-bars.md) — unified HP/stamina/mana/shield pool model and bar UI; the home of regen rules and pool-interaction patterns
- [enemy-difficulty.md](enemy-difficulty.md) — `Damage` and `XpMult` attributes that this card's `ApplyDamage` reads for enemy attacks and player kill rewards
- [vfx.md](vfx.md), [sound.md](sound.md) — hit feedback (particles, blood, screen shake, audio)
- [guns-fps.md](guns-fps.md) — gun-specific combat (ammo, recoil, ADS) built on this card's foundation
- [community-libraries.md](community-libraries.md) — `FastCastRedux` for projectile motion
- [npcs.md](npcs.md) — hostile NPCs use Combat.ApplyDamage too
- [spawn-mechanics.md](spawn-mechanics.md) — death triggers the death-cam → respawn flow
