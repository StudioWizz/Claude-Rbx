# Placement System

> Source: distilled from tycoon, sandbox, and tower-defense placement patterns

## Purpose
A **placement system** lets players put objects into the world at runtime — buildings in a sandbox, droppers/walls in a tycoon, towers in a tower-defense game. The mechanic has the same shape across genres: **client renders a ghost preview that snaps and validates → player confirms → server validates again and instantiates the actual object → save the layout for persistence**. This card walks the canonical implementation, the grid-snap math, the collision/budget validation, and serialize/deserialize for persistent builds.

## The standard placement loop

```
Client                                       Server
  │                                            │
  │ 1. Player clicks "Build" in inventory      │
  │ 2. Spawn ghost (semi-transparent clone)    │
  │                                            │
  │ 3. Update ghost position every frame       │
  │    - raycast mouse → ground                │
  │    - snap to grid                          │
  │    - color green (valid) / red (invalid)   │
  │                                            │
  │ 4. Player clicks to confirm                │
  │── PlaceRequest(itemId, cf) ───────────────▶│
  │                                            │ 5. Validate: budget, collision, ownership, range
  │                                            │ 6. Instantiate real object server-side
  │                                            │ 7. Save into layout
  │ ◀─── PlacementConfirmed ───────────────────│
  │ 8. Destroy ghost                           │
```

Client gives feedback; server is authoritative. Never let the client place objects directly — they'll place anywhere, ignoring budgets and collisions.

## Grid snap

```lua
local GRID_SIZE = 4   -- 4 studs per cell

local function snapToGrid(pos: Vector3, cellSize: number?): Vector3
    cellSize = cellSize or GRID_SIZE
    return Vector3.new(
        math.round(pos.X / cellSize) * cellSize,
        pos.Y,                                       -- Y usually free (height)
        math.round(pos.Z / cellSize) * cellSize
    )
end
```

For free placement (no grid), skip this. For more sophisticated snap (rotational, edge-aligned), add per-itemId snap rules.

## Client — ghost preview

```lua
-- LocalScript under StarterPlayerScripts
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Players = game:GetService("Players")

local Placement = {}

local ghost: Model? = nil
local activeItemId: string? = nil
local validToPlace = false

local function clearHighlight() end    -- (omitted for brevity; see vfx.md / ui-system.md)
local function tintGhost(color: Color3)
    if not ghost then return end
    for _, p in ghost:GetDescendants() do
        if p:IsA("BasePart") then p.Color = color end
    end
end

local function makeGhost(itemId: string): Model
    local template = ReplicatedStorage.PlaceableTemplates[itemId]
    local copy = template:Clone()
    for _, p in copy:GetDescendants() do
        if p:IsA("BasePart") then
            p.Transparency = 0.6
            p.CanCollide = false
            p.CanQuery = false        -- don't block our raycast
            p.Anchored = true
        end
    end
    return copy
end

function Placement.StartBuilding(itemId: string)
    Placement.Stop()
    ghost = makeGhost(itemId)
    activeItemId = itemId
    ghost.Parent = workspace.Effects
end

function Placement.Stop()
    if ghost then ghost:Destroy(); ghost = nil end
    activeItemId = nil
end

local function getMouseGroundCFrame(): CFrame?
    local cam = workspace.CurrentCamera
    local mousePos = UIS:GetMouseLocation()
    local ray = cam:ViewportPointToRay(mousePos.X, mousePos.Y)

    local params = RaycastParams.new()
    params.FilterType = Enum.RaycastFilterType.Exclude
    params.FilterDescendantsInstances = {Players.LocalPlayer.Character, ghost}

    local result = workspace:Raycast(ray.Origin, ray.Direction * 200, params)
    if not result then return nil end
    return CFrame.new(snapToGrid(result.Position))
end

local function validatePlacement(cf: CFrame, itemId: string): boolean
    if not ghost then return false end
    -- Overlap check at the snapped position
    local op = OverlapParams.new()
    op.FilterType = Enum.RaycastFilterType.Exclude
    op.FilterDescendantsInstances = {Players.LocalPlayer.Character, ghost}

    local primary = ghost.PrimaryPart or ghost:FindFirstChildWhichIsA("BasePart")
    if not primary then return false end
    local hits = workspace:GetPartBoundsInBox(cf, primary.Size, op)

    -- Allow overlap with terrain / floor; reject overlap with other placed items
    for _, p in hits do
        if not p.Parent:IsDescendantOf(workspace.Terrain or workspace) then continue end
        if isPlacedItem(p) then return false end
    end
    return true
end

RunService.RenderStepped:Connect(function()
    if not ghost then return end
    local cf = getMouseGroundCFrame()
    if not cf then return end
    ghost:PivotTo(cf)
    validToPlace = validatePlacement(cf, activeItemId)
    tintGhost(if validToPlace then Color3.new(0, 1, 0) else Color3.new(1, 0, 0))
end)

UIS.InputBegan:Connect(function(input, gp)
    if gp or not ghost then return end
    if input.UserInputType == Enum.UserInputType.MouseButton1 then
        if validToPlace then
            local cf = ghost:GetPivot()
            PlaceRemote:FireServer(activeItemId, cf)
        end
    elseif input.KeyCode == Enum.KeyCode.R then
        ghost:PivotTo(ghost:GetPivot() * CFrame.Angles(0, math.rad(90), 0))   -- rotate
    elseif input.KeyCode == Enum.KeyCode.Escape then
        Placement.Stop()
    end
end)

return Placement
```

