# Guns & FPS Mechanics

> Source: distilled from FPS / shooter / military-genre Roblox patterns

## Purpose
Guns share the `Tool` foundation (`tools.md`) and the combat foundation (`combat.md`), but FPS-genre weapons have specific patterns that don't apply to melee or magic: **ammo** (per-magazine + reserve), **reload** (animation, time, cancellation), **recoil** (vertical kick, horizontal sway, pattern), **spread** (accuracy degradation while firing), **ADS** (aim-down-sights — zoom and accuracy modifier), **kill feed** (game-wide notification of kills), and choice of **hitscan vs projectile**. This card covers each piece.

## Gun definition

```lua
--!strict
export type FireMode = "Semi" | "Auto" | "Burst"

export type GunDef = {
    id: string,
    name: string,
    damage: number,
    headshotMultiplier: number,
    fireMode: FireMode,
    fireRate: number,                 -- rounds per second
    burstCount: number?,              -- for Burst mode
    range: number,                    -- studs
    magazineSize: number,
    reserveAmmo: number,
    reloadTimeSec: number,

    -- Recoil per shot (degrees)
    recoilVertical: number,           -- camera kicks up
    recoilHorizontal: number,         -- random L/R sway

    -- Spread (degrees)
    baseSpread: number,
    spreadPerShot: number,            -- accumulates while firing
    maxSpread: number,
    spreadRecoverPerSec: number,

    -- ADS
    adsZoomFov: number,               -- camera FOV when scoped (default 70)
    adsSpeedMult: number,             -- walk speed reduction while ADS (e.g., 0.6)
    adsAccuracyBonus: number,         -- spread reduction multiplier (e.g., 0.5)

    -- Projectile vs hitscan
    projectile: boolean,
    projectileSpeed: number?,         -- studs/sec; only if projectile = true
}

local Guns = {}

Guns.AssaultRifle = {
    id = "AssaultRifle", name = "Assault Rifle",
    damage = 30, headshotMultiplier = 1.5,
    fireMode = "Auto", fireRate = 10,
    range = 250, magazineSize = 30, reserveAmmo = 120, reloadTimeSec = 2.5,
    recoilVertical = 0.8, recoilHorizontal = 0.4,
    baseSpread = 0.3, spreadPerShot = 0.15, maxSpread = 3.5, spreadRecoverPerSec = 4,
    adsZoomFov = 50, adsSpeedMult = 0.7, adsAccuracyBonus = 0.5,
    projectile = false,
}

return Guns
```

## Per-gun runtime state

```lua
type GunState = {
    inMagazine: number,
    reserveAmmo: number,
    currentSpread: number,
    isReloading: boolean,
    lastFireAt: number,
    isAdsActive: boolean,
}
```

State lives on the server (authoritative) with a mirror on the owning client (predicted, for responsive UI/feel).

## Firing

```lua
local function tryFire(player: Player, gunState: GunState, def: GunDef, aimDir: Vector3)
    local now = os.clock()
    local minInterval = 1 / def.fireRate
    if now - gunState.lastFireAt < minInterval then return end
    if gunState.isReloading then return end
    if gunState.inMagazine <= 0 then
        playClickSound(player)
        return
    end

    gunState.lastFireAt = now
    gunState.inMagazine -= 1

    -- Add spread to aim direction
    local spread = gunState.currentSpread * (gunState.isAdsActive and def.adsAccuracyBonus or 1)
    aimDir = applySpread(aimDir, spread)

    if def.projectile then
        spawnProjectile(player, aimDir, def)
    else
        -- Hitscan
        local origin = player.Character.Head.Position
        local result = workspace:Raycast(origin, aimDir.Unit * def.range, makeRaycastParams(player))
        if result then
            local hum, isHead = parseHit(result)
            if hum then
                local damage = def.damage * (isHead and def.headshotMultiplier or 1)
                Combat.ApplyDamage({
                    attacker = player, victim = hum, amount = damage,
                    damageType = "Physical",
                })
            end
        end
    end

    -- Bump spread
    gunState.currentSpread = math.min(def.maxSpread, gunState.currentSpread + def.spreadPerShot)

    -- Apply recoil to client (UI only; aim direction came from client predicted post-recoil too)
    fireRecoilToClient(player, def.recoilVertical, def.recoilHorizontal)
end
```

## Reload

```lua
local function reload(player: Player, gunState: GunState, def: GunDef)
    if gunState.isReloading then return end
    if gunState.inMagazine == def.magazineSize then return end
    if gunState.reserveAmmo == 0 then return end

    gunState.isReloading = true
    playReloadAnim(player)
    playReloadSound(player)

    task.delay(def.reloadTimeSec, function()
        if not player.Parent then return end
        if not gunState.isReloading then return end    -- cancelled

        local needed = def.magazineSize - gunState.inMagazine
        local taken = math.min(needed, gunState.reserveAmmo)
        gunState.inMagazine += taken
        gunState.reserveAmmo -= taken
        gunState.isReloading = false

        pushAmmoUpdate(player, gunState)
    end)
end
```

