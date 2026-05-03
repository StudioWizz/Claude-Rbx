# Monetization (MarketplaceService)

> Upstream: <https://create.roblox.com/docs/production/monetization> · <https://create.roblox.com/docs/reference/engine/classes/MarketplaceService>
> Source: distilled from the MarketplaceService API and the official ProcessReceipt guidance

## Purpose
Robux monetization runs through `MarketplaceService`. Three product types cover almost every use case:

| Type | Lifetime | Granted via | Typical use |
|---|---|---|---|
| **Game Pass** | Permanent ownership | `UserOwnsGamePassAsync` check | VIP, double-XP, exclusive area, cosmetic unlock |
| **Developer Product** | Consumable, repeatable | `ProcessReceipt` callback | Currency packs, reroll, instant respawn |
| **Premium Membership** | Subscription (Roblox-platform) | `Players.PlayerMembershipChanged` / `:GetPlayerMembership` | Premium-only rewards, increased payout |

You create gamepasses and devproducts on the experience's Creator Hub page. Each gets an integer **asset ID** you reference from code.

## Key API

```lua
local MarketplaceService = game:GetService("MarketplaceService")

-- Prompts (open the purchase dialog)
MarketplaceService:PromptGamePassPurchase(player, gamePassId)
MarketplaceService:PromptProductPurchase(player, productId)
MarketplaceService:PromptPremiumPurchase(player)
MarketplaceService:PromptPurchase(player, assetId)        -- catalog asset (rare)

-- Ownership / status checks (yield)
MarketplaceService:UserOwnsGamePassAsync(userId, gamePassId)   -- → bool
Players:GetPlayerByUserId(userId)                              -- player must be online; for PlayerMembership
player.MembershipType                                          -- Enum.MembershipType.Premium / None

-- Product info (yield, cached)
MarketplaceService:GetProductInfo(id, infoType)                -- infoType: Enum.InfoType.GamePass / Product / Asset

-- Server-side: ProcessReceipt callback (THIS IS REQUIRED for devproducts)
MarketplaceService.ProcessReceipt = function(receiptInfo): Enum.ProductPurchaseDecision
    -- ...
    return Enum.ProductPurchaseDecision.PurchaseGranted
end

-- Events
MarketplaceService.PromptGamePassPurchaseFinished:Connect(function(player, gamePassId, wasPurchased) end)
MarketplaceService.PromptProductPurchaseFinished:Connect(function(userId, productId, wasPurchased) end)
MarketplaceService.PromptPremiumPurchaseFinished:Connect(function(player) end)
Players.PlayerMembershipChanged:Connect(function(player) end)  -- Premium status changed mid-session
```

## Game Passes — pattern

A game pass is owned permanently once purchased.

```lua
-- Server
local MarketplaceService = game:GetService("MarketplaceService")
local Players = game:GetService("Players")

local VIP_PASS = 12345678

local function applyVIP(player)
    -- Grant VIP perks: speed boost, badge, access, whatever
end

Players.PlayerAdded:Connect(function(player)
    -- Cache the result; UserOwnsGamePassAsync yields and rate-limits
    local ok, owns = pcall(function()
        return MarketplaceService:UserOwnsGamePassAsync(player.UserId, VIP_PASS)
    end)
    if ok and owns then applyVIP(player) end
end)

-- Mid-session purchase: grant immediately
MarketplaceService.PromptGamePassPurchaseFinished:Connect(function(player, gamePassId, wasPurchased)
    if wasPurchased and gamePassId == VIP_PASS then
        applyVIP(player)
    end
end)

-- Client-side purchase prompt
MarketplaceService:PromptGamePassPurchase(player, VIP_PASS)
```

Cache `UserOwnsGamePassAsync` results per session — the check yields and counts against rate limits. Refresh via `PromptGamePassPurchaseFinished` if the player buys mid-session.

## Developer Products — `ProcessReceipt` (most-important code in this card)

Devproducts are consumable. Roblox calls `ProcessReceipt` **on the server** every time a player completes a purchase. Your callback must:
1. Grant the item.
2. Persist that you've granted it (using `PurchaseId`).
3. Return `Enum.ProductPurchaseDecision.PurchaseGranted` only after granting **and** persisting.
4. Be **idempotent** — Roblox may retry the same `PurchaseId` if your callback fails or doesn't return.

