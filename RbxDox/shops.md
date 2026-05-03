# Shops

> Source: distilled from the standard Roblox shop pattern combining UI + currency + Robux + vendor NPC

## Purpose
A shop is a composition: a **catalog** of items (each with id, name, price, currency type, optional stock limit), a **UI** that displays it, a **purchase flow** that goes client → server → validate → grant, and (often) a **vendor NPC** or world object that opens it. Pieces live in `monetization.md` (Robux), `datastores.md` (currency persistence), `ui-system.md` (UI), `interactions.md` (vendor NPCs) — this card pulls them together.

## Catalog data shape

A shop catalog is a static (or DataStore-driven) table. Define once, reference everywhere:

```lua
--!strict
export type CurrencyType = "Coins" | "Gems" | "Robux"

export type ShopItem = {
    id: string,              -- "HealthPotion", stable across versions
    name: string,            -- display name
    description: string,
    price: number,
    currency: CurrencyType,
    productId: number?,      -- if currency = "Robux"; the developer product id
    iconId: string?,         -- "rbxassetid://..."
    maxStock: number?,       -- nil = unlimited; otherwise sells at most N times per player
    requiresLevel: number?,
}

local Shop = {}

Shop.APOTHECARY: {ShopItem} = {
    {id = "HealthPotion", name = "Health Potion", description = "Restores 50 HP.",
     price = 50, currency = "Coins", iconId = "rbxassetid://...", maxStock = 99},
    {id = "ManaPotion", name = "Mana Potion", description = "Restores 50 MP.",
     price = 75, currency = "Coins"},
    {id = "PremiumElixir", name = "Premium Elixir", description = "Full restore + buff.",
     price = 99, currency = "Robux", productId = 1234567890},
}

return Shop
```

Catalogs live in a `ModuleScript` in `ReplicatedStorage` so both server and client can read them. The client uses the catalog to render the UI; the server uses it to validate purchases.

## Purchase flow (server-authoritative)

```
Client                                Server
  │                                     │
  │── BuyItem("HealthPotion") ────────▶ │
  │                                     │ 1. Look up item in catalog
  │                                     │ 2. Validate: exists, level requirement, stock left
  │                                     │ 3. If currency = Coins/Gems: check balance, deduct, grant
  │                                     │    If currency = Robux: PromptProductPurchase(player, productId)
  │                                     │      → grant happens in ProcessReceipt (see monetization.md)
  │                                     │
  │ ◀──── PurchaseResult(success, ...) ─│
  │                                     │
```

The client never updates currency — it sends intent; the server decides.

```lua
-- ReplicatedStorage/Remotes/Buy (RemoteFunction or RemoteEvent + response)
local BuyRemote = game.ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("Buy")
local Shop = require(game.ReplicatedStorage.Shop)
local PlayerData = require(game.ServerStorage.PlayerData)
local MarketplaceService = game:GetService("MarketplaceService")

local function findItem(catalogId: string, itemId: string): Shop.ShopItem?
    local catalog = Shop[catalogId]; if not catalog then return nil end
    for _, item in catalog do
        if item.id == itemId then return item end
    end
    return nil
end

BuyRemote.OnServerInvoke = function(player, catalogId, itemId)
    if typeof(catalogId) ~= "string" or typeof(itemId) ~= "string" then
        return {ok = false, err = "bad-args"}
    end

    local item = findItem(catalogId, itemId)
    if not item then return {ok = false, err = "no-such-item"} end

    local profile = PlayerData.Get(player)
    if not profile then return {ok = false, err = "no-profile"} end

    -- Level gate
    if item.requiresLevel and profile.level < item.requiresLevel then
        return {ok = false, err = "level-too-low"}
    end

    -- Per-player stock cap
    if item.maxStock then
        local owned = profile.purchaseCount[item.id] or 0
        if owned >= item.maxStock then return {ok = false, err = "out-of-stock"} end
    end

    if item.currency == "Coins" or item.currency == "Gems" then
        local balanceField = string.lower(item.currency)        -- "coins" / "gems"
        if profile[balanceField] < item.price then
            return {ok = false, err = "insufficient-funds"}
        end
        profile[balanceField] -= item.price
        grantItem(player, item)
        profile.purchaseCount[item.id] = (profile.purchaseCount[item.id] or 0) + 1
        return {ok = true, item = item.id}

    elseif item.currency == "Robux" then
        if not item.productId then return {ok = false, err = "no-product"} end
        MarketplaceService:PromptProductPurchase(player, item.productId)
        -- Actual grant happens in ProcessReceipt — see monetization.md
        return {ok = "pending", productId = item.productId}
    end

    return {ok = false, err = "unknown-currency"}
end
```

