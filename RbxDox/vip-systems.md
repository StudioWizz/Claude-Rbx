# VIP Systems

> Source: distilled from VIP / premium / season-pass patterns across Roblox monetization-driven games

## Purpose
"VIP" is a coherent feature pattern almost always **backed by a Game Pass** (one-time purchase, permanent ownership): players who own the pass get a bundle of perks — exclusive shop items, exclusive areas, discounts, daily login rewards, visual identifier (crown above head, gold name color), faster progression. This card is the canonical assembly: how to gate features by ownership, how to structure the perk catalog, how to keep VIP detection cheap (ownership cache) and authoritative (server-side checks). Builds on `monetization.md` (the gamepass primitives), `shops.md` (VIP shop integration), `interactions.md` (VIP-only doors), `collisions.md` selective barriers (VIP-only physical areas), `progression.md` (VIP multipliers), and `player-badges.md` (above-head VIP crown).

## VIP detection — the foundation

VIP status = ownership of one or more game passes. Cache aggressively:

```lua
local MarketplaceService = game:GetService("MarketplaceService")
local Players = game:GetService("Players")

local VIP_PASS_ID = 12345678        -- replace with your real gamepass id

local vipStatus: {[Player]: boolean} = {}        -- per-player cache

local function checkVip(player: Player): boolean
    if vipStatus[player] ~= nil then return vipStatus[player] end

    local owns = false
    pcall(function()
        owns = MarketplaceService:UserOwnsGamePassAsync(player.UserId, VIP_PASS_ID)
    end)
    vipStatus[player] = owns
    return owns
end

Players.PlayerAdded:Connect(function(player)
    -- Pre-cache on join
    checkVip(player)
end)

Players.PlayerRemoving:Connect(function(player)
    vipStatus[player] = nil
end)

-- Refresh on mid-session purchase
MarketplaceService.PromptGamePassPurchaseFinished:Connect(function(player, gamePassId, wasPurchased)
    if wasPurchased and gamePassId == VIP_PASS_ID then
        vipStatus[player] = true
        applyVipBenefits(player)
        notifyPlayer(player, "Welcome to VIP!")
    end
end)
```

Use `checkVip(player)` everywhere VIP gating is needed — it's effectively free after the first check.

## Apply benefits at session start

Once detected, push VIP-applied state to the player:

```lua
local function applyVipBenefits(player: Player)
    if not checkVip(player) then return end

    -- Set VIP flag on player attribute (replicates to client; usable in client UI)
    player:SetAttribute("IsVip", true)

    -- Visual: above-head crown (see player-badges.md)
    PlayerBadges.Add(player, "VIP")

    -- Progression: increment multiplier stack (see progression.md)
    local profile = PlayerData.Get(player)
    if profile then
        profile.multiplierContext.gamepass = 2.0       -- 2x XP/coins for VIP
    end

    -- Inventory: grant any one-time VIP welcome items (with idempotency)
    if profile and not profile.vipWelcomeGranted then
        Inventory.Add(profile.inventory, "VipWelcomeChest", 1)
        profile.vipWelcomeGranted = true
    end

    -- Inform client UI to show VIP-only buttons
    VipStatusRemote:FireClient(player, true)
end

local function removeVipBenefits(player: Player)
    -- Called on refund / Pass revocation (rare but possible)
    player:SetAttribute("IsVip", false)
    PlayerBadges.Remove(player, "VIP")
    local profile = PlayerData.Get(player)
    if profile then profile.multiplierContext.gamepass = 1.0 end
    VipStatusRemote:FireClient(player, false)
end

Players.PlayerAdded:Connect(function(player)
    player.CharacterAdded:Connect(function() applyVipBenefits(player) end)
    if player.Character then applyVipBenefits(player) end
end)
```

## VIP shop — exclusive items

Add a `vipOnly = true` field to shop items in the catalog ([shops.md](shops.md)):

