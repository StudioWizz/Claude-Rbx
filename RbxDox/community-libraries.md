# Community Libraries & Tools

> Sources: <https://devforum.roblox.com/c/resources/14> · curated from widely-adopted Roblox open-source ecosystem (ProfileService, Knit, Roact, Promise, Rojo, etc.)

## Purpose
Roblox has a healthy open-source ecosystem. For several common problems — session-safe DataStores, async/await, in-experience admin commands, declarative UI, on-disk project layout — there's a battle-tested library that solves it better than rolling your own. This card is a deliberately short, opinionated list: only libraries that are widely deployed in production, actively maintained (or stable enough not to need maintenance), and solve a problem our other RbxDox cards explicitly call out.

For everything else, the unfiltered list is at <https://devforum.roblox.com/c/resources/14>. Read licenses before using anything.

## Persistence

### ProfileService — `loleris/ProfileService`
The default answer for player data. Solves the session-lock problem ([datastores.md](datastores.md) calls it out) so the same player can never have two servers writing concurrently. Built-in retry, GDPR support, profile reconciliation against a default schema.
<https://madstudioroblox.github.io/ProfileService>

Use ProfileService unless you have a specific reason not to. The naive `GetAsync` / `UpdateAsync` pattern in [datastores.md](datastores.md) is correct as far as it goes, but ProfileService handles the cross-server edge cases you'll otherwise re-discover the hard way.

### ReplicaService — `loleris/ReplicaService`
Selective server→client state replication, designed to pair with ProfileService. Replaces hand-rolled "RemoteEvent fire on every state change" patterns.
<https://madstudioroblox.github.io/ReplicaService>

### DataStore2 — `Kampfkarren/Roblox`
Older alternative to ProfileService, with built-in caching and historical backups. Less common in new code today; you'll see it in older codebases.
<https://kampfkarren.github.io/Roblox/>

## Async patterns

### roblox-lua-promise — `evaera/roblox-lua-promise`
A `Promise/A+`-style implementation. Use when you have asynchronous code with composition needs (chain multiple `Async` calls, race them, retry, cancel). For one-off awaits, plain `task.spawn` + `pcall` is simpler.
<https://eryn.io/roblox-lua-promise/>

```lua
local Promise = require(ReplicatedStorage.Promise)

Promise.new(function(resolve, reject)
    local ok, data = pcall(function() return store:GetAsync(key) end)
    if ok then resolve(data) else reject(data) end
end)
:andThen(function(data) print("got", data) end)
:catch(function(err) warn(err) end)
:timeout(5)
```

## Validation

### t — `osyrisrblx/t`
Runtime type checker. Luau's `--!strict` catches mismatches at compile time, but anything crossing the network boundary (`OnServerEvent` payloads, `HttpService` responses, `DataStore` reads of legacy data) arrives as `unknown` at runtime — the type checker can't help. `t` is the standard library for validating that an `unknown` actually matches an expected shape, returning a boolean (or asserting).
<https://github.com/osyrisrblx/t>

```lua
local t = require(ReplicatedStorage.Packages.t)

local damagePayload = t.strictInterface({
    targetId = t.integer,
    amount = t.numberConstrained(0, 100),
    weapon = t.optional(t.string),
})

remote.OnServerEvent:Connect(function(player, payload)
    if not damagePayload(payload) then return end   -- malformed → drop
    -- payload now safe to use with the validated shape
end)
```

`t` provides primitive checks (`t.string`, `t.number`, `t.boolean`, `t.Instance`, every Roblox type), combinators (`t.union`, `t.intersection`, `t.optional`, `t.array`, `t.map`, `t.tuple`), and shape checks (`t.interface`, `t.strictInterface`, `t.instanceOf`). Pairs naturally with the validation discipline in [security.md](security.md).

## Frameworks

### Knit — `sleitnick/Knit`
Lightweight game framework: services on the server, controllers on the client, automatic networking between them. Imposes structure on a project without locking you into a particular paradigm. The most popular framework in the Roblox community.
<https://sleitnick.github.io/Knit/>

Useful when you'd otherwise be hand-rolling service-locator and remote-routing boilerplate. Skippable for small projects.

### RbxUtil — `sleitnick/RbxUtil`
Sleitnick's collection of utility modules (Signal, Trove, Component, Promise, etc.) that Knit composes from. Pick individual modules even if you don't use the framework.
<https://sleitnick.github.io/RbxUtil/>

The **Trove** / **Janitor** pattern is particularly useful — it gives you one object you add cleanup tasks to and `:Destroy()` once, instead of tracking N connections by hand. Solves the leak class described in [oop-patterns.md](oop-patterns.md).

## UI

### Roact (and React-Lua) — `Roblox/roact`, `jsdotlua/react-lua`
Declarative UI library, React-shaped. Components own state, re-render on change, the diff updates the actual `GuiObject` tree. Roblox's own first-party UI work uses React-Lua now.
<https://roblox.github.io/roact/>

Worth adopting when your UI has dynamic state that's painful to manage imperatively (lots of conditional visibility, lists that grow/shrink, screens that mirror remote data). For a static HUD, plain `ScreenGui` from [ui-system.md](ui-system.md) is fine.

