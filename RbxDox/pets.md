# Pets / Companions

> Source: distilled from pet/egg games (Adopt Me, Pet Simulator family) which form one of the largest Roblox genres

## Purpose
Pets are owned **non-equipment companions** that follow the player, contribute multipliers, can be hatched from eggs (RNG), inventoried, equipped (multiple slots), upgraded, and fused. The pattern is consistent across pet games: a **pet inventory** in the profile (separate from item inventory), an **equip system** (cap N active pets), a **follow behavior** (pet hovers around the player), **multiplier contributions** (each equipped pet boosts coins/luck/whatever), and **hatching** from eggs with a weighted random table. This card is the canonical assembly. For visual rendering of NPC-like pet models, see [npcs.md](npcs.md). For inventory and equipment patterns this builds on, see [inventory.md](inventory.md) and [equipment.md](equipment.md).

## Pet data

```lua
--!strict
export type PetDef = {
    id: string,
    name: string,
    rarity: "Common" | "Rare" | "Epic" | "Legendary" | "Mythic",
    modelId: string,                       -- "rbxassetid://..." or template name
    coinMultiplier: number,                -- 1.05 = +5% coins per equipped pet of this type
    xpMultiplier: number?,
    luckMultiplier: number?,
    iconId: string,
}

export type PetInstance = {
    uid: string,                           -- unique to this pet (HttpService:GenerateGUID)
    petId: string,                         -- references PetDef
    level: number,                         -- pet-level for upgrade systems
    nickname: string?,
    locked: boolean?,                      -- prevent accidental sale/fuse
}

local Pets = {}

Pets.Defs = {
    Cat = {id = "Cat", name = "Cat", rarity = "Common", modelId = "rbxassetid://...",
           coinMultiplier = 1.05, iconId = "rbxassetid://..."},
    Dragon = {id = "Dragon", name = "Dragon", rarity = "Legendary", modelId = "rbxassetid://...",
              coinMultiplier = 2.0, xpMultiplier = 1.5, iconId = "rbxassetid://..."},
}

return Pets
```

## Profile shape

```lua
profile.pets = {
    inventory = {} :: {[string]: PetInstance},   -- keyed by uid
    equipped = {} :: {string},                    -- list of uids; order = slot order
    maxEquipped = 3,                              -- cap; can grow with gamepass
}
```

Inventory holds every owned pet; `equipped` references which (up to `maxEquipped`) are active. The cap typically grows with gamepass purchases — see [monetization.md](monetization.md).

## Equip / unequip

```lua
local Pets = {}

function Pets.Equip(profile: PlayerProfile, uid: string): boolean
    if not profile.pets.inventory[uid] then return false end
    if table.find(profile.pets.equipped, uid) then return true end       -- already equipped
    if #profile.pets.equipped >= profile.pets.maxEquipped then return false end

    table.insert(profile.pets.equipped, uid)
    Progression.RecomputePetMultiplier(profile)                          -- update multiplier stack
    spawnFollowingPet(profile, uid)                                      -- spawn the visible pet
    return true
end

function Pets.Unequip(profile: PlayerProfile, uid: string)
    local idx = table.find(profile.pets.equipped, uid)
    if not idx then return end
    table.remove(profile.pets.equipped, idx)
    Progression.RecomputePetMultiplier(profile)
    despawnFollowingPet(profile, uid)
end

return Pets
```

Equipping doesn't grant the multiplier directly — it triggers a **recompute** of the player's pet multiplier (see [progression.md](progression.md) multiplier-stack discipline). This way you never duplicate or lose multiplier as pets are equipped/unequipped.

## Following behavior

Each equipped pet spawns as a small Model that hovers around the player. The simplest implementation: anchored part with periodic `:PivotTo` toward an offset from the player's HumanoidRootPart.

