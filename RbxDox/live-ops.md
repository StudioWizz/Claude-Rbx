# Live-Ops (Admin, Multi-Server Events, Notifications)

> Source: distilled from MessagingService + MemoryStore + TextChatService + Open Cloud patterns for live-operations control of a running experience

## Purpose
Once a game is published, you need to **operate** it: kick a misbehaving player, push a hotfix flag without re-publishing, fire a coordinated boss spawn at 6PM Saturday across every server, broadcast "Player Adam just found a Mythic Sword!" to the whole game-world, run a holiday giveaway. This is **live-ops** — the control plane sitting on top of the game systems already covered. The pieces all exist (`chat.md` for slash commands, `security.md` for UserId allowlists, `messaging-http.md` for cross-server pub/sub, `community-libraries.md` Cmdr for command consoles, `monetization.md` for grants), but assembling them into a coherent admin/event/notification surface is its own discipline.

## What live-ops covers

- **Admin commands** — in-game `/kick`, `/teleport`, `/give`, `/event-start`, gated by an allowlist.
- **Server-wide events** — boss spawn, double-XP weekend, holiday mode active.
- **Cross-server (game-wide) coordination** — every server in the experience reacts to the same event simultaneously.
- **Scheduled events** — at a specific real-world UTC time, fire X.
- **Notifications** — toast-style "Player X did Y" broadcasts, server-wide or game-wide.
- **Giveaways** — grant currency / items / passes to all online players (or a subset).
- **Remote control** — flip a flag from outside the game (Open Cloud) without re-publishing.

## Admin allowlist (the foundation)

Every admin action must be gated by **server-side identity check** against an allowlist of `UserId`s. Names are spoofable; UserIds aren't (they're issued by Roblox).

```lua
-- ServerStorage/Admin/Allowlist.lua
return table.freeze({
    [123456789] = "owner",       -- Adam — full access
    [987654321] = "admin",       -- staff
    [111222333] = "moderator",   -- mods
})
```

```lua
-- ServerStorage/Admin/Permissions.lua
local Allowlist = require(ServerStorage.Admin.Allowlist)

local Permissions = {}

function Permissions.GetRole(player: Player): string?
    return Allowlist[player.UserId]
end

function Permissions.IsAdmin(player: Player): boolean
    local role = Allowlist[player.UserId]
    return role == "owner" or role == "admin"
end

function Permissions.HasPermission(player: Player, action: string): boolean
    local role = Allowlist[player.UserId]
    if not role then return false end
    -- Per-action role gates
    if action == "kick" then return role == "owner" or role == "admin" or role == "moderator" end
    if action == "give" then return role == "owner" or role == "admin" end
    if action == "event-start" then return role == "owner" end
    return false
end

return Permissions
```

Use `Permissions.HasPermission` in every admin command's first line. See [security.md](security.md) for the broader server-authority discipline.

For more flexible role storage (so you can promote/demote without re-publishing), put the allowlist in a DataStore and reload it periodically — but cache aggressively (admin lists rarely change, and DataStore reads cost quota).

## Admin commands — TextChatCommand pattern

The clean Roblox-native way (covered in [chat.md](chat.md)):

```lua
local TextChatService = game:GetService("TextChatService")
local Permissions = require(game.ServerStorage.Admin.Permissions)

local function makeAdminCmd(name: string, action: string, fn: (Player, string) -> ())
    local cmd = Instance.new("TextChatCommand")
    cmd.Name = name .. "Cmd"
    cmd.PrimaryAlias = "/" .. name
    cmd.Parent = TextChatService

    cmd.Triggered:Connect(function(textSource, raw)
        local player = game:GetService("Players"):GetPlayerByUserId(textSource.UserId)
        if not player or not Permissions.HasPermission(player, action) then return end
        pcall(fn, player, raw)
    end)
end

makeAdminCmd("kick", "kick", function(admin, raw)
    local _, _, name = string.find(raw, "/kick%s+(%S+)")
    local target = game:GetService("Players"):FindFirstChild(name or "")
    if target then target:Kick("Removed by admin") end
end)

makeAdminCmd("give", "give", function(admin, raw)
    local _, _, name, item = string.find(raw, "/give%s+(%S+)%s+(%S+)")
    local target = game:GetService("Players"):FindFirstChild(name or "")
    if target then grantItem(target, item) end
end)

makeAdminCmd("event-start", "event-start", function(admin, raw)
    local _, _, eventName = string.find(raw, "/event%-start%s+(%S+)")
    if eventName then triggerEvent(eventName) end
end)
```

