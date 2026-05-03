# Time Trials (Start, Splits, Finish, Leaderboards)

> Source: distilled from racing / parkour / obby / speedrun time-trial patterns

## Purpose
A time-trial system measures how fast a player traverses a course: start gate begins the clock, **progress gates** record split times along the way, end gate stops the clock and records the result. Players see their personal best (PB), compete on a global leaderboard sorted by best time, and can restart anytime. This card covers the full pattern: gate detection, timer state, splits, PB persistence, OrderedDataStore-backed leaderboards, restart mechanics, and a brief note on ghost replays. Builds on `world-mechanics.md` (trigger volumes), `datastores.md` + `community-libraries.md` (ProfileService for PBs), and `leaderstats.md` (visible best-time column).

## Gate types

```
Start gate     ──→  Progress gates 1, 2, 3, ...  ──→  End gate
(start timer)        (record split times)               (stop timer; record result)
```

All three are tagged BasePart trigger volumes:

```lua
-- In Studio: place a thin invisible Part across the path
gatePart.Anchored = true
gatePart.CanCollide = false       -- doesn't block movement
gatePart.CanQuery = false         -- doesn't show in raycasts
gatePart.Transparency = 1         -- invisible (or barely visible for debug)
gatePart.CanTouch = true

-- Tag determines role:
gatePart:AddTag("TimeTrialStart")     -- OR "TimeTrialProgress" / "TimeTrialEnd"
gatePart:SetAttribute("CourseId", "Forest")
gatePart:SetAttribute("ProgressIndex", 1)     -- progress gates only; ordered checkpoints
```

For visible gates (banner with course name, glowing line on the ground), use BillboardGui or `Beam` between two Attachments.

## Run state

Per-player, in-memory on the server:

```lua
type RunState = {
    courseId: string,
    startedAt: number,             -- os.clock() — high-resolution timer
    splits: {[number]: number},    -- progressIndex → seconds-since-start
    nextProgressIndex: number,     -- enforce ordered crossing
    crossedProgress: {[number]: boolean},   -- dedupe per-gate
}

local activeRuns: {[Player]: RunState} = {}
```

Use `os.clock()` (high precision, server-only) for the timer — NOT `os.time()` (1-second granularity). For persistence we'll record absolute *durations* in seconds; both clocks measure the same thing for "how long did the run take."

## Server — gate handlers

```lua
local CollectionService = game:GetService("CollectionService")
local Players = game:GetService("Players")

local function getPlayerFromHit(hit: BasePart): Player?
    local model = hit:FindFirstAncestorOfClass("Model")
    return model and Players:GetPlayerFromCharacter(model)
end

-- Start gate
local function setupStartGate(gate: BasePart)
    local courseId = gate:GetAttribute("CourseId") or "Default"
    gate.Touched:Connect(function(hit)
        local p = getPlayerFromHit(hit); if not p then return end

        -- Cancel any previous run on this course (player started over)
        activeRuns[p] = {
            courseId = courseId,
            startedAt = os.clock(),
            splits = {},
            nextProgressIndex = 1,
            crossedProgress = {},
        }
        TimeTrialRemote:FireClient(p, {kind = "Started", courseId = courseId, at = os.clock()})
    end)
end

-- Progress gate
local function setupProgressGate(gate: BasePart)
    local courseId = gate:GetAttribute("CourseId") or "Default"
    local progressIndex = gate:GetAttribute("ProgressIndex") or 0
    gate.Touched:Connect(function(hit)
        local p = getPlayerFromHit(hit); if not p then return end
        local run = activeRuns[p]
        if not run or run.courseId ~= courseId then return end
        if run.crossedProgress[progressIndex] then return end       -- already crossed this checkpoint
        if progressIndex ~= run.nextProgressIndex then return end   -- out of order; ignore

        local elapsed = os.clock() - run.startedAt
        run.splits[progressIndex] = elapsed
        run.crossedProgress[progressIndex] = true
        run.nextProgressIndex += 1

        TimeTrialRemote:FireClient(p, {kind = "Split", index = progressIndex, time = elapsed})
    end)
end

-- End gate
local function setupEndGate(gate: BasePart)
    local courseId = gate:GetAttribute("CourseId") or "Default"
    gate.Touched:Connect(function(hit)
        local p = getPlayerFromHit(hit); if not p then return end
        local run = activeRuns[p]
        if not run or run.courseId ~= courseId then return end

        local totalTime = os.clock() - run.startedAt

        -- Optional: validate all required progress gates were hit (no skipping)
        local requiredCount = countCourseProgressGates(courseId)
        if #run.crossedProgress < requiredCount then
            TimeTrialRemote:FireClient(p, {kind = "InvalidRun", reason = "Missed checkpoint"})
            activeRuns[p] = nil
            return
        end

        TimeTrialRemote:FireClient(p, {kind = "Finished", time = totalTime, splits = run.splits})
        recordTime(p, courseId, totalTime, run.splits)
        activeRuns[p] = nil
    end)
end

for _, g in CollectionService:GetTagged("TimeTrialStart") do setupStartGate(g) end
for _, g in CollectionService:GetTagged("TimeTrialProgress") do setupProgressGate(g) end
for _, g in CollectionService:GetTagged("TimeTrialEnd") do setupEndGate(g) end
CollectionService:GetInstanceAddedSignal("TimeTrialStart"):Connect(setupStartGate)
CollectionService:GetInstanceAddedSignal("TimeTrialProgress"):Connect(setupProgressGate)
CollectionService:GetInstanceAddedSignal("TimeTrialEnd"):Connect(setupEndGate)
```