```lua
Shop.VIP = {
    {id = "VipSword", name = "VIP Sword", price = 5000, currency = "Coins", vipOnly = true},
    {id = "VipPet", name = "Crowned Pet", price = 25000, currency = "Coins", vipOnly = true},
}

-- In the buy handler:
if item.vipOnly and not checkVip(player) then
    return {ok = false, err = "vip-only"}
end
```

Render the VIP shop as either a separate tab (only visible to VIPs) OR mixed in the regular shop with locked/greyed VIP entries that prompt the gamepass purchase when clicked.

## VIP-only areas

### Soft gate (interaction-based)
A door / portal that checks VIP on Triggered:

```lua
vipPortal.Triggered:Connect(function(player)
    if not checkVip(player) then
        notifyPlayer(player, "VIP only — purchase the VIP pass to enter")
        MarketplaceService:PromptGamePassPurchase(player, VIP_PASS_ID)
        return
    end
    teleportToVipArea(player)
end)
```

### Hard gate (physical barrier)
For an invisible wall that lets VIPs pass but blocks everyone else, use the **selective-barriers pattern** from [collisions.md](collisions.md):

```lua
-- One-time setup at server start
PhysicsService:RegisterCollisionGroup("VipBarrier")
PhysicsService:RegisterCollisionGroup("Players")
PhysicsService:RegisterCollisionGroup("VipPlayers")

PhysicsService:CollisionGroupSetCollidable("Players", "VipBarrier", true)
PhysicsService:CollisionGroupSetCollidable("VipPlayers", "VipBarrier", false)

-- Apply on character spawn
local function assignPlayerCollisionGroup(player, character)
    local group = checkVip(player) and "VipPlayers" or "Players"
    for _, p in character:GetDescendants() do
        if p:IsA("BasePart") then p.CollisionGroup = group end
    end
end
```

The wall blocks non-VIPs physically; VIPs walk through.

