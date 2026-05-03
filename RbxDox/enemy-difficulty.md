# Enemy Difficulty Scaling

> Source: distilled from RPG / horde / soulslike / roguelike difficulty patterns

## Purpose
Difficulty scaling shapes how challenging the same enemies feel based on player count, area, chosen difficulty mode, and (in some games) dynamic adaptation to player skill. This card covers: discrete difficulty modes (Easy/Normal/Hard/Nightmare), the **stat-multiplier table** that's the core primitive, per-area difficulty (the dragon's lair is harder than the village), **Dynamic Difficulty Adjustment** (DDA — adapt to player performance), loot/XP scaling per difficulty, and achievement-gating ("clear on Nightmare for the title"). Builds on `npcs.md` (the enemy template), `combat.md` (where stats are read), `enemy-spawning.md` (where difficulty is applied at spawn), and `progression.md` (XP scaling).

## The base unit: per-stat multipliers

Every difficulty mode is essentially a table of multipliers applied to enemy base stats. Centralise in one place:

```lua
--!strict
export type DifficultyTier = "Easy" | "Normal" | "Hard" | "Nightmare" | "Endless"

export type DifficultyProfile = {
    name: string,
    healthMult: number,
    damageMult: number,
    speedMult: number,
    detectRangeMult: number,         -- aggro range
    xpMult: number,                  -- XP granted on kill
    lootMult: number,                -- loot drop rate / quality
    coinMult: number,
    requiresUnlock: boolean?,        -- "earn Nightmare by clearing on Hard" gating
}

local DIFFICULTIES: {[DifficultyTier]: DifficultyProfile} = {
    Easy      = {name = "Easy",      healthMult = 0.7, damageMult = 0.7, speedMult = 0.9,
                 detectRangeMult = 0.8, xpMult = 0.7, lootMult = 0.8, coinMult = 0.8},
    Normal    = {name = "Normal",    healthMult = 1.0, damageMult = 1.0, speedMult = 1.0,
                 detectRangeMult = 1.0, xpMult = 1.0, lootMult = 1.0, coinMult = 1.0},
    Hard      = {name = "Hard",      healthMult = 1.5, damageMult = 1.4, speedMult = 1.1,
                 detectRangeMult = 1.2, xpMult = 1.5, lootMult = 1.3, coinMult = 1.5},
    Nightmare = {name = "Nightmare", healthMult = 2.5, damageMult = 1.8, speedMult = 1.2,
                 detectRangeMult = 1.5, xpMult = 2.5, lootMult = 1.8, coinMult = 2.5,
                 requiresUnlock = true},
    Endless   = {name = "Endless",   healthMult = 1.0, damageMult = 1.0, speedMult = 1.0,    -- scales with wave
                 detectRangeMult = 1.0, xpMult = 1.0, lootMult = 1.0, coinMult = 1.0,
                 requiresUnlock = true},
}

return DIFFICULTIES
```

The reward multipliers (`xpMult`, `lootMult`, `coinMult`) generally **scale faster than the difficulty multipliers** — Hard is 1.5× HP but grants 1.5× XP and coins; Nightmare is 2.5× HP but 2.5× XP. This way, harder = better-reward-per-time, which is the genre convention.

## Apply on spawn

When the spawner instantiates an enemy ([enemy-spawning.md](enemy-spawning.md)), call `applyDifficulty` to multiply through:

```lua
local DIFFICULTIES = require(game.ReplicatedStorage.Difficulties)

local function applyDifficulty(enemy: Model, tier: DifficultyTier?)
    tier = tier or "Normal"
    local profile = DIFFICULTIES[tier]; if not profile then return end

    local hum = enemy:FindFirstChildOfClass("Humanoid"); if not hum then return end

    -- Health
    local baseMaxHealth = enemy:GetAttribute("BaseMaxHealth") or hum.MaxHealth
    enemy:SetAttribute("BaseMaxHealth", baseMaxHealth)         -- remember for re-application
    hum.MaxHealth = baseMaxHealth * profile.healthMult
    hum.Health = hum.MaxHealth

    -- Damage (used by combat.md when this enemy attacks)
    local baseDamage = enemy:GetAttribute("BaseDamage") or 10
    enemy:SetAttribute("BaseDamage", baseDamage)
    enemy:SetAttribute("Damage", baseDamage * profile.damageMult)

    -- Speed
    hum.WalkSpeed = (enemy:GetAttribute("BaseWalkSpeed") or 16) * profile.speedMult

    -- AI parameters
    enemy:SetAttribute("DetectRange", (enemy:GetAttribute("BaseDetectRange") or 30) * profile.detectRangeMult)

    -- Mark for downstream systems (loot, XP)
    enemy:SetAttribute("Difficulty", tier)
    enemy:SetAttribute("XpMult", profile.xpMult)
    enemy:SetAttribute("LootMult", profile.lootMult)
    enemy:SetAttribute("CoinMult", profile.coinMult)

    -- Visual: tint by difficulty so players can read it at a glance
    local tints = {
        Easy = Color3.fromRGB(180, 220, 180),
        Normal = nil,
        Hard = Color3.fromRGB(220, 180, 180),
        Nightmare = Color3.fromRGB(220, 100, 100),
    }
    if tints[tier] then
        for _, p in enemy:GetDescendants() do
            if p:IsA("BasePart") then p.Color = tints[tier] end
        end
    end
end
```

The pattern: **store base stats as attributes** so applyDifficulty can be re-run if the difficulty mode changes mid-session. Without `BaseMaxHealth`, you'd permanently inflate the value on each application.

## Reading the multipliers downstream

`combat.md`'s `Combat.ApplyDamage` reads damage from the enemy:
```lua
-- When an enemy attacks
local damage = enemy:GetAttribute("Damage") or 10
Combat.ApplyDamage({attacker = enemyAsAttacker, victim = playerHum, amount = damage, ...})
```

`progression.md`'s XP grant reads the multiplier:
```lua
-- When a player kills an enemy
local baseXp = enemyDef.baseXp or 10
local xpMult = enemy:GetAttribute("XpMult") or 1
Progression.GrantXp(profile, "Combat", baseXp * xpMult)
```

`inventory.md` loot tables (or your own loot system) read `LootMult` to scale drop rates:
```lua
local function rollLoot(enemy: Model)
    local mult = enemy:GetAttribute("LootMult") or 1
    for _, lootEntry in lootTable do
        if math.random() < lootEntry.chance * mult then
            grantItem(killer, lootEntry.itemId, lootEntry.count)
        end
    end
end
```

The multipliers cascade naturally — change the difficulty profile in one place; every system reads the new values.

## Per-area difficulty

Different areas naturally have different difficulty. Tag each area with a `DifficultyTier` ([areas-and-biomes.md](areas-and-biomes.md)); the spawner uses the area's difficulty if its own `Difficulty` attribute is not set:

```lua
local function getEffectiveDifficulty(spawner: BasePart): DifficultyTier
    local override = spawner:GetAttribute("Difficulty")
    if override then return override :: DifficultyTier end

    local areaId = spawner:GetAttribute("AreaId")
    if areaId then
        local areaPreset = AREA_PRESETS[areaId]
        if areaPreset and areaPreset.difficulty then return areaPreset.difficulty end
    end

    return "Normal"
end

-- In trySpawn:
local difficulty = getEffectiveDifficulty(spawner)
applyDifficulty(enemy, difficulty)
```

This makes "the dragon's lair is harder than the village" automatic — set the area's difficulty once, every spawner inside inherits it.

For more granular control, mix tier-based difficulty with **per-enemy level scaling** (see [progression.md](progression.md) level-difference scaling). An enemy in a Forest area at level 5 is fundamentally easier than the same enemy type in a Dungeon area at level 50, regardless of difficulty tier.

