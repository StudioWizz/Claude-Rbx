# Player Badges (Above-Head Indicators)

> Source: distilled from Roblox above-head display patterns (VIP crowns, prestige stars, group ranks, role badges)

## Purpose
Above-head **player badges** are visual indicators floating above a player's character — VIP crown, prestige stars, owner/admin/moderator role icons, gift-giver tag, top-spender, content-creator badge, level number. They communicate identity and status at a glance to other players. Implemented via `BillboardGui` parented to the head, with image/text labels stacked horizontally. This card covers the canonical badge stack, badge sources (VIP from gamepass, prestige from progression, gift-giver from trade history, etc.), per-badge visibility rules, and the cleanup-on-respawn discipline that catches everyone the first time.

## Anatomy

```
Character (Model)
├── Head (BasePart)
│   ├── BadgeGui (BillboardGui)
│   │   └── Container (Frame)
│   │       ├── UIListLayout (horizontal; sorts badges left-to-right)
│   │       ├── Badge_VIP (ImageLabel)         -- crown
│   │       ├── Badge_Prestige (ImageLabel)    -- star with number
│   │       ├── Badge_Level (TextLabel)        -- "Lv 52"
│   │       └── Badge_Gifter (ImageLabel)      -- ribbon
│   └── (default name display — handled by engine)
```

The BillboardGui floats above the head; its child Frame uses `UIListLayout` so badges line up automatically as they're added/removed.

## BadgeManager — server-side core

```lua
--!strict
local Players = game:GetService("Players")

export type BadgeDef = {
    id: string,
    iconId: string?,                -- "rbxassetid://..." for image badges
    text: string?,                  -- alternative: text-only badge ("Lv 52")
    color: Color3?,                 -- text color (text-only)
    bgColor: Color3?,               -- background tint (image)
    priority: number,               -- sort order (lower = leftmost)
    tooltip: string?,
}

local BadgeManager = {}

-- Per-player active badges
local activeBadges: {[Player]: {[string]: BadgeDef}} = {}

local BADGE_DEFS: {[string]: BadgeDef} = {
    Owner       = {id = "Owner", iconId = "rbxassetid://...", priority = 0,
                   bgColor = Color3.fromRGB(255, 215, 0), tooltip = "Game Owner"},
    Admin       = {id = "Admin", iconId = "rbxassetid://...", priority = 5,
                   bgColor = Color3.fromRGB(255, 100, 100), tooltip = "Admin"},
    Moderator   = {id = "Moderator", iconId = "rbxassetid://...", priority = 10,
                   bgColor = Color3.fromRGB(100, 200, 255)},
    VIP         = {id = "VIP", iconId = "rbxassetid://...", priority = 20,
                   bgColor = Color3.fromRGB(255, 200, 60), tooltip = "VIP Member"},
    Prestige    = {id = "Prestige", iconId = "rbxassetid://...", priority = 30,
                   text = "0", color = Color3.fromRGB(255, 255, 255)},      -- text overlaid on icon
    Level       = {id = "Level", text = "Lv 1", priority = 40,
                   color = Color3.fromRGB(255, 255, 255)},
    Gifter      = {id = "Gifter", iconId = "rbxassetid://...", priority = 50,
                   bgColor = Color3.fromRGB(180, 100, 220), tooltip = "Generous Gifter"},
    TopSpender  = {id = "TopSpender", iconId = "rbxassetid://...", priority = 60,
                   bgColor = Color3.fromRGB(120, 80, 180)},
    Creator     = {id = "Creator", iconId = "rbxassetid://...", priority = 5,
                   bgColor = Color3.fromRGB(255, 80, 200), tooltip = "Content Creator"},
}

function BadgeManager.Add(player: Player, badgeId: string, override: {[string]: any}?)
    local def = BADGE_DEFS[badgeId]; if not def then return end
    activeBadges[player] = activeBadges[player] or {}

    -- Allow per-player override (e.g., Prestige with "Prestige 12" text)
    local instance: BadgeDef = table.clone(def)
    if override then
        for k, v in override do instance[k] = v end
    end
    activeBadges[player][badgeId] = instance

    rebuildBadgeUi(player)
end

function BadgeManager.Remove(player: Player, badgeId: string)
    if not activeBadges[player] then return end
    activeBadges[player][badgeId] = nil
    rebuildBadgeUi(player)
end

function BadgeManager.SetText(player: Player, badgeId: string, text: string)
    if activeBadges[player] and activeBadges[player][badgeId] then
        activeBadges[player][badgeId].text = text
        rebuildBadgeUi(player)
    end
end

function BadgeManager.GetActive(player: Player): {[string]: BadgeDef}
    return activeBadges[player] or {}
end

return BadgeManager
```

