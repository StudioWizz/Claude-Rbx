# Progression — XP, Levels, Prestige, Multipliers

> Source: distilled from RPG/simulator/idle-game progression patterns

## Purpose
Most Roblox games measure player advancement: kills earn XP, XP fills a level bar, leveling up unlocks content or grants attribute points. Idle/simulator games go further with **prestige/rebirth** (reset progress for permanent multipliers) and complex **multiplier stacks** (base × upgrade × pet × VIP × event). This card covers the canonical XP-and-level system, level curve formulas, prestige loops, and the right way to compose multipliers without losing track. For per-stat allocation systems, see [skill-trees.md](skill-trees.md). For the displayed scoreboard, see [leaderstats.md](leaderstats.md).

## XP and levels — the core loop

```lua
--!strict
local Progression = {}

-- Level curve: how much XP to advance from level L to L+1
-- Common shapes:
local function xpForNextLevel(level: number): number
    -- Quadratic (RuneScape-style): XP grows quickly
    return math.floor(100 * level ^ 1.5)
end

-- Convert total XP → current level
local function levelFromXp(totalXp: number): (number, number, number)
    local level = 1
    local accum = 0
    while true do
        local need = xpForNextLevel(level)
        if accum + need > totalXp then
            return level, totalXp - accum, need        -- level, xp into this level, xp needed
        end
        accum += need
        level += 1
    end
end

function Progression.GrantXp(profile: PlayerProfile, amount: number): {leveledUp: boolean, newLevel: number}
    if amount <= 0 then return {leveledUp = false, newLevel = profile.level} end
    local oldLevel = profile.level
    profile.totalXp += amount
    local newLevel = levelFromXp(profile.totalXp)
    local leveledUp = newLevel > oldLevel
    profile.level = newLevel
    return {leveledUp = leveledUp, newLevel = newLevel}
end

return Progression
```

## Profile shape

```lua
profile.level = 1
profile.totalXp = 0           -- cumulative; level derives from this
profile.attributePoints = 0   -- granted per level; spent on skills/attributes
profile.prestige = 0          -- "rebirths" / "ascensions"
```

Storing `totalXp` (not `currentXp + level`) lets you re-derive level from one source of truth. Resists drift bugs.

## Multiple parallel XP pools (multi-skill systems)

Some genres track several independent levels per player — Combat / Magic / Crafting / Mining / Cooking each with its own curve and skill points. RuneScape, OldSchool, Hypixel SkyBlock, and many MMO-likes work this way. The pattern: replace the single `level`/`totalXp` fields with a keyed table.

```lua
profile.skills = {
    Combat   = {totalXp = 0, level = 1, attributePoints = 0},
    Magic    = {totalXp = 0, level = 1, attributePoints = 0},
    Crafting = {totalXp = 0, level = 1, attributePoints = 0},
    Mining   = {totalXp = 0, level = 1, attributePoints = 0},
    Cooking  = {totalXp = 0, level = 1, attributePoints = 0},
}
profile.totalLevel = 5      -- sum of all skill levels (often displayed as the player's "score")
```

`Progression.GrantXp` becomes skill-aware:
```lua
function Progression.GrantXp(profile, skillId: string, amount: number): {leveledUp: boolean, newLevel: number}
    local skill = profile.skills[skillId]
    if not skill or amount <= 0 then return {leveledUp = false, newLevel = skill and skill.level or 0} end

    local oldLevel = skill.level
    skill.totalXp += amount
    skill.level = levelFromXp(skill.totalXp)

    if skill.level > oldLevel then
        profile.totalLevel += (skill.level - oldLevel)
        skill.attributePoints += (skill.level - oldLevel)
    end

    return {leveledUp = skill.level > oldLevel, newLevel = skill.level}
end
```

Hooks call with the right skill: combat kills grant Combat XP; mining a node grants Mining XP; crafting an item grants Crafting XP. See [combat.md](combat.md), [crafting.md](crafting.md), [world-mechanics.md](world-mechanics.md) (resource nodes).