## Server — instantiate and validate

```lua
PlaceRemote.OnServerEvent:Connect(function(player, itemId, cf)
    if typeof(itemId) ~= "string" or typeof(cf) ~= "CFrame" then return end
    local def = Placeables[itemId]; if not def then return end

    local profile = PlayerData.Get(player); if not profile then return end

    -- Inventory / cost check
    if not Inventory.Has(profile.inventory, itemId, 1) then return end

    -- Budget check (e.g., max 50 placed items per player)
    if countPlayerPlacements(player) >= MAX_PER_PLAYER then return end

    -- Collision check (server-side; mirror client logic)
    if not validatePlacementServer(cf, def) then return end

    -- Distance check (player must be near the placement)
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if not hrp or (hrp.Position - cf.Position).Magnitude > MAX_PLACE_RANGE then return end

    -- Instantiate
    local model = ReplicatedStorage.PlaceableTemplates[itemId]:Clone()
    model:PivotTo(cf)
    model:SetAttribute("OwnerUserId", player.UserId)
    model:SetAttribute("ItemId", itemId)
    model.Parent = workspace.Placements
    CollectionService:AddTag(model, "Placed")

    -- Consume from inventory
    Inventory.Remove(profile.inventory, itemId, 1)

    -- Append to saved layout
    table.insert(profile.layout, {itemId = itemId, cf = cfToTable(cf)})

    PlacementConfirmed:FireClient(player, itemId)
end)
```

The two validations (client and server) protect against different attacks: client validation gives *feedback* for the player's intent; server validation enforces the *truth*. Don't skip either.

## Removing / destroying placed objects

```lua
RemoveRemote.OnServerEvent:Connect(function(player, model)
    if not model:IsA("Instance") or not model:IsDescendantOf(workspace.Placements) then return end
    if model:GetAttribute("OwnerUserId") ~= player.UserId then return end

    local itemId = model:GetAttribute("ItemId")
    local profile = PlayerData.Get(player); if not profile then return end

    -- Refund (or not — designer choice)
    Inventory.Add(profile.inventory, itemId, 1)

    -- Remove from saved layout
    for i, entry in profile.layout do
        if entry.itemId == itemId and cframesEqual(entry.cf, model:GetPivot()) then
            table.remove(profile.layout, i); break
        end
    end

    model:Destroy()
end)
```

## Save / load layouts (serialization)

CFrames don't serialize to JSON natively. Convert to/from a table shape:

```lua
local function cfToTable(cf: CFrame): {[number]: number}
    return {cf:GetComponents()}    -- 12 numbers: pos + 9 rotation matrix entries
end

local function tableToCf(t: {[number]: number}): CFrame
    return CFrame.new(table.unpack(t))
end
```

Save layout as:
```lua
profile.layout = {
    {itemId = "Wall", cf = {0, 5, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1}},
    {itemId = "Floor", cf = {0, 0, 0, 1, 0, 0, 0, 1, 0, 0, 0, 1}},
    -- ...
}
```

On player join, replay the layout to rebuild their build:

```lua
local function loadLayout(player: Player, profile: PlayerProfile)
    for _, entry in profile.layout do
        local def = Placeables[entry.itemId]; if not def then continue end
        local model = ReplicatedStorage.PlaceableTemplates[entry.itemId]:Clone()
        model:PivotTo(tableToCf(entry.cf))
        model:SetAttribute("OwnerUserId", player.UserId)
        model:SetAttribute("ItemId", entry.itemId)
        model.Parent = workspace.Placements
    end
end
```

