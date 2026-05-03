# Avatar Customization, Clothing & Emotes

> Upstream: <https://create.roblox.com/docs/characters/appearance> · <https://create.roblox.com/docs/avatar> · <https://create.roblox.com/docs/reference/engine/classes/HumanoidDescription>
> Source: distilled from HumanoidDescription, Accessory, and Humanoid emote APIs

## Purpose
A Roblox avatar's visual presentation — body parts, scaling, accessories (hats, hair, faces, gear), clothing (shirts, pants, layered items), colors, and emotes — is encapsulated in a single `HumanoidDescription` Instance. You can read a player's current avatar via `Humanoid:GetAppliedDescription()`, build a new one, or fetch any user's published avatar via `Players:GetHumanoidDescriptionFromUserId(userId)` and apply it. This card covers the runtime API; the Studio Avatar tab handles the *editing* workflow.

## HumanoidDescription — the unified bundle

A `HumanoidDescription` is a single Instance describing every visual aspect of an avatar:

```lua
-- Body part asset IDs (R15)
desc.Head, Torso, LeftArm, RightArm, LeftLeg, RightLeg
desc.LeftFoot, RightFoot, LeftHand, RightHand, LeftLowerArm, RightLowerArm
-- ... (15 parts for R15)

-- Scaling
desc.HeightScale       -- default 1.0
desc.WidthScale
desc.HeadScale
desc.DepthScale
desc.BodyTypeScale
desc.ProportionScale

-- Colors (BrickColor or Color3 — both supported)
desc.HeadColor : Color3
desc.TorsoColor, LeftArmColor, RightArmColor, LeftLegColor, RightLegColor

-- Clothing assets
desc.Shirt, Pants, GraphicTShirt        -- asset IDs (number)
desc.Face                                -- decal asset id

-- Accessories — comma-separated string of asset IDs (legacy) or array (modern)
desc.HairAccessory       -- "12345,67890"
desc.FaceAccessory
desc.NeckAccessory
desc.ShouldersAccessory
desc.FrontAccessory, BackAccessory, WaistAccessory
desc.HatAccessory

-- Modern API for accessories with full data (preferred):
desc:GetAccessories(includeRigidAccessories: bool): {AccessoryDescription}
desc:SetAccessories(accessories: {AccessoryDescription}, includeRigidAccessories: bool)
-- AccessoryDescription includes AssetId, AccessoryType, IsLayered, Order, Puffiness

-- Equipped emotes
desc:GetEmotes(): {[string]: {number}}      -- {Name = {assetId, ...}}
desc:SetEmotes(emotes: {[string]: {number}})

-- Equipped emote slot (the 8 quick-slots in the emote menu)
desc:GetEquippedEmotes(): {{Slot: number, Name: string}}
desc:SetEquippedEmotes(equipped: {{Slot: number, Name: string}})
```

## Reading and applying

### Get a player's current avatar
```lua
local desc = humanoid:GetAppliedDescription()      -- returns a copy you can mutate
desc.HeadColor = Color3.new(1, 0, 0)
humanoid:ApplyDescription(desc)                    -- yields; rebuilds the character visuals
```

`ApplyDescription` yields and is server-authoritative. The character is rebuilt with the new visuals — a brief flicker is normal.

### Get any player's published avatar
```lua
local Players = game:GetService("Players")
local ok, desc = pcall(function()
    return Players:GetHumanoidDescriptionFromUserId(123456789)
end)
if ok then humanoid:ApplyDescription(desc) end
```

Useful for "dress this NPC like a specific player," outfit galleries, leaderboards with avatar previews.

### Get an outfit by ID
Players publish outfits via the Avatar Editor. Each has an outfit ID:
```lua
local desc = Players:GetHumanoidDescriptionFromOutfitId(outfitId)
humanoid:ApplyDescription(desc)
```

## Accessories — the `Accessory` class

An `Accessory` is a `Model` (class `Accessory`) containing a `Handle` BasePart and `Attachment`s that snap to matching attachment names on the character.

```lua
-- Attach an accessory at runtime
local hat = ServerStorage.Hats.TopHat:Clone()    -- a pre-authored Accessory
humanoid:AddAccessory(hat)                        -- engine welds via attachment matching

-- Remove all accessories
humanoid:RemoveAccessories()

-- Iterate
for _, acc in humanoid:GetAccessories() do
    print(acc.Name, acc.AccessoryType)
end
```

`AccessoryType` enum: `Hat`, `Hair`, `Face`, `Neck`, `Shoulder`, `Front`, `Back`, `Waist`, `TShirt`, `Shirt`, `Pants`, `Jacket`, `Sweater`, `Shorts`, `LeftShoe`, `RightShoe`, `DressSkirt`, `Eyebrow`, `Eyelash`.

### Layered clothing

Layered clothing wraps the body and stacks (shirt over t-shirt over body, etc.). On a `WrapLayer` descendant of the Accessory, set:
```lua
wrapLayer.Order = 1               -- stacking order
wrapLayer.Puffiness = 0.2         -- distance from body (0 = tight, 1 = loose)
```

Most layered clothing is configured at upload time in the Avatar Setup workflow. Code-side, `:AddAccessory` handles the rest.

## Clothing — Shirt, Pants, T-Shirt (legacy 2D)

Older clothing system, still common: 2D textures wrapped onto the Torso.

