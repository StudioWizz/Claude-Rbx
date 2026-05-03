# Trading & Gifting

> Source: distilled from RPG / sandbox / pet-game trade flows and Roblox MarketplaceService gifting patterns

## Purpose
Two related player-to-player transfer patterns: **trading** (mutual exchange — both sides put items in, both confirm, swap) and **gifting** (one-way — sender gives, recipient receives, no exchange). Both are highly exploitable surfaces; this card covers the canonical server-authoritative flows, anti-scam discipline (the classic "swap items at the last second" attack), Robux-driven gifting via MarketplaceService, and trade-history persistence. Builds on `inventory.md` (item ownership), `events-signals.md` (RemoteEvents), and `monetization.md` (Robux gifting via developer products).

## Trading — the mutual exchange flow

```
Player A                    Server                      Player B
   │                          │                            │
   ├── RequestTrade(B) ──────▶│                            │
   │                          │── TradeInvite(A) ─────────▶│
   │                          │                            │
   │                          │◀──── AcceptTrade ──────────┤
   │                          │                            │
   ▼ open trade UI            ├── TradeStarted ───────────▶▼ open trade UI
   │                          │                            │
   ├── PutItem("Sword") ─────▶│                            │
   │                          │── TradeUpdate(A's items) ─▶│
   │                          │                            │
   │                          │◀──── PutItem("Gold")  ─────┤
   │                          │── TradeUpdate(B's items) ─▶▼
   ▼                          │                            │
   ├── ConfirmA ─────────────▶│                            │
   │                          │── TradeUpdate(A confirmed) ▶│
   │                          │                            │
   │                          │◀──── ConfirmB ─────────────┤
   │                          │                            │
   │                          │ ─── ATOMIC SWAP ───        │
   │                          │ ─── persist both profiles  │
   │                          │                            │
   ▼ close UI                ◀── TradeComplete ──────────▶▼ close UI
```

Three rules to follow strictly:

1. **All state on the server.** The client UI displays what the server has confirmed; never trust client state.
2. **Confirmations latch and unlatch.** When either side modifies their offer (add/remove item), **both confirmations clear**. This prevents the "swap-at-confirm" exploit where one player swaps a valuable item for junk just before the swap fires.
3. **Inventory is locked during trade.** Items in the trade can't be sold, dropped, equipped, or used until the trade resolves. Otherwise: A puts a sword in trade, sells it elsewhere, B confirms, A confirms — the server tries to give B a sword that no longer exists.

## TradeManager — server-side core