Notice: for Robux items, the server only **prompts** the purchase. The actual grant happens in `MarketplaceService.ProcessReceipt`, mapped from `productId` to the item — see [monetization.md](monetization.md) for the canonical idempotent grant pattern. The shop's catalog is the lookup source: `productId → item`.

## ProcessReceipt with shop integration

```lua
-- Server, single ProcessReceipt callback that knows about every product
MarketplaceService.ProcessReceipt = function(receiptInfo)
    local player = Players:GetPlayerByUserId(receiptInfo.PlayerId)
    if not player then return Enum.ProductPurchaseDecision.NotProcessedYet end

    -- Find the shop item by productId (loop all catalogs, or pre-build a productId → item map)
    local item = findItemByProductId(receiptInfo.ProductId)
    if not item then return Enum.ProductPurchaseDecision.NotProcessedYet end

    -- Idempotency via PurchaseId — see monetization.md
    local granted = false
    pcall(function()
        purchaseHistoryStore:UpdateAsync(receiptInfo.PurchaseId, function(prev)
            if prev then granted = true; return prev end
            grantItem(player, item)
            return true
        end)
    end)

    return granted and Enum.ProductPurchaseDecision.PurchaseGranted
        or Enum.ProductPurchaseDecision.NotProcessedYet
end
```

## Granting items

`grantItem` is the canonical "give the player this" function. Whatever the item does — adds to inventory, equips a tool, applies a buff — happens here:

```lua
local function grantItem(player: Player, item: ShopItem)
    local profile = PlayerData.Get(player)
    if not profile then return end

    if item.id == "HealthPotion" then
        profile.inventory.HealthPotion = (profile.inventory.HealthPotion or 0) + 1
    elseif item.id == "PremiumElixir" then
        profile.inventory.PremiumElixir = (profile.inventory.PremiumElixir or 0) + 1
    elseif item.id == "MysticSword" then
        local tool = ServerStorage.Tools.MysticSword:Clone()
        tool.Parent = player.Backpack
    end
    -- ... whatever effect each item has
end
```

For larger catalogs, replace the if/elseif chain with a per-item `grant` function in the catalog itself, or a registry table.

## Vendor NPC integration

Open the shop UI when the player triggers a vendor NPC's prompt:

```lua
-- Vendor NPC server script (see npcs.md)
prompt.Triggered:Connect(function(player)
    OpenShopRemote:FireClient(player, "APOTHECARY")
end)

-- Client receives, builds and shows shop UI
OpenShopRemote.OnClientEvent:Connect(function(catalogId)
    local Shop = require(ReplicatedStorage.Shop)
    ShopUI.Open(catalogId, Shop[catalogId])
end)
```

## Shop UI structure

A simple shop has:
- **Header** (shop name, currency display)
- **Item grid** (each cell shows icon, name, price)
- **Detail pane** (selected item — full description, "Buy" button)

For implementation, see [ui-system.md](ui-system.md). For animation polish, [tweens-animation.md](tweens-animation.md). For mobile-friendliness, [input-handling.md](input-handling.md).

## Patterns

### Daily rotating shop

A subset of the catalog is "featured" each UTC day. Use server-side seeding by date so all servers show the same rotation:

```lua
local function getDailyRotation(catalog: {ShopItem}, count: number): {ShopItem}
    local day = os.date("!*t", os.time()).yday          -- UTC day-of-year
    local year = os.date("!*t", os.time()).year
    local seed = day + year * 1000
    local rng = Random.new(seed)

    local indices = {}
    for i = 1, #catalog do indices[i] = i end
    -- Fisher-Yates shuffle with the deterministic RNG
    for i = #indices, 2, -1 do
        local j = rng:NextInteger(1, i)
        indices[i], indices[j] = indices[j], indices[i]
    end

    local rotation = {}
    for i = 1, math.min(count, #indices) do
        table.insert(rotation, catalog[indices[i]])
    end
    return rotation
end
```

