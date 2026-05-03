# In-Game Chat (TextChatService)

> Upstream: <https://create.roblox.com/docs/chat/text-chat-service> · <https://create.roblox.com/docs/reference/engine/classes/TextChatService>
> Source: distilled from the TextChatService API and the migration guidance from the legacy `Chat` service

## Purpose
`TextChatService` is the modern in-game chat system, **replacing the legacy `Chat` service** (which is now in maintenance mode). It models chat as **channels** with **sources** (player chatters) and **messages** that flow through filter callbacks before display. New code should use TextChatService; the legacy LegacyChatService only exists for old places that haven't migrated.

## Architecture

```
TextChatService                          (the service)
├── ChatWindowConfiguration              (visual: text size, position, theme)
├── ChatInputBarConfiguration            (visual: input bar)
├── BubbleChatConfiguration              (visual: speech bubbles above heads)
├── TextChannels (Folder)
│   ├── RBXGeneral                       (default global chat — engine-created)
│   ├── RBXSystem                        (system messages — engine-created)
│   └── (your custom channels)
└── TextChatCommands (Folder)
    └── (your slash commands)
```

`TextChannel` is a server-replicated channel. `TextSource` is a player's membership in a channel. `TextChatMessage` is a single message flowing through.

Set `TextChatService.ChatVersion = Enum.ChatVersion.TextChatService` (it's the default in new places, but verify).

## Key API

```lua
local TextChatService = game:GetService("TextChatService")

-- Send (client-side)
local channel = TextChatService.TextChannels.RBXGeneral
channel:SendAsync("Hello world")

-- Server-side announce
channel:DisplaySystemMessage("Round started!")

-- React (server-side, after server processes the message)
channel.MessageReceived:Connect(function(message)
    -- message: TextChatMessage
    -- message.Text — the (potentially filtered) text
    -- message.TextSource — the TextSource (player) who sent it
end)

-- Server-side filter — return false to drop, true to allow
channel.ShouldDeliverCallback = function(message: TextChatMessage, target: TextSource): boolean
    return true
end

-- Client-side, before sending
TextChatService.SendingMessage:Connect(function(message)
    -- can mutate message.Text or set message.Status to drop
end)

-- Client-side, every received message (any channel)
TextChatService.MessageReceived:Connect(function(message)
    -- update HUD, log, play sound, etc.
end)

-- TextSource (per-player presence in a channel)
local source = channel:AddUserAsync(userId)        -- explicitly add a user to a custom channel
source:Destroy()                                    -- remove
```

## Slash commands — `TextChatCommand`

The clean way to add `/give`, `/kick`, `/team`:

```lua
-- Server-side: under TextChatService, create a TextChatCommand Instance
local cmd = Instance.new("TextChatCommand")
cmd.Name = "GiveCommand"
cmd.PrimaryAlias = "/give"
cmd.SecondaryAlias = "/g"
cmd.Parent = TextChatService

cmd.Triggered:Connect(function(textSource: TextSource, unfilteredText: string)
    local player = game:GetService("Players"):GetPlayerByUserId(textSource.UserId)
    if not player then return end

    -- Permission check (see security.md)
    if not isAdmin(player) then return end

    -- Parse args (`unfilteredText` is the full command incl. "/give")
    local _, _, target, item = string.find(unfilteredText, "^/give%s+(%S+)%s+(%S+)$")
    if not target or not item then return end

    -- Execute
    grantItem(target, item)
end)
```

`unfilteredText` arrives **without** Roblox's chat filter applied — fine for command parsing because the user typed it as input, not displayed text. If your command echoes anything back to chat, filter that text first via `TextService:FilterStringAsync` (see [security.md](security.md)).

## Custom channels (e.g., team chat)

```lua
-- Server: create a channel
local team = Instance.new("TextChannel")
team.Name = "Team_Red"
team.Parent = TextChatService.TextChannels

-- Add players when they join the team
team:AddUserAsync(player.UserId)

-- Client-side: switch the input target to this channel
TextChatService.ChatInputBarConfiguration.TargetTextChannel = TextChatService.TextChannels.Team_Red
```

Players only see messages on channels they're a member of. Removing them removes their access.

## Filtering / moderation

TextChatService **handles platform filtering automatically** — every message goes through Roblox's filter before display. You typically don't need to call `TextService:FilterStringAsync` for messages flowing through TextChatService.

You DO need it for:
- Text the player typed that you display **outside** chat (custom name plates, signs, leaderboards).
- Text from `TextChatCommand.Triggered` that you echo to other players.
- Player-named entities saved to DataStores then displayed later.

See [security.md](security.md) for the broader text-filtering discipline.

You can add **additional** filtering via `ShouldDeliverCallback`:

```lua
-- Block messages containing a banned word
channel.ShouldDeliverCallback = function(message, target)
    if string.find(message.Text:lower(), "bannedword") then return false end
    return true
end
```

This runs server-side per delivery; returning `false` drops the message for that target only.

## Patterns

### Server-side admin command via chat
```lua
local function makeAdminCommand(name: string, fn: (Player, string) -> ())
    local cmd = Instance.new("TextChatCommand")
    cmd.PrimaryAlias = "/" .. name
    cmd.Parent = TextChatService
    cmd.Triggered:Connect(function(textSource, text)
        local player = Players:GetPlayerByUserId(textSource.UserId)
        if not player or not isAdmin(player) then return end
        fn(player, text)
    end)
end

makeAdminCommand("kick", function(_, text)
    local _, _, name = string.find(text, "/kick%s+(%S+)")
    local target = Players:FindFirstChild(name or "")
    if target then target:Kick("Removed by admin") end
end)
```

### Server announcement
```lua
TextChatService.TextChannels.RBXGeneral:DisplaySystemMessage("Boss spawning in 30s!")
```

### Mute the default chat (custom UI replacement)
```lua
-- Client-side
TextChatService.ChatWindowConfiguration.Enabled = false
TextChatService.ChatInputBarConfiguration.Enabled = false
-- Now build your own ScreenGui chat using TextChatService.MessageReceived
```

### Detect a player's chat (server-side)
```lua
TextChatService.TextChannels.RBXGeneral.MessageReceived:Connect(function(message)
    local source = message.TextSource
    if not source then return end   -- system messages
    print(source.Name, "said:", message.Text)
end)
```

## Migrating from legacy `Chat`

| Legacy `Chat` | Modern TextChatService |
|---|---|
| `Players.PlayerChatted` | `TextChannel.MessageReceived` |
| `Chat:Chat(part, message)` (chat bubbles) | `TextChatService` BubbleChatConfiguration + `TextChannel:DisplaySystemMessage` or just `TextChatMessage` |
| Old `ChatService` ModuleScript with custom commands | `TextChatCommand` Instances |
| Manually rolling chat filter | Built-in (TextChatService applies platform filter) |

If your game still uses legacy `Chat`, set `TextChatService.ChatVersion = TextChatService` to switch. Existing legacy chat scripts will stop running; migrate any custom logic to TextChatCommand and channel callbacks.

## Pitfalls

- **`SendingMessage` is client-side and editable.** A malicious client can bypass it. Server-side validation (`MessageReceived`, `ShouldDeliverCallback`) is the trusted point.
- **`Triggered` text isn't filtered.** Fine for command parsing; not fine for echoing to other players without filtering.
- **`ChatVersion` set to `LegacyChatService`** in older places means TextChatService won't fire any signals. Check the version property if confused.
- **Custom channels and `AddUserAsync`** — the user must be in the channel to send to it. Auto-add players on the relevant lifecycle event (PlayerAdded, team change, etc.).
- **`TextChatCommand` parsing is yours.** No built-in arg parser; use `string.find` patterns or split on whitespace.
- **Removing the default `RBXGeneral` channel** breaks expectations — players expect a global chat. Hide the UI instead of removing the channel.
- **Bubble chat** (`BubbleChatConfiguration.Enabled = true`) is separate from the chat window; configure both if you want speech bubbles AND the window.
- **Rate limits** — Roblox throttles per-player message rate (a few per second). Don't try to use chat as a high-throughput data channel; that's what RemoteEvents are for.
- **Server-broadcasting via `DisplaySystemMessage`** appears as a system-styled message, not from a player. For "Notice from the host," that's correct.

## See also
- [security.md](security.md) — `TextService:FilterStringAsync` for any player-typed text you display outside TextChatService
- [services.md](services.md) — TextChatService overview
- [events-signals.md](events-signals.md) — chat-driven actions usually go through RemoteEvents for game effects
- [live-ops.md](live-ops.md) — full admin-command + allowlist + cross-server pattern this card's TextChatCommand fits into
- [community-libraries.md](community-libraries.md) — Cmdr is a richer alternative for admin command consoles