## Player-chosen difficulty

For games where the player picks before starting (Diablo-style: Normal → Hard → Nightmare unlocked sequentially), persist the choice on the profile:

```lua
profile.chosenDifficulty = "Normal"
profile.unlockedDifficulties = {Normal = true}    -- Easy/Normal always; Hard/Nightmare earned
```

On level-up or campaign-clear:
```lua
local function unlockDifficulty(profile: PlayerProfile, tier: DifficultyTier)
    profile.unlockedDifficulties[tier] = true
    notifyPlayer(getPlayer(profile), "New difficulty unlocked: " .. tier)
end

-- Trigger on event
QuestSystem.OnComplete("CampaignFinale_Normal"):Connect(function(player)
    unlockDifficulty(getProfile(player), "Hard")
end)
```

When the player enters an area, override the area's default difficulty with their chosen difficulty (within unlocked tiers):
```lua
local function getEffectiveDifficultyForPlayer(spawner: BasePart, player: Player): DifficultyTier
    local profile = PlayerData.Get(player)
    if profile and profile.chosenDifficulty and profile.unlockedDifficulties[profile.chosenDifficulty] then
        return profile.chosenDifficulty
    end
    return getEffectiveDifficulty(spawner)
end
```

For multiplayer with mixed difficulty preferences, pick the average (or max — "the highest difficulty in the party determines the run") and apply uniformly. Letting different players see different stats on the same enemy is confusing.

## Player-count scaling

Co-op gets harder the more players — single enemy faces 1 player vs 4 differently. Multiply enemy stats by player count (slightly):

```lua
local function playerCountMultiplier(): number
    local n = #Players:GetPlayers()
    return 1 + math.max(0, n - 1) * 0.3       -- +30% per extra player above 1
end

-- Apply on top of difficulty:
hum.MaxHealth = baseMaxHealth * profile.healthMult * playerCountMultiplier()
```

For raid bosses specifically, use a steeper curve (`+50% per extra player`); for trash mobs, leaner (`+15%`).

## Dynamic difficulty adjustment (DDA)

DDA quietly adjusts difficulty based on player performance. The patterns:

### Reactive: scale on death streak
```lua
profile.recentDeaths = 0
profile.dynamicDifficultyMod = 1

humanoid.Died:Connect(function()
    profile.recentDeaths += 1
    if profile.recentDeaths >= 3 then
        profile.dynamicDifficultyMod = math.max(0.7, profile.dynamicDifficultyMod - 0.1)    -- get easier
        profile.recentDeaths = 0
    end
end)

-- On enemy kill, recover some difficulty
EnemyKilled:Connect(function(killer)
    local profile = PlayerData.Get(killer)
    profile.dynamicDifficultyMod = math.min(1.2, profile.dynamicDifficultyMod + 0.02)
end)

-- In applyDifficulty, multiply by dynamic mod for the targeting player
hum.MaxHealth = baseMaxHealth * profile.healthMult * profile.dynamicDifficultyMod
```

### Proactive: scale by recent kill rate
Track time-between-kills; if too fast (player is stomping), notch up; if too slow (struggling), notch down. Subtle — players generally shouldn't know DDA is happening.

### Disclosure
DDA is controversial. Some players *want* a fixed challenge and feel cheated by hidden adjustments. Either:
- Hide it entirely and tune subtly (Resident Evil 4 famously did this — players never knew)
- Disclose at game-start ("difficulty adjusts to your skill — disable in settings")

If your game has explicit difficulty modes, DDA is usually OFF at the highest tier (Nightmare = exactly as advertised, no help).

## Loot scaling per difficulty

Beyond `LootMult`, harder difficulties can unlock entirely **new loot pools**:

```lua
local lootByDifficulty = {
    Easy   = {commonOnly = true},
    Normal = {commonOnly = false},
    Hard   = {commonOnly = false, eliteChance = 0.05, legendaryChance = 0.001},
    Nightmare = {commonOnly = false, eliteChance = 0.15, legendaryChance = 0.005, mythicChance = 0.0005},
}

local function rollLoot(enemy: Model, killer: Player)
    local difficulty = enemy:GetAttribute("Difficulty") or "Normal"
    local lootRules = lootByDifficulty[difficulty]
    -- Pick from pool restricted by lootRules
end
```

Players run Nightmare not just for higher quantity but for the *exclusive* drops only available there.

## Achievement / cosmetic gating

Hardest difficulties carry prestige. Award unique titles, badges, cosmetics for clearing on top tiers:

```lua
QuestSystem.OnComplete("CampaignFinale"):Connect(function(player)
    local profile = PlayerData.Get(player)
    if profile.chosenDifficulty == "Nightmare" then
        grantItem(player, "NightmareConqueror_Title")
        AchievementService.Award(player, "Cleared_On_Nightmare")
    end
end)
```

Prestige rewards motivate players to engage with the difficulty system. Make them *cosmetic* (titles, frames, pets) — not gameplay-essential — so easier-difficulty players don't feel locked out of mechanics.

## Endless / scaling difficulty (roguelike)

For rogue-likes and survival modes, difficulty doesn't pick a tier — it scales continuously with progress (wave number, depth, time-survived). Compute the multiplier from a curve:

```lua
local function computeEndlessDifficulty(wave: number): DifficultyProfile
    local scaleFactor = 1 + (wave - 1) * 0.15        -- +15% per wave
    return {
        name = "Wave " .. wave,
        healthMult = scaleFactor,
        damageMult = math.min(scaleFactor, 3),       -- cap damage scaling
        speedMult = math.min(1 + (wave - 1) * 0.05, 1.5),
        detectRangeMult = 1 + (wave - 1) * 0.05,
        xpMult = scaleFactor * 1.2,                   -- reward outpaces challenge
        lootMult = scaleFactor,
        coinMult = scaleFactor,
    }
end

-- Spawner reads current wave from round state
applyDifficulty(enemy, computeEndlessDifficulty(currentWave))
```

Curves to consider:
- **Linear** (`1 + wave * 0.15`): smooth ramp, players hit a wall around wave 30
- **Logarithmic** (`1 + math.log(wave) * 0.5`): early ramp, late-game plateau
- **Stepwise** (every 10 waves a big jump): builds anticipation, "the wave 50 boss is going to be brutal"

Cap *damage* even if you don't cap *health* — at some point the player dies in one shot regardless, which isn't fun. HP can scale uncapped (the player just needs more time per kill).

## UI implications

Show players the difficulty their enemies are at:
- **Difficulty label on enemy nameplate**: "Goblin (Hard)" — colour-coded
- **Visual tint** (as in `applyDifficulty` example): red for Hard, deep red for Nightmare
- **Damage numbers colour-coded** by difficulty: Nightmare hits show in deep red
- **Pre-encounter UI**: "Entering Dragon's Lair (Nightmare)" — banner so players know what they're stepping into

For DDA games, *don't* show the dynamic multiplier — the whole point is invisibility.

## Patterns

### Difficulty-only enemies
Some enemies only spawn on Hard+. Filter the enemy pool:
```lua
local function poolForDifficulty(spawner: BasePart, tier: DifficultyTier): {string}
    local pool = parseList(spawner:GetAttribute("EnemyPool"))
    return filter(pool, function(enemyId)
        local def = ENEMY_DEFS[enemyId]
        return not def.minDifficulty or DIFFICULTY_RANK[tier] >= DIFFICULTY_RANK[def.minDifficulty]
    end)
end
```

### Elite / champion variants
On any difficulty, a small chance an enemy spawns as an "Elite" — same stats × 1.5, distinct visual (glow, name suffix), better loot. Roll on spawn:
```lua
if math.random() < ELITE_CHANCE then makeElite(enemy) end
```