For a richer command console (autocomplete, type-checked args, history), use **Cmdr** — see [community-libraries.md](community-libraries.md).

## Cross-server coordination via MessagingService

The fundamental primitive: one server publishes a topic; every server (in the same experience) subscribed to that topic receives it. **No delivery guarantee, no ordering** — if reliability matters, pair with MemoryStore.

```lua
local MessagingService = game:GetService("MessagingService")

local LiveOps = {}

-- Subscribe (one connection per topic, kept alive)
local subs: {[string]: RBXScriptConnection} = {}

function LiveOps.Subscribe(topic: string, handler: (data: any) -> ())
    if subs[topic] then return end       -- already subscribed
    local ok, conn = pcall(function()
        return MessagingService:SubscribeAsync(topic, function(msg)
            pcall(handler, msg.Data)
        end)
    end)
    if ok then subs[topic] = conn end
end

function LiveOps.Publish(topic: string, data: any)
    pcall(function()
        MessagingService:PublishAsync(topic, data)
    end)
end

return LiveOps
```

### Pattern: trigger an event on every server

```lua
-- One source server (admin command, scheduled job, Open Cloud webhook) publishes:
LiveOps.Publish("BossSpawn", {bossId = "Frostwyrm", at = os.time()})

-- Every server (including the source) subscribed:
LiveOps.Subscribe("BossSpawn", function(data)
    if not data or typeof(data.bossId) ~= "string" then return end
    spawnBoss(data.bossId)
    notifyAllPlayers(string.format("⚠ %s has appeared!", data.bossId), {color = "red"})
end)
```