```lua
local shirt = Instance.new("Shirt", character)
shirt.ShirtTemplate = "rbxassetid://12345"

local pants = Instance.new("Pants", character)
pants.PantsTemplate = "rbxassetid://67890"

-- T-Shirt is a Decal on the Torso
```

Setting these via HumanoidDescription is preferred; direct Instance.new works too.

## Body colors

```lua
local bodyColors = character:FindFirstChildOfClass("BodyColors")
bodyColors.HeadColor3 = Color3.fromRGB(255, 200, 150)
bodyColors.TorsoColor3 = Color3.new(0, 0, 1)
-- etc. for LeftArm, RightArm, LeftLeg, RightLeg
```

Each part's `Color3` field. The legacy `HeadColor` (without 3) takes a `BrickColor` — prefer `Color3`.

## Emotes

### Built-in emotes
Players have access to a default emote menu (/e dance, /e wave, etc.). Open via the in-game emote button or chat.

### Playing an emote programmatically
```lua
local success, animTrack = humanoid:PlayEmote("dance1")    -- emote name from the avatar's emote list
-- success: bool — false if the emote isn't equipped on this avatar
-- animTrack: AnimationTrack you can connect Stopped to
```

Or by asset ID directly (must be an emote asset):
```lua
humanoid:PlayEmote(7553967391)    -- numeric assetId
```

Any animation can be played as an emote IF its `AnimationPriority` is `Action` or higher (otherwise movement overrides it). Players standing still play the emote until they move.

### Emote events
```lua
humanoid.EmoteLoaded:Connect(function(emoteName, success)
    -- fires after the engine attempts to play
end)
```

### Custom emotes for your experience
```lua
-- Equip a custom emote on the character's HumanoidDescription
local desc = humanoid:GetAppliedDescription()
local emotes = desc:GetEmotes()
emotes["VictoryDance"] = {7777777}     -- {assetId, ...}
desc:SetEmotes(emotes)

-- Equip in a quick-slot (1-8)
local equipped = desc:GetEquippedEmotes()
table.insert(equipped, {Slot = 1, Name = "VictoryDance"})
desc:SetEquippedEmotes(equipped)

humanoid:ApplyDescription(desc)
```

After this, the player can use `/e VictoryDance` or click slot 1 in the emote menu.

## Patterns

### Outfit picker
Save the player's preferred `HumanoidDescription` as JSON in DataStore. On respawn:
```lua
player.CharacterAdded:Connect(function(char)
    local hum = char:WaitForChild("Humanoid")
    local saved = loadOutfit(player)             -- table
    if saved then
        local desc = Instance.new("HumanoidDescription")
        for k, v in saved do desc[k] = v end
        hum:ApplyDescription(desc)
    end
end)
```

### Costume on a button-press
```lua
ProximityPrompt.Triggered:Connect(function(player)
    local hum = player.Character and player.Character:FindFirstChildOfClass("Humanoid")
    if not hum then return end

    local desc = hum:GetAppliedDescription()
    desc.Shirt = COSTUME_SHIRT_ID
    desc.Pants = COSTUME_PANTS_ID
    desc.HatAccessory = tostring(WIZARD_HAT_ID)
    hum:ApplyDescription(desc)
end)
```

### Dress an NPC like a player
```lua
local desc = Players:GetHumanoidDescriptionFromUserId(player.UserId)
npcHumanoid:ApplyDescription(desc)
```

### Force a specific avatar (no customization)
In **Game Settings → Avatar → Avatar Type**, lock to your custom HumanoidDescription. All players spawn with that avatar regardless of their actual one. Useful for genre experiences (RPG with class-locked appearances).

## Pitfalls

- **`ApplyDescription` yields.** It rebuilds the character. Don't call it every frame; call on equip-change events.
- **Asset IDs must be valid for your experience's permission scope.** A private accessory only the developer owns won't load for other players unless re-published.
- **Accessories not snapping correctly** → check the Accessory's `Handle` Attachment names match the character rig's attachment names (`HatAttachment`, `HairAttachment`, etc.).
- **Clothing on R6 vs R15 differ** — Shirt/Pants templates target one rig type; the wrong template renders distorted.
- **Layered clothing requires `Lighting.Technology = Future`** (or ShadowMap) — Voxel doesn't render the wrap correctly. See [lighting.md](lighting.md).
- **Setting `BodyColors` directly after `ApplyDescription`** is overwritten by the description's color fields. Prefer setting colors via the description.
- **Modifying the description returned by `GetAppliedDescription`** mutates a copy, not the live state. You must call `ApplyDescription` to apply the changes.
- **Emote not equipped** → `PlayEmote(name)` returns false. Check `desc:GetEmotes()` first; equip via `SetEmotes`/`SetEquippedEmotes` if missing.
- **Expecting emote animations to override combat** — emotes have `Action` priority; if your combat animations are `Action4`, they win. Set priorities deliberately.
- **`GetHumanoidDescriptionFromUserId` for a banned/deleted user** errors. Always pcall.
- **Avatar Type "Player Choice" overrides your description on respawn** — set the Avatar Type to a custom mode in Game Settings if you need persistent overrides.

## See also
- [characters.md](characters.md) — Humanoid, Animator, basic appearance
- [datastores.md](datastores.md) — saving outfit selections per player
- [interactions.md](interactions.md) — ProximityPrompt for outfit-changing stations
- [tweens-animation.md](tweens-animation.md) — AnimationTrack for emote playback
- [lighting.md](lighting.md) — `Future` rendering required for layered clothing
- [community-libraries.md](community-libraries.md) — ProfileService for outfit persistence