```lua
local function spawnFollowingPet(player: Player, uid: string, slot: number)
    local profile = PlayerData.Get(player); if not profile then return end
    local petInst = profile.pets.inventory[uid]; if not petInst then return end
    local def = Pets.Defs[petInst.petId]; if not def then return end

    local model = ReplicatedStorage.PetTemplates[def.id]:Clone()
    model.Name = "Pet_" .. uid
    for _, p in model:GetDescendants() do
        if p:IsA("BasePart") then
            p.Anchored = true
            p.CanCollide = false
            p.CanQuery = false
            p.Massless = true
        end
    end
    model.Parent = workspace.Pets

    -- Update follow position every Heartbeat
    local connection
    connection = RunService.Heartbeat:Connect(function(dt)
        if not model.Parent or not player.Character then connection:Disconnect(); return end
        local hrp = player.Character:FindFirstChild("HumanoidRootPart") :: BasePart?
        if not hrp then return end

        -- Compute pet's slot offset (left, right, behind, etc.)
        local angle = (slot - 1) * (math.pi * 2 / profile.pets.maxEquipped) + os.clock()    -- orbit
        local offset = Vector3.new(math.cos(angle) * 4, 1.5, math.sin(angle) * 4)
        local target = hrp.Position + offset

        -- Smooth lerp toward target
        local cur = model:GetPivot().Position
        model:PivotTo(CFrame.new(cur:Lerp(target, math.clamp(dt * 8, 0, 1))))
    end)
end
```

For hundreds of pets across hundreds of players, anchored-with-Heartbeat-pivots is much cheaper than physics-driven following.

For network performance, **render pets only on the owner's client + nearby clients**. Tag pet models with `OwnerUserId` and have each client keep only the visible-distance pets active.

## Hatching (egg → random pet)

```lua
type EggDef = {
    id: string,
    cost: number,                          -- in some currency
    pool: {{petId: string, weight: number}},
}

local EGGS = {
    BasicEgg = {
        id = "BasicEgg", cost = 100,
        pool = {
            {petId = "Cat", weight = 70},
            {petId = "Dog", weight = 25},
            {petId = "Dragon", weight = 5},
        },
    },
}

local function rollEgg(eggId: string): string?
    local egg = EGGS[eggId]; if not egg then return nil end
    local total = 0
    for _, p in egg.pool do total += p.weight end
    local roll = math.random() * total
    local accum = 0
    for _, p in egg.pool do
        accum += p.weight
        if roll <= accum then return p.petId end
    end
end

HatchRemote.OnServerEvent:Connect(function(player, eggId)
    local profile = PlayerData.Get(player); if not profile then return end
    local egg = EGGS[eggId]; if not egg then return end
    if profile.coins < egg.cost then return end

    profile.coins -= egg.cost

    local petId = rollEgg(eggId); if not petId then return end
    local newPet: PetInstance = {
        uid = HttpService:GenerateGUID(false):sub(1, 12),
        petId = petId,
        level = 1,
    }
    profile.pets.inventory[newPet.uid] = newPet

    HatchedRemote:FireClient(player, newPet)        -- show egg-cracking animation client-side
end)
```

The roll is **server-side**. A client-side roll lets exploiters re-roll until legendary.

## Hatching animation

The egg-cracking spectacle is client-side eye candy. After `HatchedRemote` fires, the client plays the crack-shake-reveal animation, lands on the pet's icon. The actual pet is already in the player's inventory before the animation starts; the show is decorative.

For drama, randomize the *displayed* roll path (cycle through icons fast, slow down, land on the real one) — the server already decided the result.

## Fusion / upgrade

Fusing N copies of the same pet into a higher-tier version, or upgrading via level:

```lua
function Pets.Fuse(profile: PlayerProfile, uids: {string}): string?
    if #uids < 3 then return nil end
    local petId = nil
    for _, uid in uids do
        local p = profile.pets.inventory[uid]
        if not p or p.locked then return nil end
        if not petId then petId = p.petId
        elseif petId ~= p.petId then return nil end          -- all must match
    end

    -- Burn the inputs
    for _, uid in uids do
        Pets.Unequip(profile, uid)
        profile.pets.inventory[uid] = nil
    end

    -- Spawn an upgraded version (custom logic per pet)
    local newPet: PetInstance = {
        uid = HttpService:GenerateGUID(false):sub(1, 12),
        petId = petId,
        level = 2,
    }
    profile.pets.inventory[newPet.uid] = newPet
    return newPet.uid
end
```

Common fusion rules: 3 pets of same type → +1 level; level 5 unlocks "shiny" variant; etc. Designer-tuned.

## Lock / favourite

To prevent accidental sale or fusion:
```lua
profile.pets.inventory[uid].locked = true
```
UI shows a lock icon; mutations check `.locked` and refuse.

## Patterns