The publishing server also receives its own message via subscribe — keeps the code path single (don't special-case "I sent it"). Subscribe at server start, in a long-lived script.

### Pattern: scheduled event with multi-server fan-out

Combine the daily-schedule pattern from [time-and-weather.md](time-and-weather.md) with cross-server publish:

```lua
-- One server is the "coordinator" — easiest election: lowest JobId or first to acquire a MemoryStore lock
local function isCoordinator(): boolean
    -- Simple: every server tries to claim the lock; the one that wins is coordinator for ~60s
    local ok, won = pcall(function()
        return game:GetService("MemoryStoreService"):GetSortedMap("Coordinator")
            :UpdateAsync("primary", function(current)
                if current and current.serverId ~= game.JobId
                   and current.expiresAt > os.time() then return nil end
                return {serverId = game.JobId, expiresAt = os.time() + 60}, 0
            end, 90)
    end)
    return ok and won and won.serverId == game.JobId
end

scheduleDailyAt(18, 0, function()           -- 6PM UTC daily
    if isCoordinator() then
        LiveOps.Publish("DailyBoss", {bossId = "Daily" .. os.date("!*t").wday})
    end
end)
```

Without coordinator election, every server would publish, and every server would receive 30 copies of the same message (one per active server) — boss spawns 30 times. Always elect when fan-in is single-source.

## Notifications — server-wide and game-wide

### "Player X did Y" toast — game-wide

The classic loot brag: when a player finds a legendary item, every player in every server sees a toast.

```lua
-- The triggering server publishes
local function announceRareDrop(player: Player, itemId: string, rarity: string)
    LiveOps.Publish("RareDrop", {
        playerName = player.DisplayName,
        userId = player.UserId,
        itemId = itemId,
        rarity = rarity,
    })
end

-- Every server subscribed
LiveOps.Subscribe("RareDrop", function(data)
    if not data then return end
    -- Validate (msg arrived from network — could be malformed)
    if typeof(data.playerName) ~= "string" or typeof(data.itemId) ~= "string" then return end

    local message = string.format(
        "🎉 %s just found a %s %s!",
        data.playerName, data.rarity, data.itemId
    )

    -- Push to every player on this server
    for _, p in game:GetService("Players"):GetPlayers() do
        ToastRemote:FireClient(p, {
            text = message,
            duration = 5,
            color = Color3.fromRGB(255, 200, 50),
        })
    end
end)
```

The trigger fires once on the source server. Every server picks up the message and routes it to its local players. Players on no servers (offline) miss it — that's fine for ephemeral celebrations.

### Server-wide announcement (single server only)

For "this server's round started," skip MessagingService and use TextChatService directly:

```lua
local TextChatService = game:GetService("TextChatService")
TextChatService.TextChannels.RBXGeneral:DisplaySystemMessage("⚔ Round 5 begins in 30 seconds!")
```

Or push a toast to every player on this server only:

```lua
local function broadcastLocal(message: string, opts: {color: Color3?, duration: number?}?)
    for _, p in game:GetService("Players"):GetPlayers() do
        ToastRemote:FireClient(p, {text = message, color = (opts and opts.color), duration = (opts and opts.duration) or 5})
    end
end
```

### Toast UI primitive (client-side)

A toast is a Frame that slides in from a corner, holds for N seconds, slides out. Pair TweenService with a queue so multiple toasts stack instead of overlapping.

```lua
-- LocalScript under StarterPlayerScripts
local TweenService = game:GetService("TweenService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")

local screen = Instance.new("ScreenGui")
screen.Name = "Toasts"
screen.IgnoreGuiInset = true
screen.ResetOnSpawn = false
screen.Parent = Players.LocalPlayer:WaitForChild("PlayerGui")

local stack = Instance.new("Frame", screen)
stack.AnchorPoint = Vector2.new(1, 0)
stack.Position = UDim2.new(1, -16, 0, 64)
stack.Size = UDim2.fromOffset(360, 0)
stack.BackgroundTransparency = 1
stack.AutomaticSize = Enum.AutomaticSize.Y

local list = Instance.new("UIListLayout", stack)
list.Padding = UDim.new(0, 8)
list.SortOrder = Enum.SortOrder.LayoutOrder
list.HorizontalAlignment = Enum.HorizontalAlignment.Right

local nextOrder = 0

local function showToast(payload: {text: string, color: Color3?, duration: number?})
    nextOrder += 1
    local toast = Instance.new("Frame")
    toast.Size = UDim2.new(1, 0, 0, 48)
    toast.BackgroundColor3 = Color3.fromRGB(30, 30, 35)
    toast.BackgroundTransparency = 1
    toast.LayoutOrder = nextOrder
    Instance.new("UICorner", toast).CornerRadius = UDim.new(0, 8)

    local stripe = Instance.new("Frame", toast)
    stripe.Size = UDim2.new(0, 4, 1, 0)
    stripe.BackgroundColor3 = payload.color or Color3.fromRGB(100, 200, 255)
    stripe.BorderSizePixel = 0

    local label = Instance.new("TextLabel", toast)
    label.BackgroundTransparency = 1
    label.Position = UDim2.new(0, 16, 0, 0)
    label.Size = UDim2.new(1, -24, 1, 0)
    label.Font = Enum.Font.BuilderSansMedium
    label.TextSize = 16
    label.TextColor3 = Color3.fromRGB(240, 240, 240)
    label.TextXAlignment = Enum.TextXAlignment.Left
    label.Text = payload.text
    label.TextTransparency = 1

    toast.Parent = stack

    -- Slide-in fade-in
    local fadeIn = TweenInfo.new(0.3, Enum.EasingStyle.Quad, Enum.EasingDirection.Out)
    TweenService:Create(toast, fadeIn, {BackgroundTransparency = 0.1}):Play()
    TweenService:Create(label, fadeIn, {TextTransparency = 0}):Play()

    task.delay(payload.duration or 5, function()
        local fadeOut = TweenInfo.new(0.4)
        TweenService:Create(toast, fadeOut, {BackgroundTransparency = 1}):Play()
        TweenService:Create(label, fadeOut, {TextTransparency = 1}):Play()
        task.wait(0.4)
        toast:Destroy()
    end)
end

ReplicatedStorage.Remotes.Toast.OnClientEvent:Connect(showToast)
```

Cap the visible toast queue (e.g., max 5 simultaneous) to avoid spam burying the screen.

## Giveaways

A giveaway grants a reward to **every player currently online** across every server. The same MessagingService pattern, with each server granting locally to its own players:

```lua
-- Admin (or scheduled job) triggers
LiveOps.Publish("Giveaway", {
    type = "currency", amount = 100, reason = "Server is back up, sorry!",
})

-- Every server grants to its local players
LiveOps.Subscribe("Giveaway", function(data)
    if typeof(data.amount) ~= "number" or data.amount <= 0 or data.amount > 10000 then return end
    for _, p in game:GetService("Players"):GetPlayers() do
        local profile = PlayerData.Get(p); if not profile then continue end
        profile.coins += data.amount
        ToastRemote:FireClient(p, {text = string.format("🎁 +%d coins (%s)", data.amount, data.reason or "")})
    end
end)
```

For "every player who logs in for the next 24h gets X" (offline-claim style), persist the giveaway window in DataStore and grant on PlayerAdded if the player hasn't yet claimed this specific giveaway. See [offline-processing.md](offline-processing.md) for the daily-claim discipline.

**Validate the payload** — even though it came over MessagingService, the publishing source might be compromised, the message could be malformed, or you might fat-finger an `amount = 1000000` admin command. Cap server-side.

## Remote control / live config (Open Cloud)

For changing behavior **without re-publishing the place**, expose a config in DataStore (or MemoryStore for hot updates) and read it on every relevant decision. Update from outside the game via:
- The DataStore Editor plugin (Studio-side; see [community-libraries.md](community-libraries.md))
- **Open Cloud REST API** (your backend, CI, or a custom admin tool can write directly to DataStores)
- A web dashboard you build that calls Open Cloud

```lua
-- Live config example
local CONFIG_KEY = "LiveConfig"
local liveConfig = {doubleXp = false, dailyBonus = 100}      -- defaults

local function refresh()
    local ok, val = pcall(function()
        return game:GetService("DataStoreService"):GetDataStore("LiveOps"):GetAsync(CONFIG_KEY)
    end)
    if ok and val then
        for k, v in val do liveConfig[k] = v end
    end
end

-- Refresh every 60s (or push via MessagingService on change for instant updates)
task.spawn(function() while true do refresh(); task.wait(60) end end)

-- Use elsewhere
local xp = baseXp * (liveConfig.doubleXp and 2 or 1)
```

For **instant-propagation** updates (no 60-second wait), pair:
1. Admin tool writes to DataStore.
2. Admin tool publishes to a `ConfigChanged` MessagingService topic.
3. Every server subscribed re-reads the DataStore.

Open Cloud lets you do step 1 from a backend or CI without anyone being in the game.

See [messaging-http.md](messaging-http.md) for HttpService patterns and the Open Cloud reference.

## Scheduled events (real-time)

The pattern from [time-and-weather.md](time-and-weather.md) — `scheduleDailyAt(hour, min, fn)` — combined with the coordinator-election + MessagingService publish pattern from above.

```lua
scheduleDailyAt(18, 0, function()
    if not isCoordinator() then return end
    LiveOps.Publish("DailyEvent", {kind = "DoubleXp", duration = 3600})
end)

LiveOps.Subscribe("DailyEvent", function(data)
    if data.kind == "DoubleXp" then
        liveConfig.doubleXp = true
        broadcastLocal("⚡ Double XP for the next hour!")
        task.delay(data.duration, function() liveConfig.doubleXp = false end)
    end
end)
```

Servers that started after the event began won't fire the schedule until the next occurrence — so back the event state with a DataStore/MemoryStore read on server start, so late-starting servers join the event in progress.

## Patterns

### Live shutdown / restart announcement
```lua
local function announceShutdown(minutes: number)
    LiveOps.Publish("ServerShutdown", {atMinutes = minutes})
end

LiveOps.Subscribe("ServerShutdown", function(data)
    broadcastLocal(string.format("⚠ Servers restart in %d minutes!", data.atMinutes), {color = Color3.fromRGB(255, 100, 100)})
end)
```

### Player report → admin notification
```lua
reportRemote.OnServerEvent:Connect(function(reporter, targetUserId, reason)
    LiveOps.Publish("PlayerReport", {
        reporter = reporter.UserId,
        target = targetUserId,
        reason = reason,
        when = os.time(),
        jobId = game.JobId,
    })
end)

-- Admins listen and get an in-game alert if they're online
LiveOps.Subscribe("PlayerReport", function(data)
    for _, p in game:GetService("Players"):GetPlayers() do
        if Permissions.IsAdmin(p) then
            ToastRemote:FireClient(p, {text = string.format("Report from %d: %s", data.reporter, data.reason)})
        end
    end
end)
```

### Hot reload a feature flag
```lua
-- Admin types: /flag doubleXp on
makeAdminCmd("flag", "event-start", function(admin, raw)
    local _, _, name, value = string.find(raw, "/flag%s+(%S+)%s+(%S+)")
    if not name or not value then return end
    local v = (value == "on" or value == "true")
    pcall(function()
        game:GetService("DataStoreService"):GetDataStore("LiveOps")
            :UpdateAsync("LiveConfig", function(cur) cur = cur or {}; cur[name] = v; return cur end)
    end)
    LiveOps.Publish("ConfigChanged", {key = name})
end)
```

### Game-wide "leaderboard reset" event
```lua
scheduleDailyAt(0, 0, function()
    if not isCoordinator() then return end
    LiveOps.Publish("LeaderboardReset", {when = os.time()})
end)

LiveOps.Subscribe("LeaderboardReset", function()
    -- Every server clears its in-memory leaderboard cache
    -- The OrderedDataStore for the new day starts fresh
    resetLocalLeaderboard()
end)
```

## Pitfalls

- **No coordinator election → fan-out duplication.** Every server fires the scheduled event independently, so 30 boss spawns from 30 servers. Elect.
- **Trusting MessagingService payloads.** They're not signed. A compromised server (or a bug in your publish code) can send anything. Validate types/ranges in the subscribe handler.
- **Admin allowlist as a `string` of usernames.** Names change; UserIds are forever. Always allowlist by UserId.
- **Admin commands trigger server-side without rate limit.** A typo in `/give all 1000000` instantly distributes 1M to every player. Cap argument values; require confirmation for nuke-class operations.
- **MessagingService rate limits.** ~150–600 messages/min per topic depending on player count. Don't use it as high-frequency RPC; use RemoteEvents.
- **MessagingService not received because no subscriber yet.** Subscribe at server start, before publishing. New servers don't receive past messages — to catch late-starters up, persist current state in MemoryStore/DataStore and read on server start.
- **Toast spam burying the UI.** Cap visible toasts (max 3–5 stacked); coalesce duplicates ("3 players found legendary items in the last 10s") rather than firing each.
- **Notifying every player about every other player's loot.** Filter by rarity threshold — only "epic+" deserves a game-wide toast. Routine drops are noise.
- **Open Cloud secrets in scripts.** API keys for Open Cloud belong in `HttpService:GetSecret(...)`, not hard-coded. See [messaging-http.md](messaging-http.md).
- **Live-config refresh that loses the local override.** If your refresh blindly overwrites `liveConfig`, any local toggles get reset every cycle. Merge instead of overwrite, or treat live-config as read-only.
- **`scheduleDailyAt` running on a script that gets disabled/re-enabled.** The `task.spawn` loop dies; re-create on script restart from a long-lived bootstrap.
- **No audit trail for admin actions.** When something goes wrong (admin gives 100M coins to wrong player), you want a log. Print + DataStore-append for every admin command.

## See also
- [chat.md](chat.md) — TextChatCommand for the slash-command admin surface
- [security.md](security.md) — UserId allowlist discipline; never trust client claims
- [messaging-http.md](messaging-http.md) — MessagingService rate limits, HttpService for Open Cloud
- [memorystores.md](memorystores.md) — coordinator-election lock, ephemeral cross-server state
- [datastores.md](datastores.md) — live-config persistence, audit logs
- [time-and-weather.md](time-and-weather.md) — `scheduleDailyAt` for real-time triggers
- [monetization.md](monetization.md) — granting items / currency in giveaways
- [community-libraries.md](community-libraries.md) — **Cmdr** for richer admin command consoles; **DataStore Editor** plugin
- [ui-system.md](ui-system.md), [tweens-animation.md](tweens-animation.md) — toast UI primitives
- [player-lifecycle.md](player-lifecycle.md) — late-joiner reconciliation for in-progress events