To support **reload cancellation** (sprinting, switching weapons, dying), set `gunState.isReloading = false` and the deferred task no-ops.

For **chambered-round** reload (one extra in chamber when you reload not-empty), increase magazine cap by 1 conditionally:
```lua
local cap = def.magazineSize + (gunState.inMagazine > 0 and 1 or 0)
```

## Recoil — camera kick

Recoil is **client-side feel** — the server doesn't shake the camera; it only validates that aim was plausible. The client applies a temporary camera CFrame offset on each shot:

```lua
-- LocalScript
local function applyRecoil(vertical: number, horizontal: number)
    local cam = workspace.CurrentCamera
    local kick = CFrame.Angles(
        math.rad(vertical),                     -- pitch up
        math.rad(horizontal * (math.random() * 2 - 1)),  -- random yaw
        0
    )
    -- Apply over a few frames, then recover
    -- (use TweenService on a NumberValue or direct CFrame manipulation in RenderStepped)
end
```

For **realistic recoil patterns** (CS-style), use a fixed pattern table per gun: shot 1 = up, shot 2 = up-right, shot 3 = up-left, etc. Players learn to compensate.

## Spread — accuracy degradation

`currentSpread` increases per shot, recovers over time:
```lua
RunService.Heartbeat:Connect(function(dt)
    for _, gunState in activeGuns do
        gunState.currentSpread = math.max(def.baseSpread, gunState.currentSpread - def.spreadRecoverPerSec * dt)
    end
end)

-- Apply to aim direction:
local function applySpread(direction: Vector3, spreadDeg: number): Vector3
    local pitch = math.rad((math.random() * 2 - 1) * spreadDeg)
    local yaw = math.rad((math.random() * 2 - 1) * spreadDeg)
    return (CFrame.lookAt(Vector3.zero, direction) * CFrame.Angles(pitch, yaw, 0)).LookVector
end
```

ADS reduces spread by `adsAccuracyBonus` (multiply, e.g. 0.5 = halve effective spread). Crouching could add another multiplier.

## ADS — aim down sights

Toggle via Right-Mouse-Button or gamepad trigger:
```lua
-- LocalScript
local function setAds(active: boolean, def: GunDef)
    gunState.isAdsActive = active
    AdsRemote:FireServer(active)            -- mirror to server for accuracy calc

    local cam = workspace.CurrentCamera
    local targetFov = active and def.adsZoomFov or 70
    TweenService:Create(cam, TweenInfo.new(0.2), {FieldOfView = targetFov}):Play()

    -- Sway character speed
    if active then
        humanoid.WalkSpeed = baseWalkSpeed * def.adsSpeedMult
    else
        humanoid.WalkSpeed = baseWalkSpeed
    end
end
```

For **scope overlay UI** (crosshair frame, scope mask), show a ScreenGui with a transparent ring covering peripheral vision. See [ui-system.md](ui-system.md).

## Projectile vs hitscan

| Choice | Pros | Cons |
|---|---|---|
| **Hitscan** (raycast) | Cheap, instant, easy to validate | Unrealistic for slow rounds (arrows, grenades); no leading required |
| **Projectile** (simulated bullet) | Realistic ballistics, dodgeable, visible | More complex; per-tick travel + collision check |

For projectiles, use **`FastCastRedux`** (community library, see [community-libraries.md](community-libraries.md)) — it handles fast-moving projectile collision detection, gravity, lifetime, and visual interpolation in 200 lines. Rolling your own works but is fiddly to get right at high speeds.

```lua
-- FastCast skeleton
local Caster = FastCast.new()
Caster:Fire(origin, direction, velocity, behavior)
Caster.RayHit:Connect(function(activeCast, raycastResult)
    -- Apply damage at hit point
end)
```

## Kill feed

When a player kills another, broadcast a small UI banner: "Adam → Wolf via AssaultRifle". This is a [live-ops.md](live-ops.md) game-wide notification:

```lua
local function broadcastKill(killer: Player, victim: string, weapon: string, headshot: boolean)
    LiveOps.Publish("KillFeed", {
        killer = killer.DisplayName,
        victim = victim,
        weapon = weapon,
        headshot = headshot,
    })
end

LiveOps.Subscribe("KillFeed", function(data)
    for _, p in Players:GetPlayers() do
        KillFeedRemote:FireClient(p, data)
    end
end)

-- Client renders a stacked feed of recent kills (last 5, fade out after 5s)
```

