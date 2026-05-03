# MessagingService & HttpService

> Upstream: <https://create.roblox.com/docs/cloud-services/cross-server-messaging> · <https://create.roblox.com/docs/cloud-services/http-service>
> Source: `content/en-us/cloud-services/cross-server-messaging.md`, `content/en-us/cloud-services/http-service.md`

## Purpose
- **MessagingService** — pub/sub between servers in the same experience. Fire-and-forget, low-latency, no persistence. Use for "this server tells the others something happened."
- **HttpService** — outbound HTTP from the server (and JSON helpers + GUID generation everywhere). Use for webhooks, REST APIs, your backend.

Both are server-only.

## MessagingService

```lua
local Messaging = game:GetService("MessagingService")

-- Subscriber (returns RBXScriptConnection-like)
local conn = Messaging:SubscribeAsync("RoundEnded", function(msg)
    -- msg.Data: the published payload (table or scalar)
    -- msg.Sent: epoch seconds the message was published
    print("Round ended on server", msg.Data.serverJobId)
end)

-- Publisher
Messaging:PublishAsync("RoundEnded", {
    serverJobId = game.JobId,
    winner = "Red",
})

conn:Disconnect()
```

### Properties
- Topics are arbitrary strings, scoped per experience. No need to register; subscribe and publish freely.
- Payloads serialize as JSON (so: tables of basic types only — no Instances, functions, userdata).
- **Fire-and-forget.** No delivery guarantee, no ordering guarantee. Use MemoryStore Queue for guaranteed-delivery patterns.
- Per-topic and per-experience throughput limits (~150–600 messages/min depending on player count).

### When to use
- Round-end announcements that should affect every server.
- Triggering a global event (admin: "give everyone in every server a free hat").
- Cross-server chat / friend pings.
- Letting servers learn about each other (each publishes a heartbeat with its JobId).

## HttpService

```lua
local Http = game:GetService("HttpService")

-- Synchronous JSON helpers (these don't yield, no network):
local s = Http:JSONEncode({a = 1, b = "two"})
local t = Http:JSONDecode(s)
local guid = Http:GenerateGUID(false)            -- without curly braces

-- Network calls (yield, can fail):
local body = Http:GetAsync(url, nocache?, headers?)
local body = Http:PostAsync(url, body, contentType?, compress?, headers?)

-- Modern, headers + method explicit:
local response = Http:RequestAsync({
    Url = "https://api.example.com/x",
    Method = "POST",
    Headers = {
        ["Content-Type"] = "application/json",
        ["Authorization"] = "Bearer " .. token,
    },
    Body = Http:JSONEncode({hello = "world"}),
})
-- response: {Success: bool, StatusCode: int, StatusMessage: str, Headers: {...}, Body: str}
```

### Setup
HttpService is **disabled by default**. Enable in `Game Settings → Security → Allow HTTP Requests`. Without this, all HTTP calls error.

### Limits
- Up to ~500 outbound requests per minute per experience.
- 30 s timeout per request.
- Body and headers must be UTF-8 strings.
- HTTPS only; certain ports/hosts blocked (Roblox itself).

### Patterns

```lua
-- Webhook to Discord
Http:PostAsync("https://discord.com/api/webhooks/.../...", Http:JSONEncode({
    content = "Player " .. player.Name .. " bought item " .. item,
}), Enum.HttpContentType.ApplicationJson)
```

```lua
-- Resilient API call
local function call(req, attempts)
    attempts = attempts or 3
    local delay = 1
    for i = 1, attempts do
        local ok, res = pcall(function() return Http:RequestAsync(req) end)
        if ok and res.Success then return res end
        if i == attempts then error(ok and res.StatusMessage or res) end
        task.wait(delay); delay *= 2
    end
end
```

## Secrets

Hard-coded API keys in scripts get exfiltrated as soon as the place is published. Use **Open Cloud Secrets**: `HttpService:GetSecret(name)` reads a secret you stored in Creator Hub → Configure → Secrets.

```lua
local apiKey = Http:GetSecret("STRIPE_KEY")
```

Never `print` or send a secret over a remote.

## Combining Messaging + Memory + Http

Common pattern: a server processes a heavy thing, persists the result via Http to your backend, then announces "done" via MessagingService so all the other servers update their UI.

```lua
local result = doExpensiveThing()
call({Url = backend, Method = "POST", Body = Http:JSONEncode(result)})
Messaging:PublishAsync("ResultReady", result.id)
```

## Pitfalls

- **MessagingService for state.** It's pub/sub, not memory. If a server is started after the publish, it never sees it. For "current state," use MemoryStore.
- **Trusting external API responses.** Validate everything Http returns. Network responses can be unexpected.
- **No `pcall` around network calls.** A flaky API will crash your server thread.
- **HTTP from a LocalScript.** Doesn't exist. All Http calls are server-side.
- **Forgetting JSONEncode for payloads.** PostAsync wants a string, not a Lua table.
- **Spamming PublishAsync.** Throttled. Batch updates instead.
- **Discord webhooks from the client.** Won't work — and exposing a webhook URL to the client lets anyone abuse it. Server-only.
- **Storing API keys in ReplicatedStorage / scripts.** Use `GetSecret`.

## See also
- [memorystores.md](memorystores.md) — durable cross-server queues
- [security.md](security.md) — server authority
- [services.md](services.md) — service overview
- [live-ops.md](live-ops.md) — full live-ops control plane built on MessagingService + MemoryStore (admin commands, cross-server events, giveaways, toast notifications)