Each skill can use a different curve (Combat quadratic, Mining linear) — extend `xpForNextLevel` to take a skill id and look up the per-skill curve.

For UI: a skills tab in the menu showing each skill's level, XP bar, and total. Display per-skill leaderstats only if you want every skill in the top-right scoreboard (otherwise show `totalLevel` as the single column).

## Level curve choices

Pick a shape that matches your game's pacing:

| Curve | Formula | Feel |
|---|---|---|
| **Linear** | `100 * level` | Constant difficulty; rare in games |
| **Quadratic** | `100 * level^1.5` to `100 * level^2` | Moderate ramp; classic RPG |
| **Exponential** | `100 * 1.15^level` | Steep wall; idle/simulator-style |
| **Stepped** | `level <= 10 → 100; <=20 → 200; ...` | Designer-tuned per band |

Curve shape *is* the difficulty curve. Quadratic is the safe default; idle games use exponential because resource generation is exponential too.

### Soft caps

A pure quadratic or exponential curve eventually breaks: at level 100 you need 30 hours of grinding for the next level, at 200 you need 300. **Soft caps** layer a stricter ramp on top of the base curve past chosen thresholds, signalling "the easy levels are over":

```lua
local SOFT_CAPS = {
    {fromLevel = 50,  multiplier = 2},      -- past 50, XP needed × 2
    {fromLevel = 100, multiplier = 3},      -- past 100, additional × 3 (compounded ⇒ ×6 from baseline)
    {fromLevel = 200, multiplier = 5},      -- past 200, additional × 5 (compounded ⇒ ×30)
}

local function xpForNextLevel(level: number): number
    local base = 100 * level ^ 1.5
    for _, cap in SOFT_CAPS do
        if level >= cap.fromLevel then base *= cap.multiplier end
    end
    return math.floor(base)
end
```

UX-wise, **tell the player about soft caps** — show a small icon on the XP bar past the cap, or change the bar color, so they understand why progress slowed.

For a hard cap (no progress past level X), simply early-return:
```lua
if profile.level >= MAX_LEVEL then return {leveledUp = false, newLevel = MAX_LEVEL} end
```

## Level-up — what happens

On `leveledUp = true`:
- Replenish health/mana to max
- Grant attribute points or skill points
- Play VFX / sound (see [vfx.md](vfx.md), [sound.md](sound.md))
- Toast notification (see [live-ops.md](live-ops.md))
- Update leaderstats column
- Check for unlock-gated content (new area, new spell, new tier of items)

```lua
local function onLevelUp(player: Player, oldLevel: number, newLevel: number)
    local profile = PlayerData.Get(player); if not profile then return end
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    if hum then hum.Health = hum.MaxHealth end

    profile.attributePoints += (newLevel - oldLevel) * ATTRIBUTE_POINTS_PER_LEVEL

    LevelUpVfx:FireClient(player, newLevel)
    LiveOps.PublishLocal(player, string.format("Level up! You are now level %d", newLevel))

    if UNLOCKS_AT_LEVEL[newLevel] then
        unlockContent(player, UNLOCKS_AT_LEVEL[newLevel])
    end
end
```

## Multi-level-up in one grant

A big XP grant could push the player up several levels at once. The `Progression.GrantXp` above handles this naturally (level is re-derived from total). Fire `onLevelUp(oldLevel, newLevel)` once with the spread, OR fire it per-level if you want VFX to play multiple times.

## Level milestones

Beyond per-level rewards (HP refill, attribute points), every Nth level can grant a **milestone reward**: a title, cosmetic, ability slot, inventory expansion, special unlock. Milestones give the long climb to high level memorable peaks.