### Pet UI: inventory + equip slots
- **Inventory grid** — every owned pet, sortable by rarity / multiplier.
- **Equip slots** (3, growing with gamepass) at the top, draggable from inventory.
- **Pet stats panel** — shows the focused pet's multipliers, level, fuse-progress.
- **Hatching screen** — egg picker, "open egg" button, hatch animation playback.

See [ui-system.md](ui-system.md) and [tweens-animation.md](tweens-animation.md).

### Pet rarity announcements
For epic+ hatches, broadcast game-wide via [live-ops.md](live-ops.md): "Adam just hatched a Mythic Dragon!" — same shape as the rare-drop pattern.

### Auto-equip best
A button that equips the player's top-N highest-multiplier pets:
```lua
local function autoEquip(profile: PlayerProfile)
    -- Sort inventory by coinMultiplier desc
    local sorted = {}
    for uid, pet in profile.pets.inventory do
        table.insert(sorted, {uid = uid, mult = (Pets.Defs[pet.petId] or {}).coinMultiplier or 1})
    end
    table.sort(sorted, function(a, b) return a.mult > b.mult end)

    -- Unequip everything
    for _, uid in table.clone(profile.pets.equipped) do Pets.Unequip(profile, uid) end
    -- Equip top N
    for i = 1, math.min(profile.pets.maxEquipped, #sorted) do
        Pets.Equip(profile, sorted[i].uid)
    end
end
```

### Pet trading between players
Same exploit-prone surface as item trading — both confirm via UI, server validates atomically. See [inventory.md](inventory.md) trading section.

## Pitfalls

- **Hatching rolled client-side.** Easy to exploit re-rolls. Always server-side.
- **Pet duplication via dual-server profile.** Player joins server A and server B simultaneously, hatches on each, both servers save → DataStore conflict resolves to one but the pet still got spawned on both. Use ProfileService session locking. See [datastores.md](datastores.md).
- **Multiplier stack overwrites.** `profile.multiplier = petMult` instead of contributing to a `pets` field in the multiplier-context table. See [progression.md](progression.md).
- **Following pets simulated by physics.** Hundreds of unanchored pets with constraints will tank performance. Anchored + Heartbeat-driven CFrame is much cheaper.
- **Pet visible to all players.** Network burn rendering 30 players' worth of pet models for everyone. Show only owner + nearby (filter on client by distance).
- **No `maxEquipped` cap.** Players equip all their pets and the multiplier reaches absurd values, breaking economy. Always cap.
- **Recompute pet multiplier missed on unequip.** Unequipping a pet leaves its multiplier in the stack. Always recompute on every equip/unequip and on pet-fusion (since old uids vanish).
- **Pet uid collision.** Generated GUID prefixes collide. Use longer prefix (12 chars) and/or check before insert.
- **Locked pet still fused.** Forgetting the `.locked` check destroys players' favorites. Validate every mutation against `.locked`.
- **Egg pool sums don't match expected probabilities.** A "5% legendary" egg actually rolls 4.7% because total weights are 105 not 100. Either normalize on load or always compute from total like the example above.
- **Saving the entire pets inventory every change.** Players with thousands of pets bloat the save. Save on event boundaries; consider per-pet versioning or chunking by rarity tier.
- **No way to delete duplicates.** A late-game player with 2,000 common cats can't navigate the UI. Add bulk-sell, mass-fuse, or auto-trash-low-rarity tools.

## See also
- [progression.md](progression.md) — multiplier-stack discipline; pet contribution
- [inventory.md](inventory.md) — pets are inventory items with extra structure; same persistence patterns
- [equipment.md](equipment.md) — pet equip slots are structurally similar to gear slots
- [npcs.md](npcs.md) — pet model anatomy is a stripped-down NPC (no Humanoid, no AI, just visuals)
- [characters.md](characters.md) — following uses HumanoidRootPart anchoring
- [datastores.md](datastores.md) — persisting `profile.pets`
- [monetization.md](monetization.md) — gamepass to extend `maxEquipped`
- [live-ops.md](live-ops.md) — rare-hatch game-wide announcements
- [save-slots.md](save-slots.md) — pets are per active slot
- [security.md](security.md) — server-side hatching, server-validated trades
- [tweens-animation.md](tweens-animation.md), [vfx.md](vfx.md), [sound.md](sound.md) — hatching spectacle
- [community-libraries.md](community-libraries.md) — ProfileService for session-locked pet inventory
