# RobloxTemplate

**Created by RevengerTT aka StudioWizz**

A curated, AI-readable knowledge library for building Roblox experiences. Designed to be loaded into a Claude Code session (or any AI coding assistant) so it can produce idiomatic Roblox code that follows current best practices instead of patterns from older training data.

## What this is

- **`CLAUDE.md`** — root project file. Auto-loaded by Claude Code on every session in this directory. Contains the project's best-practice principles and a routing table mapping "do X" tasks to the relevant reference cards.
- **`RbxDox/`** — 77 reference cards covering the construction surface for typical Roblox games. Each card is a focused 70–600 line guide on one concept, with code examples, idiomatic patterns, common pitfalls, and links to the upstream Roblox Creator Documentation it distills.

Together they let a fresh AI session produce code that:
- Lives in the right place (correct service, script type, Instance class)
- Uses the right APIs (modern `task.*` over deprecated `wait`/`spawn`; `UpdateAsync` over `SetAsync`; `LinearVelocity` constraints over `BodyVelocity`; `TextChatService` over legacy `Chat`)
- Follows server-authoritative discipline by default (validates every `OnServerEvent`, never trusts client claims)
- Composes cleanly with the rest of the project's systems

## Quick start

### For Claude / AI sessions
`CLAUDE.md` is auto-loaded. Read it first; follow the routing table to find the relevant reference card; cross-link as needed. The cards reference each other; following the chain usually surfaces every primitive the task needs.

### For humans
Open `CLAUDE.md` for the high-level overview, best-practice principles, and the index of all cards. Then drill into individual `RbxDox/<topic>.md` files as you need them. The routing table at the bottom of `CLAUDE.md` maps common tasks to their starting card.

## Structure

```
RobloxTemplate/
├── CLAUDE.md              # Root context — best practices, card index, routing table
├── README.md              # This file
├── VALIDATION.md          # Original validation report (snapshot, mid-build)
└── RbxDox/                # 77 reference cards
    ├── project-structure.md          # Foundations: DataModel, services, where things live
    ├── client-server-model.md
    ├── luau-basics.md                # Language: variables, types, closures, GC, errors
    ├── luau-types.md
    ├── luau-style.md                 # Naming, casing, formatting, idioms
    ├── studio-environment.md
    ├── workspace-hierarchy.md
    ├── scripts-and-modules.md        # Scripting fundamentals
    ├── oop-patterns.md
    ├── services.md
    ├── events-signals.md
    ├── attributes.md
    ├── tags.md
    ├── task-scheduling.md
    ├── security.md
    ├── parts-and-models.md           # 3D & physics
    ├── cframes-vectors.md
    ├── raycasting.md
    ├── physics-constraints.md
    ├── collisions.md
    ├── vfx.md
    ├── lighting.md
    ├── time-and-weather.md
    ├── world-mechanics.md
    ├── areas-and-biomes.md
    ├── procedural-construction.md
    ├── characters.md                 # Player & character
    ├── player-lifecycle.md
    ├── spawn-mechanics.md
    ├── resource-bars.md
    ├── leaderstats.md
    ├── avatar-customization.md
    ├── tools.md
    ├── vehicles.md
    ├── interactions.md
    ├── input-handling.md
    ├── camera.md
    ├── ui-system.md
    ├── menu-systems.md
    ├── tweens-animation.md
    ├── sound.md
    ├── chat.md
    ├── datastores.md                 # Persistence & networking
    ├── save-slots.md
    ├── offline-processing.md
    ├── memorystores.md
    ├── messaging-http.md
    ├── teleport.md
    ├── monetization.md
    ├── npcs.md                       # Gameplay systems
    ├── enemy-spawning.md
    ├── enemy-difficulty.md
    ├── combat.md
    ├── status-effects.md
    ├── guns-fps.md
    ├── horror-effects.md
    ├── inventory.md
    ├── equipment.md
    ├── crafting.md
    ├── shops.md
    ├── trading-and-gifting.md
    ├── environmental-effects.md
    ├── progression.md
    ├── skill-trees.md
    ├── quests.md
    ├── multi-stage-quests.md
    ├── time-trials.md
    ├── round-based.md
    ├── placement.md
    ├── pets.md
    ├── growth-systems.md
    ├── vip-systems.md
    ├── player-badges.md
    ├── live-ops.md
    ├── performance.md                # Quality & tools
    ├── studio-mcp.md
    └── community-libraries.md
```

## What's covered