```lua
local MILESTONES = {
    [5]   = {kind = "title",     value = "Apprentice"},
    [10]  = {kind = "ability",   value = "DoubleJump"},
    [25]  = {kind = "cosmetic",  value = "BronzeCape"},
    [50]  = {kind = "cosmetic",  value = "SilverCape"},
    [50]  = {kind = "petSlot",   value = 1},               -- +1 pet slot
    [100] = {kind = "title",     value = "Master"},
    [100] = {kind = "cosmetic",  value = "GoldenCape"},
}

local function checkMilestones(profile: PlayerProfile, oldLevel: number, newLevel: number)
    for level = oldLevel + 1, newLevel do
        local milestone = MILESTONES[level]
        if milestone then grantMilestone(profile, milestone) end
    end
end
```

Call `checkMilestones` from `onLevelUp`. For multi-level-up grants, this naturally awards every milestone the player crossed.

For "every 10 levels gets a generic reward" (no per-level table needed):
```lua
if newLevel % 10 == 0 then grantSomething(profile, newLevel / 10) end
```

UX: announce milestones with a bigger toast than ordinary level-ups — see [live-ops.md](live-ops.md). For cosmetic unlocks, push a "New cosmetic available!" notification routing to the wardrobe menu.

## XP grant modifiers

Modifiers shape how much XP the player actually receives from a given action — separate from the multiplier stack (which is a player-level boost). Modifiers depend on context: who killed what, who's nearby, what level the source is.

### Level-difference scaling

If a level-50 player farms level-1 mobs, they shouldn't get full XP per kill — that breaks the curve and makes leveling trivial. Conversely, defeating something well above your level deserves a bonus.

```lua
local function levelDiffMultiplier(playerLevel: number, sourceLevel: number): number
    local diff = sourceLevel - playerLevel
    if diff >= 5 then return 1.5         -- target much higher → +50%
    elseif diff >= 0 then return 1
    elseif diff >= -5 then return 0.7    -- slightly lower
    elseif diff >= -10 then return 0.3
    else return 0.05                      -- trivial → near-zero (but not zero, so achievement counters work)
    end
end

-- In the XP-grant hook:
local source = killedNpc                   -- has GetAttribute("Level")
local sourceLevel = source:GetAttribute("Level") or playerLevel
local mult = levelDiffMultiplier(profile.level, sourceLevel)
Progression.GrantXp(profile, baseXp * mult)
```

Tune the curve by genre: hardcore RPGs cliff hard (zero at -10); casual games soften (50% even at -10).

### Group / party XP sharing

When multiple players contribute to a kill (or are nearby when a quest objective triggers), share the XP. The standard rule: split the base XP across contributors, with a small **party bonus** so grouping is *more* rewarding than soloing.

```lua
local PARTY_BONUS = 0.1     -- +10% per extra party member, capped

local function distributePartyXp(contributors: {Player}, baseXp: number)
    local n = #contributors
    if n == 0 then return end
    local share = baseXp / n
    local bonus = 1 + math.min(PARTY_BONUS * (n - 1), 0.5)    -- cap at +50%
    local each = share * bonus

    for _, p in contributors do
        local profile = PlayerData.Get(p); if not profile then continue end
        Progression.GrantXp(profile, each)
    end
end
```

For "contributors" defined by **damage dealt**: track per-kill damage attribution, distribute proportional to contribution share. For "nearby on objective complete": query `workspace:GetPartBoundsInRadius` around the objective trigger and include any party members.

For party identification — track player parties in your own table (typically formed via a UI invite/accept flow), or treat "Team" as the party.

### Catch-up mechanics

Low-level players grouped with high-level friends should level fast enough to feel useful, not feel like a baby on a leash. Apply a per-player catch-up multiplier when the level gap to nearby teammates is large:

```lua
local function catchUpMultiplier(profile: PlayerProfile, nearbyPlayers: {Player}): number
    if #nearbyPlayers == 0 then return 1 end

    local highest = profile.level
    for _, p in nearbyPlayers do
        local other = PlayerData.Get(p)
        if other and other.level > highest then highest = other.level end
    end

    local gap = highest - profile.level
    if gap < 5 then return 1 end
    return 1 + math.min(gap * 0.05, 1.5)    -- +5% per level of gap, capped at +150%
end
```

Apply alongside the base XP grant. Combined with party-XP sharing, the friend gets normal XP, the new player levels at 2.5× normal — fast catch-up without interfering with the high-level player's pacing.