### Scaling boss phases
Boss has multiple HP-threshold-triggered phases (75%, 50%, 25% — each phase changes attack pattern). On harder difficulties, add additional phases or intensify each.

### Difficulty per game mode
Story mode = Normal cap; Hardcore mode = Nightmare with permadeath; Speedrun mode = ignore difficulty, time-only. Profile tracks `gameMode` separately from `chosenDifficulty`.

### Adaptive boss
A boss that uses different attacks based on the player's gear / level / class. Distinct from DDA — this is *responsive*, not adaptive-to-skill. Call `pickBossPattern(player.profile)` in the AI tick.

## Pitfalls

- **Re-applying difficulty without storing base values.** Each application multiplies, so HP drifts upward forever. Store `BaseMaxHealth`, `BaseDamage`, etc. as attributes; applyDifficulty reads from base, not current.
- **Difficulty visible to client and trusted on the server.** Client claims "I'm playing Easy" → server scales down. Server reads from the persisted profile, not the client.
- **Reward multipliers slower than difficulty multipliers.** Hard is 1.5× harder but grants 1.0× rewards → no one plays Hard. Reward should scale at least as fast as challenge.
- **DDA observable to players.** "I'm dying — wait, the enemies got noticeably weaker. The game is patronising me." Subtle adjustments (5-10% per trigger) are invisible; large ones break immersion.
- **DDA reset on respawn.** Player dies once, dynamicDifficultyMod stays low forever. Decay it back toward 1.0 over time / kills.
- **Player-count scaling that's too steep.** A 4-player party can't beat content that 1 player solos comfortably. Test at multiple party sizes.
- **Per-area difficulty in mixed-party content.** Player A is in Easy area, Player B is in Hard area, both shooting the same enemy. Pick one (the enemy's spawner-defined difficulty), don't try to vary per-attacker.
- **Loot table that returns empty on Easy.** "Common only" → some enemies drop nothing. Ensure every difficulty's loot table can produce *something*.
- **Endless difficulty without a damage cap.** Wave 50 enemies one-shot maxxed players. Cap `damageMult` even if you don't cap HP.
- **Achievement gating gameplay-essential rewards.** "Best weapon only drops on Nightmare" → Easy players can never compete. Keep gating cosmetic.
- **Not displaying the difficulty.** Players see varying enemy stats and feel the game is buggy. Always show difficulty on enemies (label, tint).
- **Forgetting to scale enemy AI parameters.** HP/damage scaled but `DetectRange` and `WalkSpeed` left at base → "Hard" feels identical to "Normal" gameplay-wise; just slower TTK. Scale AI alongside stats.
- **Hardcoded difficulty strings throughout.** Spelling drift ("Nightmare" vs "nightmare"). Use the typed `DifficultyTier` consistently and a constants module for the strings.

## See also
- [enemy-spawning.md](enemy-spawning.md) — applies difficulty per spawn via `applyDifficulty`
- [npcs.md](npcs.md) — enemy template; base stats stored as attributes for re-scaling
- [combat.md](combat.md) — reads `Damage` attribute from enemies; applies `XpMult` for kill XP
- [progression.md](progression.md) — XP scaling integration (level-difference + difficulty multipliers)
- [inventory.md](inventory.md) — loot drops scaled by `LootMult`
- [areas-and-biomes.md](areas-and-biomes.md) — per-area default difficulty
- [round-based.md](round-based.md) — wave-based endless difficulty progression
- [datastores.md](datastores.md) — persisting `chosenDifficulty` and `unlockedDifficulties`
- [menu-systems.md](menu-systems.md) — difficulty selection menu
- [security.md](security.md) — server-side difficulty validation; never trust client choice
- [live-ops.md](live-ops.md) — community-event difficulty modifiers ("Halloween +Difficulty week")
- [ui-system.md](ui-system.md), [tweens-animation.md](tweens-animation.md) — difficulty banners, tinted enemy nameplates
- [save-slots.md](save-slots.md) — chosen difficulty is per active slot