For very large builds (hundreds of objects), batch-instantiate via the build-then-parent pattern from [workspace-hierarchy.md](workspace-hierarchy.md): assemble in a detached folder, then parent once.

## Sharing builds between players

To let a player share their layout (e.g., post a "build code"):
1. Serialize `profile.layout` to JSON (`HttpService:JSONEncode`).
2. Compress + encode (base64) for portability.
3. Store in DataStore under a global "BuildCode_<id>" key, return the id to the player.
4. Other players load by entering the id; server reads, decodes, applies via the same `loadLayout` path.

Add validation: cap deserialized layout size (don't trust an attacker uploading a 10MB layout).

## Patterns

### Tycoon dropper / collector
A "dropper" placeable spawns currency-instances on a timer; a "collector" tagged Touched detector adds them to the owner's currency. Standard composition:
```lua
-- Each Dropper Model has Attributes: DropRate, DropValue
-- Each Collector tagged "Collector"; attribute OwnerUserId set on placement
```
The placement system handles the placement; the dropper/collector logic lives in their own scripts that key off the OwnerUserId attribute.

### Buy-button progression (tycoon-style)
Place buy-buttons (`ProximityPrompt`-driven) that, on Triggered, charge currency and place a new tycoon section. See [interactions.md](interactions.md) and [shops.md](shops.md).

### Tower defense — place towers, attack waves
Towers placed via this system; their AI (target acquisition, firing) is in their own scripts. The placement system is just "buy a tower → spawn it on the tile → save which tiles are occupied so you can't double-place."

### Free-form (no grid)
Skip `snapToGrid`; let the ghost follow the mouse exactly. Combine with a **support check** (raycast down from ghost, ensure something solid beneath) for sandbox-style building.

## Pitfalls

- **No server-side validation.** Players place 100 walls inside the map's collision geometry, on top of NPCs, or 1000 studs in the air. Always re-validate server-side.
- **Distance check forgotten.** Players place from across the map. Cap the placement range from the player's character.
- **Budget not enforced.** Players place infinite items. Cap per-player counts.
- **Ghost stays alive after placing.** Forgot to destroy on confirm. Build a `Stop()` that you call after every confirm and on inventory change.
- **Ghost intercepts the raycast.** Forgot `CanQuery = false` on the ghost. The mouse-to-ground raycast hits the ghost itself, glitches the position.
- **CFrame serialization saves only position.** A wall placed rotated 90° loads back upright. Use `:GetComponents()` (12 numbers) not just position.
- **Saved layout exceeds DataStore byte limit.** A 10000-item layout doesn't fit in 4MB. Cap layout size, use compression, or chunk across multiple keys.
- **Layout replays a part you've since removed from PlaceableTemplates.** Skip unknown itemIds with a warn; don't crash.
- **Collision check too strict.** A wall snaps into "valid" but the floor it's placed on counts as overlap → never green. Filter the floor out of the overlap check (or check only against placed-items via the `"Placed"` tag).
- **Rotation key bound to a key the player uses for movement.** R is conventional; verify it's not bound to something else by ContextActionService.
- **Client-grid and server-grid differ.** If the client snaps to a 4-stud grid but the server validates against a 5-stud grid, every placement fails. Single source of truth.
- **Preview drift on lag.** The ghost lags behind the mouse on slow machines (RenderStepped runs less often). Use `RenderStepped` (renders every frame) not `Heartbeat` for ghost updates.
- **Network ownership of placed dynamic objects.** If placed objects are unanchored (e.g., physics-driven), they get assigned to the placing player's network. Consider `:SetNetworkOwner(nil)` for game-critical placements.

## See also
- [procedural-construction.md](procedural-construction.md) — Claude/script-driven construction (vs the player-driven placement covered here)
- [raycasting.md](raycasting.md) — mouse-to-ground raycast
- [parts-and-models.md](parts-and-models.md) — `:PivotTo`, BasePart properties for the ghost
- [workspace-hierarchy.md](workspace-hierarchy.md) — Folder organization, build-then-parent
- [collisions.md](collisions.md) — overlap queries for collision check
- [datastores.md](datastores.md) — persisting `profile.layout`
- [inventory.md](inventory.md) — consume the placeable from inventory on place; refund on remove
- [security.md](security.md) — server-side validation discipline
- [tags.md](tags.md), [attributes.md](attributes.md) — tag placed items, store ownership and itemId
- [interactions.md](interactions.md) — ProximityPrompt-driven place / remove
- [community-libraries.md](community-libraries.md) — `PlacementService` (third-party) for sandbox-tycoon games