```lua
--!strict
local Players = game:GetService("Players")
local TradeManager = {}

type TradeState = {
    a: Player, b: Player,
    aOffer: {[string]: number},        -- itemKey → count (works for both stackable + unique uids)
    bOffer: {[string]: number},
    aConfirmed: boolean,
    bConfirmed: boolean,
    startedAt: number,
}

local activeTrades: {[Player]: TradeState} = {}     -- both players point to the same TradeState

local function isInTrade(player: Player): boolean
    return activeTrades[player] ~= nil
end

function TradeManager.Invite(from: Player, to: Player)
    if from == to then return end
    if isInTrade(from) or isInTrade(to) then return end
    if not arePlayersNearby(from, to, 30) then return end       -- proximity check; optional
    TradeInviteRemote:FireClient(to, {from = from})
end

function TradeManager.Accept(initiator: Player, accepter: Player)
    if isInTrade(initiator) or isInTrade(accepter) then return end

    local state: TradeState = {
        a = initiator, b = accepter,
        aOffer = {}, bOffer = {},
        aConfirmed = false, bConfirmed = false,
        startedAt = os.time(),
    }
    activeTrades[initiator] = state
    activeTrades[accepter] = state

    -- Lock items? We use a soft lock: validate each item is still owned at every operation.
    TradeStartedRemote:FireClient(initiator, {with = accepter})
    TradeStartedRemote:FireClient(accepter, {with = initiator})
end

function TradeManager.PutItem(player: Player, itemKey: string, count: number)
    local state = activeTrades[player]; if not state then return end
    if count <= 0 then return end

    local profile = PlayerData.Get(player); if not profile then return end
    local owned = Inventory.Count(profile.inventory, getItemIdFromKey(itemKey))
    local offer = (player == state.a) and state.aOffer or state.bOffer
    local alreadyOffered = offer[itemKey] or 0

    if owned < alreadyOffered + count then return end       -- not enough
    offer[itemKey] = alreadyOffered + count

    -- CRITICAL: changes clear both confirmations
    state.aConfirmed = false
    state.bConfirmed = false

    pushUpdate(state)
end

function TradeManager.RemoveItem(player: Player, itemKey: string, count: number)
    local state = activeTrades[player]; if not state then return end
    local offer = (player == state.a) and state.aOffer or state.bOffer
    local current = offer[itemKey] or 0
    offer[itemKey] = math.max(0, current - count)
    if offer[itemKey] == 0 then offer[itemKey] = nil end

    state.aConfirmed = false
    state.bConfirmed = false

    pushUpdate(state)
end

function TradeManager.Confirm(player: Player)
    local state = activeTrades[player]; if not state then return end

    if player == state.a then state.aConfirmed = true
    elseif player == state.b then state.bConfirmed = true end

    pushUpdate(state)

    if state.aConfirmed and state.bConfirmed then
        TradeManager._ExecuteSwap(state)
    end
end

function TradeManager.Cancel(player: Player)
    local state = activeTrades[player]; if not state then return end
    activeTrades[state.a] = nil
    activeTrades[state.b] = nil
    TradeCancelledRemote:FireClient(state.a, {by = player})
    TradeCancelledRemote:FireClient(state.b, {by = player})
end

function TradeManager._ExecuteSwap(state: TradeState)
    local profileA = PlayerData.Get(state.a)
    local profileB = PlayerData.Get(state.b)
    if not profileA or not profileB then
        TradeManager.Cancel(state.a)
        return
    end

    -- Re-validate both sides still own everything they offered (anti-cheat / item changed during trade)
    if not validateOffer(profileA, state.aOffer) or not validateOffer(profileB, state.bOffer) then
        TradeManager.Cancel(state.a)
        return
    end

    -- Atomic swap: remove from both, then add to both
    -- (Lua is single-threaded so this is naturally atomic for these operations)
    for itemKey, count in state.aOffer do
        Inventory.RemoveByKey(profileA.inventory, itemKey, count)
        Inventory.AddByKey(profileB.inventory, itemKey, count)
    end
    for itemKey, count in state.bOffer do
        Inventory.RemoveByKey(profileB.inventory, itemKey, count)
        Inventory.AddByKey(profileA.inventory, itemKey, count)
    end

    -- Persist immediately (don't wait for autosave; trades are high-stakes)
    saveProfile(state.a)
    saveProfile(state.b)

    -- History
    logTrade(state.a, state.b, state.aOffer, state.bOffer)

    activeTrades[state.a] = nil
    activeTrades[state.b] = nil

    TradeCompletedRemote:FireClient(state.a, {received = state.bOffer})
    TradeCompletedRemote:FireClient(state.b, {received = state.aOffer})
end

local function pushUpdate(state)
    TradeUpdateRemote:FireClient(state.a, {
        myOffer = state.aOffer, theirOffer = state.bOffer,
        myConfirmed = state.aConfirmed, theirConfirmed = state.bConfirmed,
    })
    TradeUpdateRemote:FireClient(state.b, {
        myOffer = state.bOffer, theirOffer = state.aOffer,
        myConfirmed = state.bConfirmed, theirConfirmed = state.aConfirmed,
    })
end

return TradeManager
```

## Anti-scam discipline

The classic exploit: A puts in a "Mythic Sword," B confirms, A swaps it for a "Wood Stick" right before the trade fires. With the mutual-confirmation-clears-on-change rule above, **A can't confirm again until they restore the sword AND B re-confirms** — both sides see the change, both choose to re-accept.

UX-wise, the trade UI should:
- **Show both sides' confirmations as small icons** (✓ green when both confirmed). Clear visually when either side modifies.
- **Display a brief countdown after both confirm** (3 seconds) before the swap fires, with a "Cancel" button. Gives the recipient one last chance to back out if something feels off.
- **Highlight changes** — when the other player adds/removes an item, briefly flash the new entry in red/green so the user notices.

## Player departure handling

If either player leaves mid-trade, cancel:
```lua
Players.PlayerRemoving:Connect(function(player)
    if activeTrades[player] then TradeManager.Cancel(player) end
end)
```

Save profiles on PlayerRemoving (already done by `player-lifecycle.md`); active trades are discarded.

## Trade history (audit log)

For dispute resolution and bot detection, log every completed trade:

```lua
local function logTrade(a: Player, b: Player, aOffer, bOffer)
    local entry = {
        at = os.time(),
        a = a.UserId, b = b.UserId,
        aOffer = aOffer, bOffer = bOffer,
    }
    pcall(function()
        local store = game:GetService("DataStoreService"):GetDataStore("TradeHistory_v1")
        store:UpdateAsync(tostring(a.UserId), function(history)
            history = history or {}
            table.insert(history, entry)
            -- Trim to last 100 entries to control size
            while #history > 100 do table.remove(history, 1) end
            return history
        end)
        -- And for player B too (or skip if you only need one side)
    end)
end
```