The **out-of-order check** (`progressIndex == run.nextProgressIndex`) prevents shortcuts. Combined with the missed-checkpoint validation at the end gate, players must traverse the course as authored.

`countCourseProgressGates` counts tagged progress gates with the matching `CourseId` — cache this at server start; it doesn't change.

## Personal best (PB) tracking

```lua
local function recordTime(player: Player, courseId: string, time: number, splits)
    local profile = PlayerData.Get(player); if not profile then return end
    profile.timeTrials = profile.timeTrials or {}
    local prev = profile.timeTrials[courseId]

    local isPb = not prev or time < prev.bestTime

    profile.timeTrials[courseId] = {
        bestTime = isPb and time or prev.bestTime,
        bestSplits = isPb and splits or prev.bestSplits,
        runCount = (prev and prev.runCount or 0) + 1,
        lastTime = time,
        lastAt = os.time(),
    }

    -- Update leaderstats column
    local courseLeaderstat = player.leaderstats:FindFirstChild("Best_" .. courseId)
    if courseLeaderstat then
        courseLeaderstat.Value = formatTime(profile.timeTrials[courseId].bestTime)
    end

    -- Push to global leaderboard
    if isPb then submitToLeaderboard(player, courseId, time) end

    TimeTrialRemote:FireClient(player, {kind = "Result", time = time, isPb = isPb, prevBest = prev and prev.bestTime})
end

local function formatTime(seconds: number): string
    local mins = math.floor(seconds / 60)
    local secs = seconds % 60
    return string.format("%d:%05.2f", mins, secs)        -- "1:23.45"
end
```

## Global leaderboard (OrderedDataStore)