For the "best of both" — visible to non-VIPs (so they know what they're missing), invisible/intangible to VIPs — keep the Part visible but use the collision group + a per-player Touched handler that shows a "VIP only" prompt instead of teleporting.

## VIP discount

Apply a percentage discount in the shop's purchase flow:

```lua
function getEffectivePrice(player: Player, item: ShopItem): number
    if checkVip(player) and item.currency == "Coins" then
        return math.floor(item.price * 0.85)        -- 15% VIP discount on coin purchases
    end
    return item.price
end

-- In buy handler:
local price = getEffectivePrice(player, item)
if profile.coins < price then return {ok = false, err = "insufficient"} end
profile.coins -= price
```

Apply only to in-game-currency purchases — not Robux. (Robux discounts would require dynamic devproduct prices, which Roblox doesn't support easily.)

Display in the UI: "1000 coins ~~1200~~" with the original struck through and the VIP price highlighted.

## VIP daily / weekly rewards

Recurring rewards for VIPs only:

```lua
local DAILY_VIP_REWARD = {coins = 500, gems = 5}

local function tryGrantDailyVip(player: Player)
    if not checkVip(player) then return end
    local profile = PlayerData.Get(player); if not profile then return end

    local today = os.date("!*t", os.time()).yday
    if profile.lastVipDailyYday == today then return end       -- already claimed today

    profile.coins += DAILY_VIP_REWARD.coins
    profile.gems += DAILY_VIP_REWARD.gems
    profile.lastVipDailyYday = today

    -- Streak counter (for "+50% bonus on day 7+")
    profile.vipDailyStreak = (profile.vipDailyStreak or 0) + 1

    notifyPlayer(player, string.format("VIP daily reward: +%d coins, +%d gems", DAILY_VIP_REWARD.coins, DAILY_VIP_REWARD.gems))
end

Players.PlayerAdded:Connect(function(p)
    p.CharacterAdded:Connect(function()
        task.wait(2)        -- give time for profile to load
        tryGrantDailyVip(p)
    end)
end)
```

For a "VIP daily chest" that the player must actively claim (not auto-granted), add a UI button in the menu that fires `ClaimVipDailyRemote:FireServer()`. Server checks day boundary and grants.

## VIP achievement / cosmetic gating

Some content unlocks for VIPs only (titles, hats, emotes):

```lua
local function isContentUnlocked(player: Player, contentId: string): boolean
    if VIP_ONLY_CONTENT[contentId] and not checkVip(player) then return false end
    return true       -- normal unlock rules
end
```

For "exclusive emote VIPs get": grant the emote to their `HumanoidDescription` on `applyVipBenefits` ([avatar-customization.md](avatar-customization.md)).

## Multi-tier VIP (Bronze / Silver / Gold)

For tiered systems, use multiple gamepasses:

```lua
local TIERS = {
    Bronze = {id = 11111111, multiplier = 1.5, dailyCoins = 200, badge = "VIP_Bronze"},
    Silver = {id = 22222222, multiplier = 2.0, dailyCoins = 500, badge = "VIP_Silver"},
    Gold   = {id = 33333333, multiplier = 3.0, dailyCoins = 1000, badge = "VIP_Gold"},
}

local function getHighestTier(player: Player): string?
    -- Check from highest down
    if checkPassOwnership(player, TIERS.Gold.id) then return "Gold"
    elseif checkPassOwnership(player, TIERS.Silver.id) then return "Silver"
    elseif checkPassOwnership(player, TIERS.Bronze.id) then return "Bronze" end
    return nil
end
```

Apply only the highest tier's benefits (Gold supersedes Silver supersedes Bronze). UX: show all three tiers in a "VIP tiers" menu; current tier highlighted; upgrade-to-higher prompts to purchase.

## Premium membership integration (separate from custom VIP)

Roblox's own Premium membership is separate from your custom VIP Game Pass. Reward Premium too if you want — see [monetization.md](monetization.md):

```lua
local function checkPremium(player: Player): boolean
    return player.MembershipType == Enum.MembershipType.Premium
end

Players.PlayerMembershipChanged:Connect(function(player)
    if checkPremium(player) then grantPremiumPerks(player) end
end)
```

Some games stack Premium + custom VIP for compounding benefits; others treat Premium as equivalent to a low VIP tier. Designer's call.

## Patterns

### VIP grace period for refunds
If a player refunds the gamepass, `UserOwnsGamePassAsync` returns false on next check. To detect: re-check periodically in long sessions and `removeVipBenefits` if status changes:
```lua
task.spawn(function()
    while true do
        task.wait(300)        -- every 5 minutes
        for _, p in Players:GetPlayers() do
            local cached = vipStatus[p]
            vipStatus[p] = nil       -- force re-fetch
            local current = checkVip(p)
            if cached and not current then removeVipBenefits(p) end
        end
    end
end)
```

For most games, accept the loose-end (Roblox refunds are rare) and skip this.

### VIP visual flair
- **Trail behind player** (Beam between two attachments — see [vfx.md](vfx.md))
- **Particle effect at feet** (gold sparkles)
- **Custom name color** (`Player.NameDisplayDistance` + custom BillboardGui — see [player-badges.md](player-badges.md))
- **Crown above head** (BillboardGui ImageLabel)

Apply visuals on `applyVipBenefits`; clean up on `removeVipBenefits`.

### VIP-exclusive events
"VIP-only Halloween event" — gate via the same `checkVip` in event-trigger code. Quest givers / portals only respond to VIPs.

### VIP-exclusive matchmaking lobby
Use [memorystores.md](memorystores.md) Queue with a separate VIP queue id; VIPs face shorter queues (more resources, less competition).

### "Try VIP" temporary trial
Grant 24-hour VIP after a player completes some action (first purchase of any kind, refer-a-friend, daily-login-streak-of-7). Track `vipTrialUntil = os.time() + 86400` in the profile; `checkVip` checks the trial timestamp first, then ownership.

```lua
local function checkVip(player: Player): boolean
    local profile = PlayerData.Get(player)
    if profile and profile.vipTrialUntil and os.time() < profile.vipTrialUntil then return true end
    -- Then fall through to gamepass check (cached as before)
end
```

After the trial ends, prompt to purchase: "Enjoyed VIP? Get the permanent pass!" with a discount or limited-time offer. Conversion-driving pattern.

### VIP affinity / faction
"You can be a member of the VIP Knights" — joining the VIP faction grants additional perks beyond the base VIP pass. Track faction membership in the profile.

## Pitfalls

- **`UserOwnsGamePassAsync` called per frame.** Yields and rate-limits. Cache in `vipStatus` and refresh only on PlayerAdded + PromptGamePassPurchaseFinished + manual periodic re-check.
- **Trusting client claim of VIP status.** A `IsVipRemote` from client is forgeable. All gates check server-side `checkVip(player)`.
- **Pass refund not detected.** Player buys VIP, refunds, still has perks until session ends or you re-check. Either accept the loophole or implement periodic refresh.
- **VIP perks inflated cumulatively.** Two systems both apply VIP multiplier → 4× instead of 2×. Track in the multiplier-stack `gamepass` slot (see [progression.md](progression.md)) — single source of truth.
- **Forgetting to clean up VIP visuals on character respawn.** The crown billboard exists on the old character, gone on the new one. Re-apply in `CharacterAdded` after the brief delay for character to load.
- **VIP-only items leaking into non-VIP shop UI.** Either filter the catalog client-side (which the server can override anyway) OR show with a lock icon + "Buy VIP" prompt — choose the conversion-driving one.
- **VIP-area collision group not refreshed on team change.** Team changes that make a player VIP-eligible don't auto-update the collision group. Hook to attribute-changed signals, not just CharacterAdded.
- **Multiple VIP tiers without highest-only logic.** Player owns Bronze AND Gold → applies both → 4.5× multiplier. Apply only highest.
- **Premium membership ignored.** Roblox Premium players who don't own your custom VIP feel left out. Either acknowledge them with smaller perks or document explicitly that "VIP Pass" and "Premium" are different things.
- **Daily VIP reward granted multiple times per day** if `lastVipDailyYday` doesn't update atomically. Update in the same UpdateAsync call as the grant.
- **VIP shop items granted to player without ProcessReceipt-style idempotency.** If you grant on every PromptPurchaseFinished and the event re-fires, double grant. Use a `vipWelcomeGranted` flag.
- **VIP trial timestamp set client-side.** Trial extended forever via clock manipulation. Server-only.
- **Notification "You are now VIP!" fires on every join.** Annoying. Track `vipNotificationShownVersion` per profile and only show on new acquisition or version bumps.

## See also
- [monetization.md](monetization.md) — Game Pass primitives this card is built on (UserOwnsGamePassAsync, PromptGamePassPurchase, PromptGamePassPurchaseFinished)
- [shops.md](shops.md) — VIP shop tab, vipOnly catalog flag, discount in purchase flow
- [interactions.md](interactions.md) — ProximityPrompt-gated VIP doors / portals
- [collisions.md](collisions.md) — selective-barriers pattern for VIP-only physical areas
- [progression.md](progression.md) — `gamepass` multiplier slot in the multiplier stack
- [player-badges.md](player-badges.md) — VIP crown above-head badge
- [datastores.md](datastores.md) — persisting `vipWelcomeGranted`, `lastVipDailyYday`, `vipDailyStreak`, `vipTrialUntil`
- [memorystores.md](memorystores.md) — VIP-only matchmaking queue
- [avatar-customization.md](avatar-customization.md) — VIP-exclusive emotes / cosmetics via HumanoidDescription
- [vfx.md](vfx.md), [sound.md](sound.md) — VIP visual flair (sparkles, crown shine)
- [security.md](security.md) — server-side VIP detection; never trust client claim
- [save-slots.md](save-slots.md) — VIP applies across all slots (it's per-account, not per-character)