For real-time fraud detection, also publish to MessagingService ([live-ops.md](live-ops.md)) — admins can be alerted to unusual trades (e.g., legendaries flowing one direction repeatedly).

## Gifting items (one-way, no exchange)

Simpler than trading because there's no mutual offer. The pattern:

```lua
function GiftSystem.GiveItem(sender: Player, recipient: Player, itemKey: string, count: number): (boolean, string?)
    if sender == recipient then return false, "self-gift" end
    if not arePlayersNearby(sender, recipient, 30) then return false, "out-of-range" end

    local senderProfile = PlayerData.Get(sender); if not senderProfile then return false, "no-profile" end
    local recipientProfile = PlayerData.Get(recipient); if not recipientProfile then return false, "no-recipient-profile" end

    -- Check item exists, validate count
    if Inventory.Count(senderProfile.inventory, getItemIdFromKey(itemKey)) < count then
        return false, "insufficient"
    end

    -- Check recipient has space (capacity)
    if not Inventory.HasSpace(recipientProfile.inventory, itemKey, count) then
        return false, "recipient-full"
    end

    -- Confirm the gift (popup on recipient side: "Accept gift from X?")
    local accepted = pollGiftAcceptance(recipient, sender, itemKey, count)
    if not accepted then return false, "declined" end

    -- Atomic transfer
    Inventory.RemoveByKey(senderProfile.inventory, itemKey, count)
    Inventory.AddByKey(recipientProfile.inventory, itemKey, count)

    saveProfile(sender)
    saveProfile(recipient)
    logGift(sender, recipient, itemKey, count)

    -- Update gifting stats (for badges; see player-badges.md)
    senderProfile.giftsSent = (senderProfile.giftsSent or 0) + 1
    recipientProfile.giftsReceived = (recipientProfile.giftsReceived or 0) + 1

    return true, nil
end
```

The `pollGiftAcceptance` step is important — gifts shouldn't appear silently in the recipient's inventory (could be used to dump unwanted items, or to dump something cursed). Always require explicit Accept or Decline.

## Gifting Robux items (cross-account purchases)

For "buy this developer product as a gift for player X," use MarketplaceService's purchase prompt with the **sender** as the buyer, then in `ProcessReceipt` deliver to the recipient. The Robux comes from the sender's account, the item goes to the recipient.

```lua
local MarketplaceService = game:GetService("MarketplaceService")

-- A "Gift Robux Item" UI flow:
-- 1. Sender selects item + recipient
-- 2. Server records the intended recipient before prompting
-- 3. Sender prompts to purchase
-- 4. ProcessReceipt looks up the recipient and grants there

local pendingGifts: {[number]: {recipient: number, productId: number}} = {}
-- key = senderUserId; or use a more durable map by purchaseId

GiftPurchaseRemote.OnServerEvent:Connect(function(sender, recipientUserId, productId)
    if typeof(recipientUserId) ~= "number" or typeof(productId) ~= "number" then return end
    local recipient = Players:GetPlayerByUserId(recipientUserId)
    if not recipient then
        notifyPlayer(sender, "Recipient must be in this server")
        return
    end

    -- Record intent (server-side; client can't tamper)
    pendingGifts[sender.UserId] = {recipient = recipientUserId, productId = productId}

    MarketplaceService:PromptProductPurchase(sender, productId)
end)

MarketplaceService.ProcessReceipt = function(receiptInfo)
    local intent = pendingGifts[receiptInfo.PlayerId]
    pendingGifts[receiptInfo.PlayerId] = nil    -- consume

    if not intent or intent.productId ~= receiptInfo.ProductId then
        -- This was a regular purchase, not a gift; deliver to the buyer normally
        return grantToPlayer(receiptInfo.PlayerId, receiptInfo.ProductId)
    end

    -- Gift purchase: deliver to recipient
    local recipientUserId = intent.recipient
    local recipient = Players:GetPlayerByUserId(recipientUserId)

    if not recipient then
        -- Recipient left mid-purchase; queue the grant for when they rejoin (DataStore)
        queueGiftForLater(recipientUserId, receiptInfo.ProductId, receiptInfo.PurchaseId)
        return Enum.ProductPurchaseDecision.PurchaseGranted
    end

    -- Grant to recipient using the standard idempotent ProcessReceipt pattern
    local granted = grantToPlayer(recipient.UserId, receiptInfo.ProductId, receiptInfo.PurchaseId)

    -- Notify sender
    notifyPlayer(Players:GetPlayerByUserId(receiptInfo.PlayerId),
                 string.format("You gifted %s to %s!", productName(receiptInfo.ProductId), recipient.Name))

    return granted and Enum.ProductPurchaseDecision.PurchaseGranted
        or Enum.ProductPurchaseDecision.NotProcessedYet
end
```