## UI rebuild

`rebuildBadgeUi` destroys the existing BadgeGui (if any) and rebuilds from the active list. Cheap; only fires on add/remove/text-change, not per-frame.

```lua
local function rebuildBadgeUi(player: Player)
    local char = player.Character; if not char then return end
    local head = char:FindFirstChild("Head"); if not head then return end

    local existing = head:FindFirstChild("BadgeGui"); if existing then existing:Destroy() end

    local badges = activeBadges[player] or {}

    -- Sort by priority
    local sorted = {}
    for _, def in badges do table.insert(sorted, def) end
    table.sort(sorted, function(a, b) return a.priority < b.priority end)

    if #sorted == 0 then return end

    local gui = Instance.new("BillboardGui")
    gui.Name = "BadgeGui"
    gui.Adornee = head
    gui.Size = UDim2.fromOffset(220, 28)
    gui.StudsOffsetWorldSpace = Vector3.new(0, 2.5, 0)        -- above the default name display
    gui.AlwaysOnTop = true
    gui.LightInfluence = 0
    gui.MaxDistance = 100                                       -- hide beyond this distance for perf
    gui.Parent = head

    local container = Instance.new("Frame", gui)
    container.Size = UDim2.fromScale(1, 1)
    container.BackgroundTransparency = 1

    local list = Instance.new("UIListLayout", container)
    list.FillDirection = Enum.FillDirection.Horizontal
    list.HorizontalAlignment = Enum.HorizontalAlignment.Center
    list.VerticalAlignment = Enum.VerticalAlignment.Center
    list.Padding = UDim.new(0, 4)

    for i, def in sorted do
        local badge: GuiObject
        if def.iconId then
            badge = Instance.new("ImageLabel")
            badge.Image = def.iconId
            badge.Size = UDim2.fromOffset(24, 24)
            badge.BackgroundColor3 = def.bgColor or Color3.new(0, 0, 0)
            badge.BackgroundTransparency = def.bgColor and 0.2 or 1
            -- Text overlay (e.g., prestige number on top of star icon)
            if def.text then
                local overlay = Instance.new("TextLabel", badge)
                overlay.BackgroundTransparency = 1
                overlay.Size = UDim2.fromScale(1, 1)
                overlay.Text = def.text
                overlay.TextColor3 = def.color or Color3.new(1, 1, 1)
                overlay.Font = Enum.Font.BuilderSansBold
                overlay.TextSize = 12
            end
        else
            -- Text-only badge
            badge = Instance.new("TextLabel")
            badge.Size = UDim2.fromOffset(50, 24)
            badge.Text = def.text or "?"
            badge.Font = Enum.Font.BuilderSansBold
            badge.TextSize = 14
            badge.TextColor3 = def.color or Color3.new(1, 1, 1)
            badge.BackgroundColor3 = def.bgColor or Color3.fromRGB(0, 0, 0)
            badge.BackgroundTransparency = 0.4
        end

        Instance.new("UICorner", badge).CornerRadius = UDim.new(0, 4)
        badge.LayoutOrder = i
        badge.Parent = container
    end
end
```

The `MaxDistance = 100` matters — hundreds of players each with several badges visible everywhere kills performance. 100 studs is a comfortable peer-vision range; tune per-game.

## Cleanup on respawn

The BillboardGui lives under the character's head, so it's destroyed when the character respawns. Re-create on `CharacterAdded`:

```lua
Players.PlayerAdded:Connect(function(player)
    activeBadges[player] = activeBadges[player] or {}

    -- Re-add the badges that should be visible from the start
    syncBadgesFromProfile(player)

    player.CharacterAdded:Connect(function()
        task.wait(0.5)        -- let the character finish loading
        rebuildBadgeUi(player)
    end)
end)

Players.PlayerRemoving:Connect(function(player)
    activeBadges[player] = nil
end)
```

## Badge sources

Each badge has a different trigger / lookup. Wire them in centrally:

```lua
local function syncBadgesFromProfile(player: Player)
    -- Owner / Admin / Moderator (from allowlist; see live-ops.md)
    local role = Permissions.GetRole(player)
    if role == "owner" then BadgeManager.Add(player, "Owner") end
    if role == "admin" then BadgeManager.Add(player, "Admin") end
    if role == "moderator" then BadgeManager.Add(player, "Moderator") end

    -- VIP (from gamepass; see vip-systems.md)
    if VIPSystem.IsVip(player) then BadgeManager.Add(player, "VIP") end

    -- Prestige (from profile; see progression.md)
    local profile = PlayerData.Get(player)
    if profile then
        if profile.prestige and profile.prestige > 0 then
            BadgeManager.Add(player, "Prestige", {text = tostring(profile.prestige)})
        end
        BadgeManager.Add(player, "Level", {text = "Lv " .. profile.level})

        if profile.giftsSent and profile.giftsSent >= 50 then
            BadgeManager.Add(player, "Gifter")
        end
        if profile.totalRobuxSpent and profile.totalRobuxSpent >= 1000 then
            BadgeManager.Add(player, "TopSpender")
        end
    end

    -- Content creator (from a maintained list; could be in DataStore or hardcoded)
    if isContentCreator(player.UserId) then BadgeManager.Add(player, "Creator") end
end
```

When state changes mid-session, push updates:
- VIP purchased mid-session → `BadgeManager.Add(player, "VIP")`
- Prestige incremented → `BadgeManager.Add(player, "Prestige", {text = newPrestigeStr})`
- Level changed → `BadgeManager.SetText(player, "Level", "Lv " .. newLevel)`
- Player became admin via runtime promotion → `BadgeManager.Add(player, "Admin")`

## Badge visibility rules

Not every badge fits every context. Common rules:

### Per-player toggle ("Hide my badges")
Some players prefer a clean look. Profile setting:
```lua
profile.settings.hideOwnBadges = false        -- toggle in settings menu (see menu-systems.md)
```

In `rebuildBadgeUi`, skip the build if their setting says hide. (They still appear visible to *other* players — the toggle only affects whether this player wants their own visible.)

For "hide ALL players' badges from MY view," set the visibility client-side via a `LocalScript` that toggles the GUIs on/off:
```lua
-- LocalScript that listens to a player setting and toggles AllBadgeGuis
local function setAllBadgeVisibility(visible: boolean)
    for _, p in Players:GetPlayers() do
        local gui = p.Character and p.Character:FindFirstChild("Head"):FindFirstChild("BadgeGui")
        if gui then gui.Enabled = visible end
    end
end
```

### Context-specific hiding
- **In a stealth area** — hide all badges (gameplay reason; can't tell who's VIP)
- **In a competitive PvP arena** — hide level badge (avoid intimidation, encourage even matches)
- **During a cinematic / cutscene** — hide all UI including badges

Use a context flag and re-rebuild affected badges.

## Patterns

### Animated badge effects
A "newly-acquired VIP" badge shimmers/glows for the first 30 minutes after purchase to draw attention. Add a TweenService animation on the BadgeGui after creation:
```lua
local tween = TweenService:Create(badge, TweenInfo.new(1, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true),
                                  {BackgroundTransparency = 0.6})
tween:Play()
-- Stop after 30 minutes
task.delay(1800, function() tween:Cancel() end)
```

### Limited-edition / event badges
Halloween 2026 attendance badge, special occasion ceremony badge — same BadgeManager, just with `EventHalloween26` etc. and a profile flag tracking attendance.

### Stacking limits (don't show all 8 badges)
For players with many badges, show only the top-3 (by priority) and have a "..." hover for the rest. Or have a "Showcase" feature where the player picks which 3 to display.