```lua
local MarketplaceService = game:GetService("MarketplaceService")
local DataStoreService = game:GetService("DataStoreService")
local Players = game:GetService("Players")

local purchaseHistoryStore = DataStoreService:GetDataStore("PurchaseHistory")

local COIN_PACK_SMALL = 11111111
local COIN_PACK_LARGE = 22222222

local productGrants: {[number]: (Player) -> ()} = {
    [COIN_PACK_SMALL] = function(p) addCoins(p, 100) end,
    [COIN_PACK_LARGE] = function(p) addCoins(p, 1000) end,
}

MarketplaceService.ProcessReceipt = function(receiptInfo)
    local player = Players:GetPlayerByUserId(receiptInfo.PlayerId)
    if not player then
        -- Player left before we could grant — ask Roblox to retry later
        return Enum.ProductPurchaseDecision.NotProcessedYet
    end

    local key = receiptInfo.PurchaseId
    local granted

    local ok, err = pcall(function()
        granted = purchaseHistoryStore:UpdateAsync(key, function(alreadyGranted)
            if alreadyGranted then return alreadyGranted end       -- idempotent: don't re-grant
            local grant = productGrants[receiptInfo.ProductId]
            if not grant then return nil end                       -- unknown product — bail without marking
            grant(player)
            return true                                            -- mark as granted
        end)
    end)

    if not ok then
        warn("ProcessReceipt failed:", err)
        return Enum.ProductPurchaseDecision.NotProcessedYet
    end

    if granted then
        return Enum.ProductPurchaseDecision.PurchaseGranted
    else
        return Enum.ProductPurchaseDecision.NotProcessedYet
    end
end
```

**Why this shape:**
- `PurchaseId` is a unique transaction id from Roblox. Storing it prevents double-granting if Roblox retries.
- `UpdateAsync` ensures atomicity vs concurrent retries from different servers.
- Returning `NotProcessedYet` on any failure asks Roblox to retry — better than granting twice.
- Returning `PurchaseGranted` for an unknown ProductId would silently consume the player's Robux. Don't return PurchaseGranted unless you actually granted something.

## Premium Membership

```lua
local function checkPremium(player)
    return player.MembershipType == Enum.MembershipType.Premium
end

Players.PlayerAdded:Connect(function(player)
    if checkPremium(player) then grantPremiumPerks(player) end
end)

Players.PlayerMembershipChanged:Connect(function(player)
    if checkPremium(player) then grantPremiumPerks(player) else revokePremiumPerks(player) end
end)

-- Prompt the player to upgrade
MarketplaceService:PromptPremiumPurchase(player)
```

For payout-share (Premium Payouts), Roblox handles the payment based on player engagement time; no API call needed.

## Reading product info (price, name, description)

```lua
local ok, info = pcall(function()
    return MarketplaceService:GetProductInfo(productId, Enum.InfoType.Product)
end)
if ok then
    print(info.Name, info.Description, info.PriceInRobux, info.IconImageAssetId)
end
```

`GetProductInfo` is cached and yields. Use it to populate a shop UI; never hardcode prices client-side (Robux prices can change, and you should always show the current value).

## Pitfalls

- **No `ProcessReceipt` callback set** → players are charged Robux but receive nothing. Roblox will retry indefinitely until you return `PurchaseGranted`.
- **`ProcessReceipt` not idempotent** → players granted the same item multiple times. Always store and check `PurchaseId`.
- **Returning `PurchaseGranted` without persisting** → if the server crashes between grant and persist, retry will grant again. Persist *before* returning PurchaseGranted.
- **`UserOwnsGamePassAsync` called every frame** → throttled. Cache per player per session.
- **Trusting the client about gamepass ownership** → exploit clients can lie. Always check server-side with `UserOwnsGamePassAsync`.
- **Trusting `PromptGamePassPurchaseFinished.wasPurchased` as proof of ownership without re-checking** → wasPurchased reflects the dialog outcome. Best practice is still to call `UserOwnsGamePassAsync` to confirm before granting.
- **Devproduct grant on the client** → exploiters bypass it. Devproduct grants must happen in `ProcessReceipt` on the server.
- **PromptPurchase from the server with a non-online player** → silently fails. Prompts only work for currently-connected players.
- **GamePass and DevProduct IDs aren't interchangeable.** `UserOwnsGamePassAsync` only works for gamepasses; calling it with a devproduct id returns false.
- **No `pcall` around async calls** → network blip propagates as an unhandled error and you lose the grant logic. Always pcall.
- **Refunds and chargebacks aren't surfaced.** If you grant a permanent perk and Roblox refunds the purchase later, your code won't know. Server-side ownership checks via `UserOwnsGamePassAsync` reflect refunds for gamepasses; devproduct refunds you can't detect, so design devproducts as consumed-on-purchase value rather than persistent unlocks.

## See also
- [datastores.md](datastores.md) — `PurchaseHistory` store for ProcessReceipt idempotency
- [security.md](security.md) — server-authoritative grants; never trust the client
- [services.md](services.md) — MarketplaceService overview
- [shops.md](shops.md) — full shop pattern that uses MarketplaceService for Robux items
- [ui-system.md](ui-system.md) — building a shop UI that calls `PromptPurchase`
- [community-libraries.md](community-libraries.md) — ProfileService for the player-data store these grants update
- [vip-systems.md](vip-systems.md) — VIP gamepass application pattern (the most common gamepass use case)
- [trading-and-gifting.md](trading-and-gifting.md) — Robux gifting via cross-account ProcessReceipt routing