OrderedDataStore stores integer values; encode time-as-milliseconds (smaller is better, so we'll display ascending):

```lua
local DSS = game:GetService("DataStoreService")

local function getCourseLeaderboard(courseId: string)
    return DSS:GetOrderedDataStore("TimeTrial_" .. courseId .. "_v1")
end

local function submitToLeaderboard(player: Player, courseId: string, time: number)
    local board = getCourseLeaderboard(courseId)
    local timeMs = math.floor(time * 1000)
    pcall(function() board:SetAsync(tostring(player.UserId), timeMs) end)
end

local function getTopTimes(courseId: string, count: number?)
    count = count or 100
    local board = getCourseLeaderboard(courseId)
    local ok, pages = pcall(function() return board:GetSortedAsync(true, count) end)
    if not ok then return nil end
    local entries = {}
    for rank, e in pages:GetCurrentPage() do
        table.insert(entries, {
            rank = rank,
            userId = tonumber(e.key),
            timeMs = e.value,
            timeSec = e.value / 1000,
        })
    end
    return entries
end
```

`GetSortedAsync(true, count)` returns ascending — fastest first. For player names, look up via `Players:GetNameFromUserIdAsync(userId)` (cached server-side; rate-limited).

For richer leaderboards (best-times-this-week, best-by-friends), see [memorystores.md](memorystores.md) SortedMap (faster, ephemeral).

## Restart mid-run

If the player wants to retry a run in progress, just have them touch the start gate again — the existing handler resets `activeRuns[p]`. For an explicit "Restart" button, fire a remote that nukes the active run:

```lua
RestartTrialRemote.OnServerEvent:Connect(function(player)
    activeRuns[player] = nil
    TimeTrialRemote:FireClient(player, {kind = "Restarted"})
    -- Optionally teleport them back to the start gate
    local startGate = findStartGateFor(getCurrentCourse(player))
    if startGate and player.Character then
        player.Character:PivotTo(startGate.CFrame * CFrame.new(0, 5, 0))
    end
end)
```

For racing-style "auto-respawn at last checkpoint on death," store the last-crossed progress gate's CFrame and teleport there on `humanoid.Died`.

## Client UI — running timer + splits

The client renders a live timer based on `startedAt`. No need to push every frame; the client knows the start time and runs its own clock:

```lua
-- LocalScript
local startedAt: number? = nil
local timerLabel = ...    -- TextLabel in HUD

TimeTrialRemote.OnClientEvent:Connect(function(payload)
    if payload.kind == "Started" then
        startedAt = payload.at
    elseif payload.kind == "Finished" or payload.kind == "InvalidRun" or payload.kind == "Restarted" then
        startedAt = nil
        if payload.kind == "Finished" then
            showResultPanel(payload.time, payload.splits, payload.isPb, payload.prevBest)
        end
    elseif payload.kind == "Split" then
        flashSplit(payload.index, payload.time)
    end
end)

RunService.RenderStepped:Connect(function()
    if startedAt then
        local elapsed = os.clock() - startedAt    -- approximate; client clock isn't server clock
        timerLabel.Text = formatTime(elapsed)
    else
        timerLabel.Text = ""
    end
end)
```

**Important caveat**: `os.clock()` on client and server aren't the same. The client's local timer drifts slightly from the server's authoritative time over a long run (typically <0.1s drift over a few minutes — negligible for visual). The server's recorded time is the authoritative result.

For per-split feedback, the client receives `{kind = "Split", index, time}` from the server with the authoritative split — the UI shows "Checkpoint 2: 0:34.21 (-0:01.45 from PB)".

## Showing PB delta during run

If you saved the player's previous best splits, show running comparison:
```lua
-- "Sector 2 split: 0:34.21 (-0:01.45 vs PB)"
local prevSplits = profile.timeTrials[courseId] and profile.timeTrials[courseId].bestSplits
if prevSplits and prevSplits[index] then
    local delta = time - prevSplits[index]
    showSplitWithDelta(index, time, delta)
else
    showSplit(index, time)
end
```

## Ghost replay (advanced)

A ghost is a translucent copy of the player's previous run, replaying alongside the current attempt. Implementation outline:

1. **Record**: during the run, sample HumanoidRootPart CFrame every N frames (e.g., 10Hz) into an array; on finish, save with the PB.
2. **Replay**: on next attempt, spawn a transparent character clone; lerp its CFrame between samples timestamped to match the current run's elapsed time.

```lua
-- Recording (server)
local recordingInterval = 0.1
local recording: {{t: number, cf: CFrame}} = {}
local lastSample = 0
RunService.Heartbeat:Connect(function()
    if not activeRun then return end
    if os.clock() - lastSample < recordingInterval then return end
    lastSample = os.clock()
    local hrp = player.Character and player.Character:FindFirstChild("HumanoidRootPart")
    if hrp then table.insert(recording, {t = os.clock() - activeRun.startedAt, cf = hrp.CFrame}) end
end)
-- On finish, persist the recording with the PB

-- Replay (client)
local function replayGhost(samples, startTime)
    local ghost = createGhostModel()
    RunService.Heartbeat:Connect(function()
        local elapsed = os.clock() - startTime
        local i = findSampleIndex(samples, elapsed)
        if i and samples[i + 1] then
            local a, b = samples[i], samples[i + 1]
            local frac = (elapsed - a.t) / (b.t - a.t)
            ghost:PivotTo(a.cf:Lerp(b.cf, frac))
        end
    end)
end
```

Ghost data scales with run length and sample rate — at 10Hz, a 60-second run = 600 samples × ~28 bytes per CFrame = ~17KB. Fits in DataStore but watch the size cap.

For multi-player ghost-vs-ghost (race against the world record), pull the world record's recording from a shared DataStore key.

## Patterns

### Daily / weekly time challenges
Reset the leaderboard daily: use `OrderedDataStore` per UTC day (`"Time_Forest_2026-05-04"`). At UTC midnight, switch to a new key. See [time-and-weather.md](time-and-weather.md) for `scheduleDailyAt`.

### Multi-segment racing course
Long courses with many checkpoints benefit from "sector best" tracking — even if the player's overall PB wasn't beaten, they may have set a personal sector record. Save best-per-sector separately:
```lua
profile.timeTrials.Forest.bestSectorTimes = {[1] = 12.3, [2] = 18.7, ...}
```
The "theoretical best" = sum of sector PBs. Display it as the goal.

### Time-trial mode toggle
A course may be playable as a time trial OR free roam. Toggle via prompt or menu — when not in trial mode, gates do nothing. Use a `TimeTrialEnabled` per-player attribute.

### No-restart timed challenges
For "Daily Challenge: complete this course on first try without restart," disallow restart mid-run. Track `runsToday` in the profile; lock further runs after first attempt.

## Pitfalls

- **`os.time()` for the timer.** 1-second granularity ruins competitive timing. Use `os.clock()` (server) — it's high-precision (microsecond) and monotonic per server.
- **Server clock vs client clock drift.** Don't mix authoritative time (server) with client display time. Server's recorded time is truth; client's display is approximation.
- **Re-touching the start gate during a run cancels the run.** Intentional? Document it. If unintentional, add a "lock-after-start" cooldown.
- **Touched fires multiple times per crossing** because of multiple BaseParts on the player or physics ticks. Dedupe via `crossedProgress` table or a per-player-per-gate cooldown.
- **Players standing on a gate.** Touched fires repeatedly. Use trigger volumes that are thin (e.g., 0.5 stud thick) and require fast crossing, OR use a one-shot pattern (`crossedProgress`).
- **Ghost replay blocking new runs.** A ghost character interfering with collision. Set `CanCollide = false` and `CanQuery = false` on every part of the ghost.
- **OrderedDataStore key length / character set.** Use string `tostring(userId)` keys; avoid special characters. Set/get at most ~60/min/server (DataStore quotas).
- **No PB validation server-side.** A bad client could send "I finished in 0.001s" — but in this card the server measures `os.clock()` itself, so the client can't fake. *If* you ever pass time from client to server, validate.
- **Timer running while paused / AFK.** A player going AFK mid-run keeps the timer running. Some games auto-fail after 5 minutes of no checkpoint progress; cancel the run. Others let the timer run (no penalty for slow runs since they don't beat the PB).
- **Restart while server-restart pending.** A run started just before `BindToClose` may not record. Save active runs' state, OR just discard.
- **Course design with overlapping triggers.** Two progress gates close together may both fire on the same touch. Space gates at least one character-length apart.
- **Leaderboard showing offline players' bests but no name.** Cache `Players:GetNameFromUserIdAsync` results — it's rate-limited and yields. For a guaranteed name, persist `displayName` in the player's profile and look up there.
- **Ghost data persisted unencrypted in shared keys.** Anyone could deserialize and read another player's recording. Fine for racing games (it's just CFrames); concerning for stealth/exploit research.
- **Splits saved per attempt rather than per PB.** You only want to remember the best-runs splits; storing every attempt's splits balloons the profile.

## See also
- [world-mechanics.md](world-mechanics.md) — trigger volumes, Touched patterns
- [datastores.md](datastores.md) — saving PBs alongside profile
- [memorystores.md](memorystores.md) — alternative for ephemeral live-leaderboards
- [leaderstats.md](leaderstats.md) — best-time column in the top-right scoreboard
- [round-based.md](round-based.md) — for race-style time trials with simultaneous players, this card's gates can drive a round's win condition
- [vehicles.md](vehicles.md) — racing context where time trials are common
- [tags.md](tags.md), [attributes.md](attributes.md) — designer-driven gate setup
- [interactions.md](interactions.md) — alternative restart trigger (ProximityPrompt instead of touch)
- [ui-system.md](ui-system.md), [tweens-animation.md](tweens-animation.md) — running timer + result panel UI
- [live-ops.md](live-ops.md) — daily challenge resets via scheduleDailyAt
- [security.md](security.md) — server-authoritative timing