### Group / catch-up combine
```lua
local nearby = findNearbyPartyMembers(player, 50)    -- studs
local levelMult = levelDiffMultiplier(profile.level, sourceLevel)
local catchUp = catchUpMultiplier(profile, nearby)
local actual = baseXp * levelMult * catchUp
Progression.GrantXp(profile, actual)
```

Tune so even with all bonuses combined, leveling can't be more than ~3× normal speed — beyond that, the curve loses meaning.

## Death penalties

In casual games, dying just respawns. In harder games, dying costs XP (or even levels) — increases the weight of decisions and recovery loops.

### Soft penalty: lose XP toward current level

```lua
local DEATH_XP_LOSS_FRACTION = 0.1     -- lose 10% of XP earned toward current level

local function applyDeathPenalty(profile: PlayerProfile)
    local _, xpInLevel, _ = levelFromXp(profile.totalXp)
    local lost = math.floor(xpInLevel * DEATH_XP_LOSS_FRACTION)
    profile.totalXp = math.max(0, profile.totalXp - lost)
    -- Note: never let totalXp drop you below current level (no level-down via this rule)
    return lost
end
```

Show a "Lost X XP" toast on respawn. Soft penalties don't level-down — the player loses progress *within* the current level only.

### Hard penalty: lose levels (hardcore mode)
For hardcore / soulslike, allow the loss to drop you down a level:
```lua
local function applyHardDeathPenalty(profile: PlayerProfile)
    local _, xpInLevel, _ = levelFromXp(profile.totalXp)
    local lost = xpInLevel + math.floor(xpForNextLevel(profile.level - 1) * 0.5)
    profile.totalXp = math.max(0, profile.totalXp - lost)
    profile.level = levelFromXp(profile.totalXp)
    return lost
end
```

### Recoverable on death-spot return (soulslike)
Drop the lost XP at the death location as a recoverable corpse / "soul." If the player returns within X minutes (or before dying again), they reclaim it.

```lua
profile.lostSoul = {
    xp = lost,
    position = deathPosition,
    expiresAt = os.time() + 600,     -- 10 minutes
}

-- On reaching the spot:
local hrp = player.Character.HumanoidRootPart
if profile.lostSoul and (hrp.Position - profile.lostSoul.position).Magnitude < 5 then
    Progression.GrantXp(profile, profile.lostSoul.xp)
    profile.lostSoul = nil
end
```

If they die again before reclaiming, the soul is lost permanently (soulslike convention) OR it's replaced by the new death's soul (Bloodborne convention) — designer choice.

### Choose the right severity
- **Casual / family games**: no penalty (or a few seconds of respawn delay).
- **Mid-difficulty PvE**: soft penalty (10% of current-level XP).
- **Hardcore / soulslike**: recoverable-corpse soul system.
- **Permadeath roguelike**: full account / character reset on death (separate save-slot or run system).

Penalty severity directly shapes player risk-taking — a 50% loss makes players turtle; a 5% loss is barely noticed.

## Prestige / rebirth (idle/simulator core loop)

Prestige resets progress in exchange for a permanent multiplier or unlock. The pattern:

```lua
local function canPrestige(profile: PlayerProfile): boolean
    return profile.level >= PRESTIGE_LEVEL_REQUIREMENT
end

local function prestige(profile: PlayerProfile)
    if not canPrestige(profile) then return end

    -- Grant prestige rewards
    profile.prestige += 1
    profile.prestigeMultiplier = 1 + (profile.prestige * 0.1)    -- +10% per prestige

    -- Reset progression (but keep prestige-tier rewards)
    profile.level = 1
    profile.totalXp = 0
    profile.coins = 0
    profile.inventory = createFreshInventory()
    -- Keep: prestige count, prestige multiplier, cosmetics, achievements
end
```

The trade — give up tangible progress for a multiplier — fuels the idle-game retention loop. Designers tune the multiplier so each prestige feels worth it but doesn't trivialize the next loop.

