# Leaderstats (Player Scoreboard)

> Upstream: <https://create.roblox.com/docs/players/leaderboards>
> Source: distilled from the `Player.leaderstats` Folder convention used by virtually every Roblox game

## Purpose
**`leaderstats`** is a magic Folder name. When you parent a `Folder` named exactly `"leaderstats"` to a `Player`, the engine renders its `IntValue` / `StringValue` / `NumberValue` children as the **top-right scoreboard** that every Roblox player sees on press of `Tab` (or by default on screen). This is the canonical, no-UI-required way to show "Stage: 12 · Wins: 3 · Coins: 4,520" — used by obbies, simulators, tycoons, fighting games, racing games, RPGs. It costs nothing to add and players expect it.

## The minimum viable leaderstats

```lua
local Players = game:GetService("Players")

Players.PlayerAdded:Connect(function(player)
    local stats = Instance.new("Folder")
    stats.Name = "leaderstats"               -- exact name; case-sensitive
    stats.Parent = player

    local coins = Instance.new("IntValue")
    coins.Name = "Coins"
    coins.Value = 0
    coins.Parent = stats

    local wins = Instance.new("IntValue")
    wins.Name = "Wins"
    wins.Value = 0
    wins.Parent = stats
end)
```

That's it. The engine auto-detects the folder, builds the scoreboard, and updates whenever a value changes. Each `IntValue.Name` becomes a column header; `Value` becomes the cell.

## Supported child types

| Class | Display |
|---|---|
| `IntValue` | Integer (most common) |
| `NumberValue` | Floating point |
| `StringValue` | Text |
| `BoolValue` | "true" / "false" (rarely useful — prefer text) |

You can have any number of stats; the engine renders columns left-to-right in **child-order** (so the first child added is the leftmost column).

## Updating from gameplay

Just assign `Value`. The scoreboard updates everywhere automatically (server-set values replicate to all clients):

```lua
local function awardCoins(player: Player, amount: number)
    local stats = player:FindFirstChild("leaderstats")
    if not stats then return end
    local coins = stats:FindFirstChild("Coins")
    if coins then coins.Value += amount end
end
```

Set on the **server**, not the client. Client-set values don't replicate to the scoreboard for other players.

## Common patterns

### Tied to persistent profile
```lua
Players.PlayerAdded:Connect(function(player)
    local profile = loadProfile(player)        -- DataStore (see datastores.md)

    local stats = Instance.new("Folder")
    stats.Name = "leaderstats"
    stats.Parent = player

    local coins = Instance.new("IntValue")
    coins.Name = "Coins"
    coins.Value = profile.coins or 0
    coins.Parent = stats

    -- Mirror profile changes back to leaderstats
    coins:GetPropertyChangedSignal("Value"):Connect(function()
        profile.coins = coins.Value
    end)
end)
```

The `leaderstats` Folder is the *display*; the profile is the truth. Synchronize them either by mirroring (as above) or by writing to both whenever you grant currency.

### Big-number formatting (hide raw value, show formatted)

`leaderstats` only renders the `Value` directly — there's no format hook. For displays like "1.2M" instead of "1,234,567":
- Use a `StringValue` instead of `IntValue` and write the formatted string.
- Keep the raw integer in a separate **non-leaderstats** value or in a hidden field on the profile.

```lua
local coinsDisplay = Instance.new("StringValue")
coinsDisplay.Name = "Coins"
coinsDisplay.Value = formatBigNumber(profile.coins)    -- "1.2M", "456K", etc.
coinsDisplay.Parent = stats

local function refreshCoins()
    coinsDisplay.Value = formatBigNumber(profile.coins)
end
```

The formatter:
```lua
local function formatBigNumber(n: number): string
    if n >= 1e12 then return string.format("%.2fT", n/1e12)
    elseif n >= 1e9 then return string.format("%.2fB", n/1e9)
    elseif n >= 1e6 then return string.format("%.2fM", n/1e6)
    elseif n >= 1e3 then return string.format("%.1fK", n/1e3)
    end
    return tostring(n)
end
```

### Sort the scoreboard by a stat

By default players are sorted by the **first** numeric leaderstat (descending). For more control, set `Player.PlayerListAddedKey` and `PlayerListRemovedKey` — niche; rarely needed. The simpler path: put the column you want to sort by **first** in child order.

To use a specific stat as the sort key:
```lua
-- The leaderstats child whose Name is "Score" is the sort key
local score = Instance.new("IntValue")
score.Name = "Score"
score.Value = 0
score.Parent = stats
-- Make sure it's the FIRST child (order Of insertion matters)
```

For total custom sort (e.g., "by ELO"), build your own player-list UI and hide the default with `StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.PlayerList, false)`.

### Hide the leaderstats scoreboard
```lua
game:GetService("StarterGui"):SetCoreGuiEnabled(Enum.CoreGuiType.PlayerList, false)
```
Then build your own — common when you want a richer scoreboard with avatars, team colors, KDR, etc. See [ui-system.md](ui-system.md).

### Multi-stat updates from one place
Don't scatter `coins.Value += 1` across 50 scripts. Centralize:

```lua
local Stats = {}
function Stats.Add(player: Player, name: string, amount: number)
    local stats = player:FindFirstChild("leaderstats"); if not stats then return end
    local stat = stats:FindFirstChild(name); if not stat then return end
    stat.Value += amount
end

return Stats
```
Now every script calls `Stats.Add(player, "Coins", 10)` and the implementation can change in one place (e.g. add a multiplier from [progression.md](progression.md)).

## Pitfalls

- **Wrong folder name.** Must be lowercase `leaderstats` — `LeaderStats`, `Leaderstats`, or `leaderStats` won't trigger the magic behavior.
- **Folder parented to the wrong place.** Must be parented under the `Player` (e.g. `player.leaderstats`), not under Workspace, ServerStorage, or the character.
- **Updated client-side.** Other players won't see the change. Update on the server.
- **Removed and re-added every respawn.** Don't recreate `leaderstats` in `CharacterAdded` — it lives for the whole player session. Create in `PlayerAdded` only.
- **Adding too many columns.** The scoreboard gets crowded past 4–5 columns and is unreadable on mobile. Pick the most important stats; put the rest in a custom UI menu.
- **`IntValue` overflow at 2^31.** A coin counter can hit this in idle/simulator games. Switch to `NumberValue` (double-precision) once you expect totals over ~2 billion, or use `StringValue` with a BigInt-style serialization.
- **Raw display of huge numbers.** "1234567890 coins" is unreadable. Use the formatter above.
- **Updating per-frame.** Each `Value` set replicates. Setting coins 60 times per second in a Heartbeat is wasteful. Update on events (kill, click, tick) — and at most ~10Hz for displays that need to look smooth.
- **Sorting confusion.** The first numeric leaderstat is the sort key. Add stats in the order you want them displayed AND sorted by.
- **Forgetting to save the underlying profile.** Leaderstats are display only — they don't persist by themselves. The actual values must be saved via [datastores.md](datastores.md) on PlayerRemoving + BindToClose.

## See also
- [datastores.md](datastores.md) — persisting the values backing leaderstats
- [player-lifecycle.md](player-lifecycle.md) — create the folder in PlayerAdded
- [ui-system.md](ui-system.md) — building a custom scoreboard when leaderstats isn't enough
- [progression.md](progression.md) — XP and level columns commonly displayed in leaderstats
- [save-slots.md](save-slots.md) — leaderstats reflect the *active* slot's values
- [services.md](services.md) — Players service basics
