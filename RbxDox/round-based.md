# Round-Based / Mini-Game Framework

> Source: distilled from party-game, fighting-arena, tower-defense, and round-based PvP patterns

## Purpose
Many Roblox genres are organized as **rounds**: lobby → countdown → active gameplay → results → loop. Party games rotate through mini-games; fighting arenas run match-after-match; tower defense pushes wave-after-wave. The shape is a **state machine on the server** that drives countdowns, picks the next mini-game/map, manages player participation, broadcasts state to clients, and handles late-joiners. This card provides the canonical framework, plus the voting and mini-game-registry patterns built on top.

## Round states

```lua
--!strict
type RoundState = "Lobby" | "Voting" | "Countdown" | "Active" | "Ending" | "Intermission"
```

| State | Duration | Behavior |
|---|---|---|
| **Lobby** | Until enough players | Players wait, browse shop, customize. No round logic running. |
| **Voting** | 15s | Players vote on next map / mode. |
| **Countdown** | 5s | Selected map loads, players teleport in, "starting in 5… 4… 3…" |
| **Active** | Variable (e.g., 3 min) | Round runs. Win conditions checked. |
| **Ending** | 8s | Winner announced, rewards granted, fanfare plays. |
| **Intermission** | 5s | Cleanup, return players to lobby. Loop. |

## RoundManager — server-side state machine

```lua
--!strict
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local MIN_PLAYERS = 2
local STATE_DURATIONS = {Voting = 15, Countdown = 5, Active = 180, Ending = 8, Intermission = 5}

local RoundManager = {}

local state: RoundState = "Lobby"
local stateEndsAt = 0
local currentMap: string? = nil
local participants: {[Player]: boolean} = {}
local stateRemote = ReplicatedStorage:WaitForChild("RoundState") :: RemoteEvent

local function broadcast()
    stateRemote:FireAllClients({
        state = state,
        endsAt = stateEndsAt,
        map = currentMap,
        participants = participants,
    })
    -- Also reflect into a Folder for newly-joining clients to read
    ReplicatedStorage:SetAttribute("RoundState", state)
    ReplicatedStorage:SetAttribute("RoundEndsAt", stateEndsAt)
    ReplicatedStorage:SetAttribute("RoundMap", currentMap)
end

local function transition(newState: RoundState)
    state = newState
    stateEndsAt = os.clock() + (STATE_DURATIONS[newState] or 0)
    broadcast()
end

local handlers: {[RoundState]: () -> RoundState?} = {
    Lobby = function()
        if #Players:GetPlayers() >= MIN_PLAYERS then return "Voting" end
        return nil
    end,
    Voting = function()
        currentMap = Voting.Tally()      -- see Voting section
        return "Countdown"
    end,
    Countdown = function()
        loadMap(currentMap)
        for _, p in Players:GetPlayers() do
            participants[p] = true
            teleportToMap(p, currentMap)
        end
        return "Active"
    end,
    Active = function()
        local winner = checkWinCondition() or pickFallbackWinner()
        announceWinner(winner)
        return "Ending"
    end,
    Ending = function()
        grantRewards()
        return "Intermission"
    end,
    Intermission = function()
        cleanupMap()
        for p in participants do returnToLobby(p) end
        participants = {}
        return "Lobby"
    end,
}

-- Tick loop
task.spawn(function()
    while true do
        task.wait(0.5)
        if state == "Lobby" then
            local next = handlers.Lobby()
            if next then transition(next) end
        elseif os.clock() >= stateEndsAt then
            local next = handlers[state]()
            if next then transition(next) end
        end
    end
end)

return RoundManager
```

The state machine **drives time forward** via the `stateEndsAt` deadline. When `os.clock()` passes it, the current state's handler runs and decides the next state.

`Lobby` is special — it has no fixed duration; it transitions only when `MIN_PLAYERS` is reached.

## Win conditions — pluggable

Extract the per-mini-game logic so the framework doesn't grow:

```lua
type MiniGame = {
    id: string,
    name: string,
    duration: number,
    setup: (participants: {Player}) -> (),
    checkWin: () -> Player?,           -- nil = no winner yet; non-nil = winner found
    cleanup: () -> (),
}

local MiniGames: {[string]: MiniGame} = {
    LastManStanding = {
        id = "LMS", name = "Last Man Standing", duration = 180,
        setup = function(players) for _, p in players do giveRandomWeapon(p) end end,
        checkWin = function()
            local alive = {}
            for p in participants do
                local hum = p.Character and p.Character:FindFirstChildOfClass("Humanoid")
                if hum and hum.Health > 0 then table.insert(alive, p) end
            end
            return #alive == 1 and alive[1] or nil
        end,
        cleanup = function() removeAllWeapons() end,
    },
    -- ...
}
```

The framework loads a mini-game by id, runs `setup(participants)` on Countdown→Active, polls `checkWin()` during Active, and runs `cleanup()` on Active→Ending.

## Voting

Players vote on the next mini-game during the Voting state. Pick by majority (with random tie-break).

```lua
local Voting = {}
local votes: {[Player]: string} = {}
local options: {string} = {}

function Voting.Begin(opts: {string})
    options = opts
    votes = {}
    voteRemote:FireAllClients({options = options})
end

function Voting.Cast(player: Player, optionId: string)
    if not table.find(options, optionId) then return end
    votes[player] = optionId
end

function Voting.Tally(): string
    local counts: {[string]: number} = {}
    for _, opt in options do counts[opt] = 0 end
    for _, opt in votes do counts[opt] = (counts[opt] or 0) + 1 end

    -- Find max
    local maxCount = 0
    local winners = {}
    for opt, c in counts do
        if c > maxCount then maxCount = c; winners = {opt}
        elseif c == maxCount then table.insert(winners, opt) end
    end
    return winners[math.random(#winners)]    -- random tie-break
end

return Voting
```

Pair with a UI showing the three choices and their current vote counts (push live updates as votes come in).

## Late-joiners

A player joining mid-round shouldn't drop into a chaotic Active state without context. Strategies:

| Strategy | Behavior |
|---|---|
| **Spectate** | Joiner enters as a spectator (camera follows a random participant); plays next round |
| **Lobby hold** | Joiner stays in lobby until current round ends |
| **Drop-in** | Joiner spawns into the active map at a default position (works for sandbox/social, not competitive) |

Read the current round state from the replicated attributes (`ReplicatedStorage:GetAttribute("RoundState")`) on PlayerAdded and route accordingly.

```lua
Players.PlayerAdded:Connect(function(player)
    local state = ReplicatedStorage:GetAttribute("RoundState")
    if state == "Active" then
        sendToSpectatorMode(player)
    else
        teleportToLobby(player)
    end
end)
```

## Score tracking per round

For round-based games with multiple winners or partial credit, track scores in a per-round table reset on Intermission:

```lua
local roundScores: {[Player]: number} = {}

local function awardScore(player: Player, n: number)
    roundScores[player] = (roundScores[player] or 0) + n
end

-- On Ending, sort and rank
local function rankRound(): {{player: Player, score: number, rank: number}}
    local list = {}
    for p, s in roundScores do table.insert(list, {player = p, score = s}) end
    table.sort(list, function(a, b) return a.score > b.score end)
    for i, entry in list do entry.rank = i end
    return list
end

-- On Intermission
roundScores = {}
```

Persist career stats (total wins, best score) via [datastores.md](datastores.md); the round-table is ephemeral.

## Patterns

### Map rotation without voting (simpler)
```lua
local MAPS = {"Ruins", "Castle", "Forest"}
local mapIndex = 0
local function nextMap(): string
    mapIndex = (mapIndex % #MAPS) + 1
    return MAPS[mapIndex]
end
```