## Tracers

Visual line from gun barrel to hit point (or path of projectile). Use a `Beam` from a barrel `Attachment` to a hit `Attachment`, fade out via TweenService. See [vfx.md](vfx.md).

```lua
local function showTracer(barrelAttachment: Attachment, hitPos: Vector3)
    local hitAttach = Instance.new("Attachment")
    hitAttach.WorldPosition = hitPos
    hitAttach.Parent = workspace.Terrain

    local beam = Instance.new("Beam")
    beam.Attachment0 = barrelAttachment
    beam.Attachment1 = hitAttach
    beam.Texture = "rbxassetid://..."
    beam.Width0 = 0.1; beam.Width1 = 0.1
    beam.Color = ColorSequence.new(Color3.new(1, 0.8, 0.4))
    beam.Parent = barrelAttachment.Parent

    Debris:AddItem(beam, 0.1)
    Debris:AddItem(hitAttach, 0.1)
end
```

## Ammo HUD

Client UI showing `inMagazine / reserveAmmo`, with reload progress when reloading. Push state from server on every change so the UI matches truth:

```lua
AmmoUpdateRemote.OnClientEvent:Connect(function(state)
    ammoLabel.Text = string.format("%d / %d", state.inMagazine, state.reserveAmmo)
    if state.isReloading then showReloadAnim() end
end)
```

## Pitfalls

- **Damage applied client-side.** Trivially exploitable. Server fires the raycast and applies damage; client only sends aim direction.
- **No anti-cheat aim validation.** Players can claim aim straight at heads. Add server-side checks: aim direction roughly matches character orientation; first-shot-of-burst at full accuracy is fine; rapid identical headshots flag for review.
- **Hitscan from camera position vs gun barrel.** Camera is what the player sees but the muzzle is offset. Most games hitscan from camera (matches crosshair) and visually start the tracer at the barrel.
- **Spread purely client-side.** Players run a no-spread mod and shoot lasers. Server applies spread; client predicts.
- **Reload cancel mid-animation that doesn't reset state.** Player switches weapon during reload — old reload completes anyway, ammo gets added to a different weapon. Cancel by setting `isReloading = false` *before* checking it inside the deferred task.
- **Fire rate cap allows burst-fire exploit.** Client sends fire requests faster than rate; server must enforce `now - lastFireAt >= minInterval` (the example does).
- **Recoil on the server.** Camera shake is feel; the server doesn't have a camera. Don't try to "apply recoil server-side."
- **ADS speed not restored on weapon switch.** Switching weapons mid-ADS leaves WalkSpeed at the reduced value forever. Reset on weapon equip/unequip.
- **No infinite-ammo cap on reserves.** A player picking up ammo packs without cap eventually overflows. Cap reserveAmmo per-gun.
- **Tracer for every shot at high fire rates.** Auto-rifle at 10 RPS = 10 tracers/sec all overlapping. Spawn tracer only every 3rd shot for visual clarity.
- **Recoil pattern that's identical every shot.** Even with random horizontal, players notice. Use either truly random or a hand-tuned pattern; both are valid.
- **Headshot detection by checking `result.Instance.Name == "Head"`.** Brittle if part is renamed. Tag head parts or check the part is a child named "Head" with a Humanoid sibling.
- **No friendly-fire filter.** Teammates kill each other in PvP modes. Filter by `player.Team == victim.Team` and skip damage.
- **Kill feed posting before the kill is confirmed.** Race: "Adam killed Bob" then Bob actually survives. Post in `humanoid.Died` handler, not in damage application.

## See also
- [tools.md](tools.md) — gun is a Tool; Equipped/Activated for fire input
- [combat.md](combat.md) — Combat.ApplyDamage; the foundation
- [raycasting.md](raycasting.md) — hitscan ray queries
- [security.md](security.md) — server-side validation; never trust client hits
- [characters.md](characters.md) — character WalkSpeed for ADS, custom locomotion (sprint cancels reload)
- [input-handling.md](input-handling.md) — bind RMB for ADS, R for reload, etc., cross-platform
- [camera.md](camera.md) — FOV manipulation for ADS
- [vfx.md](vfx.md) — muzzle flash, tracers, hit-impact particles
- [sound.md](sound.md) — gunshot, reload, click-empty, hit-marker
- [tweens-animation.md](tweens-animation.md) — recoil recovery, ADS FOV transition
- [equipment.md](equipment.md) — Weapon slot integration
- [live-ops.md](live-ops.md) — kill-feed broadcast pattern
- [ui-system.md](ui-system.md) — ammo HUD, scope overlay
- [community-libraries.md](community-libraries.md) — `FastCastRedux` for projectile guns
