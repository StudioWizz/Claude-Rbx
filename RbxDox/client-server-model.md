# Client-Server Model

> Upstream: <https://create.roblox.com/docs/projects/client-server>
> Source: `content/en-us/projects/client-server.md`

## Purpose
A Roblox experience always runs as one **server** plus N **clients** (one per player). Each runs separate Luau VMs with separate Instance trees that the engine partially synchronises. Knowing what runs where, what gets replicated, and what the server can trust is fundamental to every gameplay system.

## What runs where

- **Server** runs `Script` objects in `ServerScriptService`/`ServerStorage`/`Workspace`. It owns the canonical world state and is the only place where things like DataStores, MessagingService, and global game state should be authoritative.
- **Client** runs `LocalScript` objects under the player (PlayerScripts, PlayerGui, Backpack, Character). Each client sees its own copy of the replicated world and its own local-only changes.
- **Shared** code lives in `ModuleScript`s in `ReplicatedStorage`; both sides `require()` and run their own copy.

## What replicates

The engine auto-replicates `Workspace`, `ReplicatedStorage`, `Lighting`, `SoundService`, `Players`, `Teams`, etc. from server → client. Property changes on replicated Instances also stream. **Client → server changes do NOT replicate**, with two exceptions:
1. The client owns its own character physics by default (the server accepts the position the client reports — see network ownership in [security.md](security.md)).
2. Tool/Humanoid input that the engine itself routes (e.g. `Tool.Activated`).

For all other client → server communication, you use **RemoteEvent** or **RemoteFunction**.

## Patterns

### Server → all clients
```lua
-- Server
local remote = game:GetService("ReplicatedStorage").RoundStarted
remote:FireAllClients(roundNumber)

-- Client (LocalScript)
remote.OnClientEvent:Connect(function(roundNumber)
    print("Round", roundNumber, "began")
end)
```

### Client → server (with validation!)
```lua
-- Client
remote:FireServer({action = "buy", itemId = 42})

-- Server — NEVER trust the payload
remote.OnServerEvent:Connect(function(player, payload)
    if typeof(payload) ~= "table" then return end
    if payload.action ~= "buy" then return end
    if typeof(payload.itemId) ~= "number" then return end
    -- now do the actual purchase logic
end)
```

### Server → specific client
```lua
remote:FireClient(player, data)
```

## Pitfalls

- **Trusting the client.** Any value sent from a client is attacker-controlled. Validate types, ranges, and ownership server-side before acting.
- **Confusing replicated edits.** A client editing a part in `Workspace` only changes it on that client; the server (and other clients) won't see it. To make changes authoritative, do them server-side.
- **Yielding inside `OnServerEvent` without queueing.** Multiple events can arrive between yields; design handlers to be re-entrant or use a per-player debounce/queue.
- **Using `RemoteFunction` for server → client return values.** A malicious client can yield forever and hang your server thread. Use `RemoteFunction` only client → server, and even then prefer `RemoteEvent` + a separate response event when you can.
- **Forgetting `WaitForChild`.** On the client, replicated children may not exist yet on first frame. Use `instance:WaitForChild("Name")` rather than direct indexing.

## See also
- [events-signals.md](events-signals.md) — RemoteEvent / RemoteFunction / BindableEvent
- [security.md](security.md) — server authority, validation, network ownership
- [project-structure.md](project-structure.md) — where each service replicates