### Wave-based (tower defense, horde mode)
The Active state is a sub-state-machine of waves:
```lua
type WaveState = {wave: number, enemiesRemaining: number}

local function spawnWave(n: number)
    local count = 5 + n * 2
    for _ = 1, count do spawnEnemy(getEnemyTier(n)) end
    return count
end

-- Inside Active checkWin:
checkWin = function()
    if waveState.enemiesRemaining > 0 then return nil end
    if waveState.wave >= TOTAL_WAVES then return getRoundWinner() end
    -- Start next wave
    waveState.wave += 1
    waveState.enemiesRemaining = spawnWave(waveState.wave)
    return nil
end
```

### Round timer UI
The client reads `ReplicatedStorage:GetAttribute("RoundEndsAt")` and renders a countdown. Update every Heartbeat (no need for the server to push every second — endsAt is fixed):
```lua
RunService.RenderStepped:Connect(function()
    local endsAt = ReplicatedStorage:GetAttribute("RoundEndsAt") or 0
    local remaining = math.max(0, endsAt - os.clock())
    timerLabel.Text = string.format("%d:%02d", remaining // 60, remaining % 60)
end)
```

### Pause / abort
For an admin-triggered pause: set `state = "Paused"` and skip the tick handler until unpaused. For an abort: jump straight to `Intermission`.

## Pitfalls

- **State stored client-side.** Players manipulating "I'm the winner" client claim → exploitable. State and win-condition checks are server-only.
- **Forgetting to reset between rounds.** Stale weapons in inventory, leftover decorations, cached score from previous round — all bugs. `cleanup()` must be thorough.
- **`os.clock()` for the deadline.** Resets on server restart; if you persist `stateEndsAt` to DataStore for resume-after-crash, use `os.time()` (epoch seconds) instead.
- **Tick interval too coarse.** A 0.5s tick is fine for state transitions but bad for win-condition polling in fast games. Poll win conditions at Heartbeat rate during Active.
- **Late-joiner Active state confusion.** Spawning them into the map without context = bad UX. Spectate or hold in lobby.
- **MIN_PLAYERS check too strict.** A 4-player party game with `MIN_PLAYERS = 4` never starts on a sparse server. Start at 2 with a "more players will join" hint, scale difficulty/duration with player count.
- **Win condition that ties forever.** If two players are simultaneously alive in LMS for the entire duration, `checkWin` returns nil until the timer expires. Have a `pickFallbackWinner()` (most damage dealt, last to take damage, etc.) when the timer ends.
- **Voting tie always picks first option.** Players notice and vote-game it. Random tie-break is more fair.
- **State broadcast every Heartbeat.** Burns network. Broadcast on transition only; clients compute remaining time from `endsAt`.
- **Cleanup that destroys players' tools.** If you `cleanup()` by destroying every Tool in Backpack, you've nuked equippable cosmetics too. Track what the round added and only remove that.
- **Round ends but players still in the active map.** Forgot the `returnToLobby` step. Symptoms: next round's countdown begins while previous-round players are still mid-air.

## See also
- [oop-patterns.md](oop-patterns.md) — state machine pattern this card builds on
- [memorystores.md](memorystores.md) — cross-server matchmaking that fills lobbies before round starts
- [teleport.md](teleport.md) — moving players between lobby place and dedicated round servers
- [datastores.md](datastores.md) — persisting career stats / wins / leaderstats
- [leaderstats.md](leaderstats.md) — cumulative wins display
- [live-ops.md](live-ops.md) — admin "force start round" / "skip round" commands; cross-server schedule
- [ui-system.md](ui-system.md) — vote UI, round timer, results screen
- [tweens-animation.md](tweens-animation.md) — countdown VFX, results banner
- [npcs.md](npcs.md) — wave-spawned enemies for horde modes
- [enemy-spawning.md](enemy-spawning.md) — full wave-spawning patterns (used by horde-mode round-based games)
- [enemy-difficulty.md](enemy-difficulty.md) — endless difficulty scaling per wave
- [combat.md](combat.md) — round-time damage and win-condition polling