### Tooltip on hover
For richer info ("VIP Member — purchased 2024-08-15"), use a separate ScreenGui that shows on hover via MouseEnter/Leave on the badge ImageLabel. Won't work reliably above-head (BillboardGui doesn't get easy hover), but you could do it on the player list (the right-side scoreboard).

### Prestige badge with stars
For prestige tier visualisation: 1 star for prestige 1, 2 stars for 2, ... up to 10. Past 10, show "10★+":
```lua
local stars = math.min(profile.prestige, 10)
local label = string.rep("★", stars) .. (profile.prestige > 10 and "+" or "")
BadgeManager.Add(player, "Prestige", {text = label})
```

### Badge-bound rewards
"Earn the Generous badge to unlock a free emote." Tie cosmetic unlocks to badge acquisition events. Couple with an achievement system if you have one.

### Player-list integration
The right-side scoreboard (default Roblox PlayerList) doesn't show badges. To show them, hide the default with `StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.PlayerList, false)` and build your own custom scoreboard ([menu-systems.md](menu-systems.md)) that includes badge rows.

## Pitfalls

- **BadgeGui re-built every Heartbeat.** Only rebuild on add/remove/text change. Per-frame builds destroy GuiObjects faster than you can blink and tank performance.
- **Badges visible at any distance.** Beyond ~100 studs, players can't read them. Set `MaxDistance` to cull.
- **Badges hidden on respawn forever.** The CharacterAdded re-add handler is required. Skip it and badges vanish after the player's first death.
- **Adding the same badge multiple times per frame** (race conditions when several systems each call Add). Use `if not activeBadges[player][badgeId] then` guard if it matters; the rebuild is idempotent but firing too often causes flicker.
- **Text on icon-only ImageLabel.** The text overlay needs to be a child TextLabel, not setting `.Text` on an ImageLabel (which has no Text property — it errors).
- **Stacking too many badges.** A player with 8 badges has a banner of icons that obscures the screen. Cap to top-3 or implement showcase.
- **Forgetting to remove badge on permission revocation.** Player demoted from admin → still shows Admin badge until next reload. Hook to permission-changed events.
- **Updating Level badge per XP point.** Triggers a UI rebuild every kill. Only rebuild on actual level change.
- **No `MaxDistance` on the BillboardGui.** Performance dies in 30-player servers. Set explicitly.
- **`BillboardGui.AlwaysOnTop = true` causing it to render through walls.** Sometimes desired (you want to know who's behind that pillar), sometimes not (in stealth games, badges revealing player positions through walls is bad). Choose deliberately per-game.
- **Badge size that doesn't scale with screen DPI.** On 4K monitors badges look tiny; on phones they look enormous. Use `BillboardGui.SizeOffset` and test at multiple resolutions.
- **Badge added before character has a Head.** Race condition on first spawn. The CharacterAdded handler with `task.wait(0.5)` covers most cases; for safety, `:WaitForChild("Head", 5)` in `rebuildBadgeUi`.
- **Badge `iconId` pointing to an asset the experience can't access.** Image fails to load, badge appears blank. Pre-validate at server start; warn if any badge def's iconId is bad.

## See also
- [characters.md](characters.md) — Head Instance the BillboardGui parents to; CharacterAdded for re-init
- [vip-systems.md](vip-systems.md) — sources the VIP badge
- [progression.md](progression.md) — sources the Prestige and Level badges
- [trading-and-gifting.md](trading-and-gifting.md) — sources the Gifter badge from `giftsSent` profile field
- [monetization.md](monetization.md) — could source TopSpender from total Robux spent
- [live-ops.md](live-ops.md) — Permissions module sourcing Owner/Admin/Moderator badges
- [ui-system.md](ui-system.md) — BillboardGui, ImageLabel, TextLabel primitives
- [menu-systems.md](menu-systems.md) — settings toggle for hiding badges; custom player list integration
- [tweens-animation.md](tweens-animation.md) — animated effects on newly-acquired badges
- [datastores.md](datastores.md) — persisting badge-source profile fields
- [save-slots.md](save-slots.md) — most badges are per-account (not per-slot); some (Prestige) are per-slot