### UI Labs — plugin
Storybook-style component preview inside Studio. Pair with Roact/React-Lua for component-driven UI development.

## Spatial / gameplay

### ZonePlus — `ForeverHD/ZonePlus`
High-level "this player entered/left this volume" detection built on `OverlapParams`. Replaces the hand-rolled per-Heartbeat overlap loops shown in [collisions.md](collisions.md).
<https://1foreverhd.github.io/ZonePlus/>

### TopbarPlus — `ForeverHD/TopbarPlus`
Topbar-icon framework with sane defaults for menus, settings, etc. The Roblox topbar is awkward to drive directly; this hides the awkwardness.
<https://1foreverhd.github.io/TopbarPlus/>

## Networking

### Warp — `imezx/Warp`
High-throughput networking library that bundles many small remote calls into single packets per frame. Useful when [performance.md](performance.md)'s "throttle remotes" advice isn't enough — typically combat-heavy or simulation-heavy games.
<https://imezx.github.io/Warp/>

For ordinary projects, the engine's `RemoteEvent` and `UnreliableRemoteEvent` (see [events-signals.md](events-signals.md)) are sufficient.

## Tooling

### Rojo — `rojo-rbx/rojo`
The standard for professional Roblox development. Roblox source lives on disk as `.lua` / `.luau` files in a Git repo; Rojo syncs them into a running Studio session in real time. Unlocks proper version control, code review, and external editors (VS Code with the Luau extension).
<https://rojo.space/>

File naming convention (already in [luau-style.md](luau-style.md)):
- `Foo.luau` → `ModuleScript`
- `Foo.server.luau` → `Script`
- `Foo.client.luau` → `LocalScript`
- `init.luau` / `init.server.luau` / `init.client.luau` → the script *becomes* its parent folder

`default.project.json` defines the DataModel mapping. Once set up, Studio is just a runtime — all editing happens in your editor of choice.

### Wally — `UpliftGames/wally`
Package manager for Luau, modeled after Cargo/npm. Pairs naturally with Rojo for installing libraries (ProfileService, Knit, Roact, etc.) into a `Packages/` folder rather than copy-pasting `.rbxm` files.
<https://wally.run/>

### Cmdr — `evaera/Cmdr`
Extensible in-experience command console. Drop in for admin commands, debug toggles, dev cheats. Saves you writing yet another `chat:Connect → parse → dispatch` system.
<https://eryn.io/Cmdr/>

### DataStore Editor — Studio plugin (sleitnick, paid; Free DataStore Editor by Xsticcy as alternative)
View and edit live DataStores from Studio. Indispensable for debugging a stuck player save without writing throwaway scripts.

### Open Cloud — official Roblox HTTP API
Roblox-hosted REST API for editing DataStores, sending MessagingService messages, publishing places, and more — from outside Roblox (your CI, a backend service, an admin tool). Combine with the **OpenCloudTools** desktop app or `curl` from your shell.
<https://create.roblox.com/docs/cloud/open-cloud>

## Tutorials worth bookmarking

- **Roblox Lua Style Guide** — <https://roblox.github.io/lua-style-guide/> — the convention [luau-style.md](luau-style.md) is anchored to.
- **Luau language reference** — <https://luau.org/> — definitive source for syntax, type system, and standard library.
- **Sleitnick's YouTube channel** — <https://www.youtube.com/@sleitnick> — careful, well-explained framework / architecture videos.
- **TheDevKing** — beginner-friendly scripting walkthroughs.

## Selection guidance

Pull in a community library when:
- It solves a problem our docs explicitly say is hard (session-locked DataStores, async composition, declarative UI).
- It's actively used in many open-source Roblox projects.
- The license suits your project (most are MIT or similar).

Roll your own when:
- The library is heavier than the problem you're solving (e.g. Roact for a 2-button HUD).
- You'd be the only user — abandoned-library risk.
- You need very specific behaviour the library doesn't expose.

## Pitfalls

- **`require`-by-AssetId for "free models."** Free models can ship with malicious code (backdoors, IP loggers). Vendor the source instead — pull from the GitHub repo, audit, commit into your own project.
- **Mixing competing solutions.** ProfileService AND DataStore2 in the same project is two ways to do the same thing — pick one.
- **Adopting a framework mid-project.** Knit / Roact restructure how the whole project works. Cheap on day 1, expensive on day 100.
- **Outdated copies.** A library you copied two years ago has bugs that are fixed upstream. Pin a version (Wally helps), and re-check periodically.
- **Skipping the license check.** Most Roblox community libraries are MIT, but not all. Verify before shipping.

## See also
- [datastores.md](datastores.md) — the underlying API that ProfileService wraps
- [ui-system.md](ui-system.md) — what Roact/React-Lua sit on top of
- [scripts-and-modules.md](scripts-and-modules.md) — module require patterns Rojo and Wally produce
- [events-signals.md](events-signals.md) — what Warp optimises
- [oop-patterns.md](oop-patterns.md) — Trove/Janitor pattern for cleanup
- [collisions.md](collisions.md) — ZonePlus replaces hand-rolled overlap loops
- [chat.md](chat.md), [monetization.md](monetization.md) — Cmdr / ProfileService common pairings