| Area | Cards | What you can build |
|---|---|---|
| **Foundations** | 7 | Where scripts live; client/server model; Luau language + types + style; Studio environment; Workspace organisation |
| **Scripting** | 8 | Modules + classes; service caching; events + remotes; attributes; tags; task scheduling; OOP; security validation |
| **3D & physics** | 11 | Parts/models; CFrame math; raycasting; constraints; collisions (incl. selective barriers); VFX; lighting; weather; biomes/areas; procedural construction |
| **Player & character** | 6 | Humanoid; lifecycle (PlayerAdded/CharacterAdded/Died/Idled); spawn mechanics; resource bars (HP/stamina/mana); leaderstats; avatar customization |
| **Items & UI** | 9 | Tools; vehicles; interactions (ProximityPrompt); input; camera; UI primitives + menu systems; tweens; sound; chat |
| **Persistence & networking** | 7 | DataStores (with ProfileService); save slots; offline accrual; MemoryStore; MessagingService + HTTP; TeleportService; monetization |
| **Gameplay systems** | 22 | NPCs (friendly + hostile); enemy spawning + difficulty; combat + status effects; guns/FPS; horror effects; inventory; equipment; crafting; shops; trading + gifting; environmental effect blocks; progression + skill trees; quests + multi-stage quests; time trials; round-based; placement; pets; growth systems; VIP; player badges; live-ops |
| **Quality & tools** | 3 | Performance + MicroProfiler; Roblox Studio MCP catalogue; curated community libraries |

Genre coverage: obby, simulator, tycoon, RPG, FPS, fighting, racing, survival, horror, party/mini-game, sandbox, tower defense, pet games, farming sims.

## How to use it (workflow)

1. **Open Claude Code in this directory.** `CLAUDE.md` is auto-loaded; the routing table tells Claude which cards to read for the task at hand.
2. **Connect a Roblox Studio session** via the [Roblox Studio MCP](https://create.roblox.com/docs/studio/mcp). The `studio-mcp.md` card documents the tool catalogue.
3. **Tell Claude what you want to build.** It should route to the relevant cards, read them, and produce code that follows the documented patterns. If it doesn't, the routing table or that card needs improvement — it's a doc bug, not an AI bug.
4. **Iterate.** When the AI produces something wrong, that's signal: usually the wrong card was consulted, or a card has a gap. The fix is to update the card or the routing table.

## Updating & extending

The library is meant to grow with the project. To add a new card:

1. Author it as `RbxDox/<topic>.md` following the same template (`Purpose` → `Key APIs / Patterns` → `Pitfalls` → `See also`).
2. Add it to the appropriate section in `CLAUDE.md`'s index table.
3. Add a routing entry under the right group (Architecture, UI, World, Gameplay, etc.).
4. Cross-link from related cards in their `See also` sections.
5. Verify with `python3` checking that all markdown links resolve and the new card is indexed.

To update an existing card when Roblox APIs change: edit the card, verify the upstream link still resolves (the source-of-truth is `https://github.com/Roblox/creator-docs` under `content/en-us/`), and update any cross-references that depend on the changed behaviour.

## Sources & attribution

The cards are distilled from:
- **Official Roblox Creator Documentation** — open source at [`github.com/Roblox/creator-docs`](https://github.com/Roblox/creator-docs) (CC-BY-4.0). Each card cites the specific upstream pages it draws from.
- **Roblox Lua Style Guide** — [`roblox.github.io/lua-style-guide`](https://roblox.github.io/lua-style-guide/). The casing, formatting, and naming rules in `luau-style.md` follow this.
- **Community open-source projects** — ProfileService, Knit, Roact, RbxUtil, FastCastRedux, ZonePlus, etc. Curated and described in `community-libraries.md`. Each library's licence applies; check before using.
- **Common idioms across Roblox open-source projects** — patterns observed across many production Roblox experiences.

The cards are derived works of the upstream documentation; they're not verbatim copies, and they intentionally compress, opinionate, and Roblox-specific-ize the source material for the use case of "feeding to an AI assistant."

## License

This documentation library is the user's project. Review `LICENSE` file. The reference cards are derived from CC-BY-4.0 source material; if you republish, attribute the upstream Roblox Creator Documentation accordingly.

All I ask is that if you make any improvements or spot any issues, please link/fork or message me to let me know and help share the knowledge!

## Stats

- **77 reference cards** + `CLAUDE.md` + this `README.md`
- **~19,750 lines** of structured documentation
- **0 broken markdown links** (verified)
- All 15 best-practice principles reinforced across the relevant topic cards
- All cards indexed in `CLAUDE.md`; all routing entries resolve