Player joining server A and server B on the same day sees the same featured items because the seed is date-based.

### Limited-time / event shop

Add an `availableUntil: number?` (Unix timestamp) field to items. Filter at render and at validate time:
```lua
if item.availableUntil and os.time() > item.availableUntil then
    return {ok = false, err = "expired"}
end
```

### Multi-tab shop (categories)

Define separate catalogs per tab (`Shop.Weapons`, `Shop.Armor`, `Shop.Consumables`). The UI shows tabs; the same purchase flow handles all tabs, with `catalogId` distinguishing.

### Discounted item / sale price

Add a `salePrice: number?` field; if present, charge that and show the original `price` struck through in UI. Validate `salePrice` on the server too — never let the client claim "this is on sale."

### Bulk purchase

Add a `quantity` argument to the buy remote; server multiplies price × quantity, validates total balance, grants in a loop. Cap quantity (e.g., max 99) to prevent UI exploits.

```lua
BuyRemote.OnServerInvoke = function(player, catalogId, itemId, quantity)
    quantity = math.clamp(tonumber(quantity) or 1, 1, 99)
    local item = findItem(catalogId, itemId)
    -- ...
    local total = item.price * quantity
    if profile.coins < total then return {ok = false, err = "insufficient-funds"} end
    profile.coins -= total
    for _ = 1, quantity do grantItem(player, item) end
    return {ok = true, qty = quantity}
end
```

### Owned items / "you have N" indicator

Read `profile.purchaseCount[item.id]` and pass to client UI alongside the catalog. Don't trust the client's own count — always look it up server-side from the profile.

## Pitfalls

- **Trusting the client's price.** Always look up `item.price` server-side from the catalog. Even if the client sent the price in the buy request, ignore it.
- **Race condition on currency** — two simultaneous BuyRemote calls could both pass the balance check before either deducts. Use a per-player lock or order-by-receipt-time, or accept the brief over-spend (and clamp to 0 with `profile.coins = math.max(0, profile.coins - price)`).
- **Granting before deducting.** If grant succeeds and deduct fails (DataStore blip), the player has the item AND the money. Deduct first; grant second; if grant fails, refund.
- **`PromptProductPurchase` and assuming success.** Wait for `ProcessReceipt`. The client's `PromptProductPurchaseFinished` `wasPurchased` is informational only; only `ProcessReceipt` returning `PurchaseGranted` confirms the grant.
- **Shop UI showing currency from a stale client cache** — always sync the canonical balance after each purchase via a RemoteEvent push. Don't let client and server diverge.
- **Sale prices stored client-side.** A sale flag tampered by the client doesn't change the server price, but it does mean the UI lies — and players will scream "false advertising." Push the sale state from server.
- **Forgetting `maxStock` is per-player, not per-server.** Persist purchase counts in the profile (DataStore), not in a server-local table.
- **Shop items granted to the wrong slot** in multi-slot games — grant to the player's *active* slot's profile, not all slots. See [save-slots.md](save-slots.md).
- **Robux item without a `productId`** — server `:PromptProductPurchase(player, nil)` errors. Validate at catalog load time.
- **No level-gate on a level-locked item** — players can buy past requirements via direct remote calls. Always re-check level/conditions server-side.

## See also
- [monetization.md](monetization.md) — Robux purchases, ProcessReceipt idempotency
- [datastores.md](datastores.md) — currency and purchaseCount persistence
- [ui-system.md](ui-system.md) — building the shop UI
- [interactions.md](interactions.md) — ProximityPrompt on a vendor NPC opens the shop
- [npcs.md](npcs.md) — vendor NPC composition
- [security.md](security.md) — server-side validation discipline
- [save-slots.md](save-slots.md) — granting to active slot only
- [tweens-animation.md](tweens-animation.md) — shop open/close transitions
- [community-libraries.md](community-libraries.md) — ProfileService for the profile updates this card mutates