### Multiple prestige tiers

For deeper games, support **prestige currencies**:
- Tier 1: rebirth → +Multiplier (resets level)
- Tier 2: ascend → +Bigger Multiplier (resets prestige)
- Tier 3: transcend → +Permanent unlocks (resets ascension)

Each tier resets the previous and grants a different scarcer currency.

## Multiplier stack — the right shape

The classic bug: 5 different scripts each set `coinMultiplier = X` and they overwrite each other. Solution: **never store the final multiplier**; store the *contributing factors* and compute on read.

```lua
type MultiplierContext = {
    base: number,                     -- 1
    permanent: number,                -- prestige bonus
    gamepass: number,                 -- VIP gamepass owner
    pets: number,                     -- equipped pet boost
    event: number,                    -- live double-XP event
    boost: {[string]: {mult: number, expiresAt: number}},  -- temporary stacked boosts
}

local function getCoinMultiplier(profile: PlayerProfile): number
    local now = os.time()
    local ctx = profile.multiplierContext

    -- Sweep expired boosts
    for k, b in ctx.boost do
        if b.expiresAt < now then ctx.boost[k] = nil end
    end

    local mult = ctx.base
    mult *= ctx.permanent
    mult *= ctx.gamepass
    mult *= ctx.pets
    mult *= ctx.event

    for _, b in ctx.boost do
        mult *= b.mult
    end

    return mult
end

-- Whenever you grant currency, multiply through:
function awardCoins(player: Player, baseAmount: number)
    local profile = PlayerData.Get(player); if not profile then return end
    local actual = math.floor(baseAmount * getCoinMultiplier(profile))
    profile.coins += actual
    -- ... show "+100 coins (×4.5)" toast
end
```

Each contributor lives in its own field and is updated only by its own logic (gamepass check, equip-pet handler, event start, etc.). You never overwrite the wrong one because they're additive (each multiplied in) rather than replacing each other.

## Temporary boosts (consumable / time-limited)

```lua
local function applyBoost(profile: PlayerProfile, key: string, mult: number, durationSec: number)
    profile.multiplierContext.boost[key] = {
        mult = mult,
        expiresAt = os.time() + durationSec,
    }
end

-- "Drink XP Potion" → applyBoost(profile, "xpPotion", 2.0, 600)
```

Boosts are keyed so re-applying the same key replaces (extends) the existing one rather than stacking infinite copies. For separate-stack semantics (5 potions all multiply), use unique keys per consume.

## Pets in the multiplier stack

Pets typically contribute multiplicatively. Aggregate equipped pet multipliers into `ctx.pets`:

```lua
local function recomputePetMultiplier(profile: PlayerProfile)
    local mult = 1
    for _, petKey in profile.equippedPets do
        local pet = profile.pets[petKey]
        if pet then mult *= (pet.coinMultiplier or 1) end
    end
    profile.multiplierContext.pets = mult
end

-- Call this whenever pets change (equip/unequip/upgrade/fuse)
```

See [pets.md](pets.md).

## Skill / attribute points

When the player levels up, grant points to spend. Points buy specific upgrades:

```lua
profile.attributePoints = 5

-- Player chooses to spend 1 on Strength
allocateRemote.OnServerEvent:Connect(function(player, attribute)
    if typeof(attribute) ~= "string" then return end
    local profile = PlayerData.Get(player); if not profile then return end
    if profile.attributePoints <= 0 then return end
    if not VALID_ATTRIBUTES[attribute] then return end

    profile.attributePoints -= 1
    profile.attributes[attribute] = (profile.attributes[attribute] or 0) + 1
    -- Recompute derived stats (max HP, damage, etc.)
end)
```

For tree-shaped allocation with prerequisites, see [skill-trees.md](skill-trees.md).

## Patterns

### Hook XP into combat
```lua
-- combat.md fires this when an enemy dies
EnemyDied:Connect(function(killer: Player, enemyType: string)
    local xp = ENEMY_XP[enemyType] or 10
    local result = Progression.GrantXp(profile, xp * getXpMultiplier(profile))
    if result.leveledUp then onLevelUp(killer, oldLevel, result.newLevel) end
end)
```