This works for **same-server gifting** (recipient must be online in the same server). For cross-server gifting (recipient may be offline or in a different server), use a global DataStore key for pending-gifts indexed by recipient UserId; the recipient picks it up when they next join.

For Robux-grade gifting flow, see [monetization.md](monetization.md) for the underlying ProcessReceipt idempotency pattern; this card builds on it.

## Patterns

### Trade with currency
For trading items + currency (sword + 100 gold), treat currency as another "item" in the offer:
```lua
state.aOffer = {Sword_a8d2 = 1, Coins = 100}
```
Validate `profile.coins >= count` instead of `Inventory.Count`.

### Trade lockout window after suspicious activity
A player who joined < 5 minutes ago can't initiate trades (anti-bot, anti-alt). Track `joinedAt` per session, check before allowing trade actions.

### Trade chat
Players in active trade get a private chat channel between them. Use TextChatService (`chat.md`) with a dynamically-created TextChannel.

### Marketplace listings (auction house style)
Players list items with asking prices; other players browse and buy. Each listing is a record in a global DataStore (`MarketListings`); sales transfer the item key from seller to buyer and pay the seller. Higher-traffic alternative: MemoryStore SortedMap for the listings ([memorystores.md](memorystores.md)) with periodic sync to DataStore.

### Gift wrapping
The recipient sees "Gift from Adam" with a wrap animation before the item is revealed. Pure UX polish; doesn't affect the underlying transfer.

### Gift cooldown / daily cap
Limit gifts to N per day per sender to prevent spam (and bot money laundering). Track `giftsToday`, `lastGiftDay` in profile; reset on UTC day boundary.

## Pitfalls

- **Trading state on the client.** Trivially exploitable. State, validation, and the swap are server-side; client only displays.
- **Confirmation that doesn't latch on change.** A single confirmation flag, or per-side flags that don't reset when items change → swap-at-confirm exploit. Always reset both flags on any modification.
- **No re-validation at swap time.** Player A removes the sword from inventory mid-trade (sells it, drops it). At swap time the inventory check passes by stale data. **Always re-validate both inventories at the moment of swap**.
- **No proximity / chat-existence check.** Players can be invited to trade by random users globally — annoying. Require they're in the same server, ideally near each other.
- **Gift items appearing silently in recipient inventory.** Could be exploited to dump unwanted items into someone's bag. Always require explicit Accept.
- **Trade history saved per-trade with a write per side.** Each trade = 2 DataStore writes. With high trade volumes, hit quota. Batch into a single global key + per-day rotation, OR record only on dispute.
- **Atomic swap with `pcall`-wrapped ops where one fails.** Half the items move; half don't. Item duplication or loss. Ensure both sides' Remove + Add succeed before finalizing — if anything errors, restore from snapshot.
- **Player B's profile not loaded when trade fires.** They just joined and haven't finished loading. Validate both profiles exist in `_ExecuteSwap` and gracefully cancel.
- **Robux gift to offline recipient lost.** If recipient leaves before ProcessReceipt fires, the grant disappears. Always queue offline grants in a DataStore for delivery on next join.
- **Robux gift `pendingGifts` lost on server restart.** A purchase initiated, server crashes before ProcessReceipt — the intent is lost, recipient doesn't get the gift, sender paid Robux. Either persist intents or reconcile gracefully (check the receipt against a database of "pending gifts").
- **No server-side cooldown on gift requests.** Bots spam-gift items between alts to launder. Rate-limit per-sender per-period.
- **Player A invites all 30 server players to trade.** Spam. Cap simultaneous outbound invites.
- **Two simultaneous trades involving the same player.** P trades with A, then accepts B's invite while still trading with A. Currently `isInTrade` check prevents this; verify it on every entry point.
- **Trade UI shows item icons but server has different items.** Client-server desync. Always fully re-render from the latest TradeUpdate payload, not from local memory.

## See also
- [inventory.md](inventory.md) — the underlying item store; trading transfers items between profiles
- [monetization.md](monetization.md) — Robux purchases and ProcessReceipt idempotency (foundation of Robux gifting)
- [security.md](security.md) — server-authoritative validation; never trust client trade state
- [events-signals.md](events-signals.md) — RemoteEvents for trade UI updates
- [datastores.md](datastores.md) — saving profiles and trade history
- [player-lifecycle.md](player-lifecycle.md) — cancel trades on PlayerRemoving
- [chat.md](chat.md) — TextChatService for trade chat channel
- [ui-system.md](ui-system.md), [menu-systems.md](menu-systems.md) — trade UI, gift acceptance modal
- [player-badges.md](player-badges.md) — gift-giver badge based on `giftsSent`
- [community-libraries.md](community-libraries.md) — `t` for runtime payload validation; ProfileService for atomic profile saves