### Display level bar
A UI bar showing "Level 12 — 423/1500 XP" needs the breakdown:
```lua
local level, xpInLevel, xpForNext = levelFromXp(profile.totalXp)
xpBar.Size = UDim2.fromScale(xpInLevel / xpForNext, 1)
levelLabel.Text = string.format("Level %d  (%d/%d)", level, xpInLevel, xpForNext)
```

Push these to client whenever XP changes (don't recompute client-side from raw `totalXp` and risk drift with custom curves — push the parsed values).

### Prestige UI
Show "Prestige: ★ ★ ★" stars in leaderstats via a `StringValue`; full prestige menu in a dedicated UI showing current multipliers and "next prestige unlocks" preview.

### Floating XP-gain numbers

The "+50 XP" floating up from where the kill happened is small but **dramatically** improves game-feel. Players feel rewarded with each gain, even before the level-up.

```lua
-- Server tells the client where + how much
XpGainedRemote:FireClient(player, {
    amount = actualGained,
    worldPos = killPosition,
    color = Color3.fromRGB(180, 220, 100),
})

-- Client renders a floating BillboardGui that drifts up + fades
XpGainedRemote.OnClientEvent:Connect(function(data)
    local anchor = Instance.new("Part")
    anchor.Anchored = true
    anchor.CanCollide = false; anchor.CanQuery = false
    anchor.Transparency = 1
    anchor.Size = Vector3.one
    anchor.Position = data.worldPos
    anchor.Parent = workspace.Effects

    local bb = Instance.new("BillboardGui", anchor)
    bb.Size = UDim2.fromOffset(120, 32)
    bb.AlwaysOnTop = true
    bb.StudsOffset = Vector3.new(0, 0, 0)

    local label = Instance.new("TextLabel", bb)
    label.BackgroundTransparency = 1
    label.Size = UDim2.fromScale(1, 1)
    label.Font = Enum.Font.BuilderSansBold
    label.TextSize = 22
    label.TextColor3 = data.color or Color3.fromRGB(180, 220, 100)
    label.TextStrokeTransparency = 0.4
    label.Text = string.format("+%d XP", data.amount)

    -- Drift up + fade
    local TweenService = game:GetService("TweenService")
    TweenService:Create(bb, TweenInfo.new(1.2), {StudsOffset = Vector3.new(0, 4, 0)}):Play()
    TweenService:Create(label, TweenInfo.new(1.2), {TextTransparency = 1, TextStrokeTransparency = 1}):Play()

    game:GetService("Debris"):AddItem(anchor, 1.3)
end)
```

Variants:
- **Color by source**: combat XP green, crafting XP blue, quest XP gold.
- **Critical hits / crits**: bigger TextSize and a different color for crit-XP gains.
- **Stack identical gains**: if multiple +50 XP fire in the same 0.3s window, coalesce into "+150 XP" instead of three overlapping labels.
- **Multi-skill display**: "+25 Combat / +10 Mining" for actions that grant several skills at once.

For non-XP gains (currency drops, item pickups), use the same primitive — it's a generic floating-text-over-world pattern.

## Pitfalls

- **Storing both `level` and `xpInLevel` and trying to keep them in sync.** Drift bugs are inevitable. Store only `totalXp`; derive everything else.
- **Granting XP client-side.** Trivially exploitable. Server only.
- **Per-frame XP grants.** Save on event boundaries (kill, quest turn-in, level up), not every Heartbeat. The DataStore won't survive otherwise.
- **Multiplier stack as a single field.** Five scripts overwrite each other. Always store contributors separately.
- **Floor before multiply, not after.** `math.floor(amount * mult)` is right. `math.floor(amount) * mult` rounds away the multiplier benefit.
- **Curve creates 32-bit overflow at high levels.** Exponential curves hit `2^31` sooner than you'd expect. Cap the level (level 9999 max), or move to `NumberValue` (double precision).
- **Prestige reset that doesn't actually reset.** Forgetting to reset coins or inventory leaves the prestige with no scarcity. Define exactly what resets and what survives in one place.
- **Rebirth requirement too low.** Players prestige immediately, multiplier hits unmanageable values. Tune the requirement and multiplier per-prestige carefully; lock prestige #1 behind a meaningful achievement.
- **Boost not expiring.** Sweep expired boosts on every read OR on a periodic Heartbeat. Don't assume a Heartbeat sweep alone runs while the player is offline.
- **Multi-level-up firing onLevelUp many times rapidly.** A single huge XP grant pumps 10 toasts in a row. Coalesce into "Level up! 10 → 15".
- **Attribute points spent client-side.** All allocation through server validation. Server is the only place that knows the current `attributePoints`.
- **Multi-pool XP without per-pool curves.** Same curve for Combat (combat-heavy gameplay → fast XP) and Cooking (rare action → slow XP) makes Cooking feel hopeless. Tune per-skill.
- **Soft cap surprise.** Players who hit a soft cap and don't know why feel like they're broken. UI label or color shift on the bar communicates "you're past the cap."
- **Milestone reward overlap.** Two milestones at the same level (rare, but happens with a typo'd table) — one silently overrides. Use a list-of-rewards per level instead.
- **Level-difference scaling so harsh players abandon under-level content.** "Trivial" XP (5%) is the floor most games use; zero-XP makes the world feel broken at high level. Keep the floor non-zero.
- **Party XP with no contributor tracking.** "Anyone in the party gets full XP" → grief by inviting AFK alts. Track contribution (damage dealt, presence at objective), distribute proportionally.
- **Catch-up multiplier stacking with double-XP event.** Combined multipliers can grant 5×, 10×, 20× XP. Cap the total multiplier (`math.min(combinedMult, MAX_MULT)`).
- **Death penalty on top of low respawn time = misery loop.** If you respawn in 2 seconds and lose 10% XP, you're worse off after a small mistake than dying once carefully. Either soft-penalty + slow respawn, or harsh-penalty + safe respawn — not both.
- **Recoverable soul that despawns silently.** Player runs back, sees nothing. Always show "Soul recovered: +250 XP" or "Soul lost forever" — never silent state changes.
- **Floating XP numbers spawning into Workspace forever.** Forgot Debris cleanup → memory leak. Always `Debris:AddItem` the anchor.
- **Floating XP-number spam at high fire rates.** A machine-gun killing minions fires 10 numbers/sec, each overlapping. Coalesce within a 0.3s window into "+150 XP".

## See also
- [datastores.md](datastores.md) — persisting profile fields
- [leaderstats.md](leaderstats.md) — Level + XP columns commonly displayed
- [skill-trees.md](skill-trees.md) — branching attribute allocation with prerequisites
- [combat.md](combat.md) — XP grants on kill (level-difference scaling applies here)
- [quests.md](quests.md) — XP rewards on quest turn-in
- [crafting.md](crafting.md), [world-mechanics.md](world-mechanics.md) — sources of multi-pool XP (Crafting, Mining, Gathering)
- [pets.md](pets.md) — pet contribution to multiplier stack
- [monetization.md](monetization.md) — VIP gamepass contributing to multiplier stack
- [live-ops.md](live-ops.md) — event-driven multipliers (double XP weekend); level-up toasts; milestone announcements
- [ui-system.md](ui-system.md) — XP bar and level display
- [resource-bars.md](resource-bars.md) — XP bar as part of the unified resource-bar pattern
- [spawn-mechanics.md](spawn-mechanics.md) — death penalties applied alongside the respawn flow
- [save-slots.md](save-slots.md) — progression is per active slot
- [community-libraries.md](community-libraries.md) — ProfileService for the persisted profile
- [player-badges.md](player-badges.md) — Prestige and Level badges sourced from this card's profile fields
- [vip-systems.md](vip-systems.md) — VIP multiplier slot in the multiplier stack
