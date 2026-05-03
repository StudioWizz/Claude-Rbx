# CLAUDE.md — Roblox Template

This is a Roblox Studio project. The user is exploring Roblox development with Claude as a pair-programmer connected directly to a running Roblox Studio session via MCP. Expect Luau scripts, `.rbxl`/`.rbxm` place files, and an in-Studio workflow rather than a traditional file-tree codebase.

When you start work in this repo, orient yourself by reading the project structure and the relevant `RbxDox/` reference cards (indexed below) before touching code or the live Studio.

## Roblox Studio MCP

A Roblox Studio instance is connected via MCP. You can read and mutate the live DataModel from this conversation. Standard workflow at the start of a session:

1. `mcp__Roblox_Studio__list_roblox_studios` — see what's connected.
2. `mcp__Roblox_Studio__set_active_studio` — pick one if more than one is open.
3. Use `execute_luau`, `inspect_instance`, `script_read`/`grep`/`search`, `multi_edit`, `screen_capture`, `start_stop_play`, etc. as needed.

For the full tool catalogue and usage patterns see **[RbxDox/studio-mcp.md](RbxDox/studio-mcp.md)**.

When you'd otherwise type Luau into the Studio command bar, use `execute_luau` instead. When you'd otherwise click around the Explorer and Properties panels, use `inspect_instance` and `search_game_tree`.

## Best-practice principles (the things that matter)

These are the cross-cutting rules distilled from the official Roblox docs. Apply them by default; deviate only with reason.

1. **The server is authoritative.** Never trust client-supplied data. Validate types, ranges, and ownership in every `OnServerEvent`. See [RbxDox/security.md](RbxDox/security.md).
2. **Place scripts correctly.** Server logic in `ServerScriptService`/`ServerStorage`; client logic in `StarterPlayerScripts`/`StarterCharacterScripts`/`StarterGui`; shared modules in `ReplicatedStorage`. See [RbxDox/project-structure.md](RbxDox/project-structure.md).
3. **Share code via `ModuleScript` + `require`.** No copy-pasting between `Script` and `LocalScript`. See [RbxDox/scripts-and-modules.md](RbxDox/scripts-and-modules.md).
4. **Prefer `task.wait` / `task.spawn` / `task.defer`** over the deprecated `wait` / `spawn` / `delay`. See [RbxDox/task-scheduling.md](RbxDox/task-scheduling.md).
5. **Use `RemoteEvent` for fire-and-forget** and `RemoteFunction` only when you genuinely need a return value (and never `:InvokeClient` if avoidable — a malicious client can hang the server thread). See [RbxDox/events-signals.md](RbxDox/events-signals.md).
6. **Use Attributes (and `CollectionService` tags) for designer-tunable values** instead of hard-coding constants in scripts. See [RbxDox/attributes.md](RbxDox/attributes.md).
7. **Type-check Luau** with `--!strict` (or at least `--!nonstrict`) for non-trivial modules. See [RbxDox/luau-types.md](RbxDox/luau-types.md).
8. **Profile before you optimize.** Use the MicroProfiler (`Ctrl/Cmd+F6`) to find real bottlenecks; pool reusable Instances; throttle remote traffic; consider Streaming for large worlds. See [RbxDox/performance.md](RbxDox/performance.md).
9. **Persist with `UpdateAsync`, not `SetAsync`.** Always `pcall` with retry/backoff. Save on `PlayerRemoving` and `BindToClose`, not in `Heartbeat`. Respect rate limits across DataStores, MemoryStores, MessagingService, HttpService. See [RbxDox/datastores.md](RbxDox/datastores.md).
10. **Set `:SetNetworkOwner(nil)` on game-critical physics** (projectiles, hazards, NPCs) so the client can't manipulate them. See [RbxDox/collisions.md](RbxDox/collisions.md) and [RbxDox/security.md](RbxDox/security.md).
11. **Filter player-entered text** with `TextService:FilterStringAsync` before displaying or saving. See [RbxDox/services.md](RbxDox/services.md).
12. **`:Destroy()` to release Instances** — `Parent = nil` alone leaves connections live and leaks memory.
13. **Use `:WaitForChild` on the client** for replicated children — they may not exist on the first frame.
14. **Use `RenderStepped` for client-side per-frame work** (camera, UI), `Heartbeat` for gameplay/physics ticks. Don't `while task.wait()` for either.
15. **Build then parent.** Construct Instance trees inside an unparented root, then assign `Parent` once — single replication step instead of N.

## RbxDox — concept reference index

Each file is a concise reference card: purpose, key APIs, idiomatic patterns, common pitfalls, links to upstream Roblox docs. Read the card relevant to the task before writing code; don't go to the live `create.roblox.com/docs` first — these cards already distill it.

### Foundations
| File | What it covers |
|---|---|
| [RbxDox/project-structure.md](RbxDox/project-structure.md) | DataModel hierarchy, services, where each script type lives, what replicates |
| [RbxDox/workspace-hierarchy.md](RbxDox/workspace-hierarchy.md) | Organising Workspace: Folder vs Model, parenting semantics, pivot, build-then-parent, Terrain |
| [RbxDox/client-server-model.md](RbxDox/client-server-model.md) | Server vs client VMs, replication, the trust boundary |
| [RbxDox/luau-basics.md](RbxDox/luau-basics.md) | Variables, tables, functions, control flow, operators, strings |
| [RbxDox/luau-types.md](RbxDox/luau-types.md) | `--!strict`, type annotations, narrowing, casts, generics |
| [RbxDox/luau-style.md](RbxDox/luau-style.md) | Casing rules (PascalCase vs camelCase), naming patterns, file conventions, idioms |
| [RbxDox/studio-environment.md](RbxDox/studio-environment.md) | Explorer, Properties, Output, Script Editor, testing modes |

### Scripting
| File | What it covers |
|---|---|
| [RbxDox/scripts-and-modules.md](RbxDox/scripts-and-modules.md) | `Script` vs `LocalScript` vs `ModuleScript`; `require`; class pattern |
| [RbxDox/oop-patterns.md](RbxDox/oop-patterns.md) | Class pattern with `__index`, `.new`, composition over inheritance, `:Destroy()`, forward declarations, state machines |
| [RbxDox/services.md](RbxDox/services.md) | `game:GetService` and the most-used services |
| [RbxDox/events-signals.md](RbxDox/events-signals.md) | RBXScriptSignal, `RemoteEvent`, `RemoteFunction`, `BindableEvent`, deferred behaviour |
| [RbxDox/attributes.md](RbxDox/attributes.md) | Designer-tunable typed values on Instances |
| [RbxDox/tags.md](RbxDox/tags.md) | `CollectionService` — tag-driven architecture, setup-on-add pattern |
| [RbxDox/task-scheduling.md](RbxDox/task-scheduling.md) | `task.*` library, `RunService` signals, parallel Luau Actors |
| [RbxDox/security.md](RbxDox/security.md) | Validation patterns, network ownership, exploiter defences, text filtering |

### 3D & Physics
| File | What it covers |
|---|---|
| [RbxDox/parts-and-models.md](RbxDox/parts-and-models.md) | `BasePart` properties, Models, `:PivotTo`, MeshPart, materials |
| [RbxDox/procedural-construction.md](RbxDox/procedural-construction.md) | What Claude can/can't build from parts, decision tree (procedural / Creator Store / generate_mesh), primitive library (walls, stairs, towers, arches), algorithmic organics (L-system trees, bushes, rocks, paths), iteration workflow with screen_capture |
| [RbxDox/cframes-vectors.md](RbxDox/cframes-vectors.md) | `Vector3` and `CFrame` math, common transformations |
| [RbxDox/raycasting.md](RbxDox/raycasting.md) | `:Raycast`, `RaycastParams`, overlap queries, shape-casts |
| [RbxDox/physics-constraints.md](RbxDox/physics-constraints.md) | Hinge / weld / spring / align constraints; mover vs mechanical |
| [RbxDox/collisions.md](RbxDox/collisions.md) | `CanCollide`/`CanQuery`/`CanTouch`, CollisionGroups, `Touched` |
| [RbxDox/vfx.md](RbxDox/vfx.md) | `ParticleEmitter`, `Beam`, `Trail`, `Smoke`/`Fire`, burst pattern, perf caps |
| [RbxDox/lighting.md](RbxDox/lighting.md) | Lighting service, `Atmosphere`, post-effects (Bloom, ColorCorrection, DOF) |
| [RbxDox/time-and-weather.md](RbxDox/time-and-weather.md) | Real time vs game time vs ClockTime, day/night cycle, weather state machine, seasons, scheduled real-time events |
| [RbxDox/world-mechanics.md](RbxDox/world-mechanics.md) | `SpawnLocation`, conveyor belts, moving platforms, jump pads, trigger volumes, tagged pickups |
| [RbxDox/areas-and-biomes.md](RbxDox/areas-and-biomes.md) | Defining areas (tagged volumes / Region3 / Models), region detection, per-area state (music/lighting/weather/enemy pool), entry transitions, discrete stage/level systems, when to split into separate places, boundaries, map / discovery |

### Player Experience
| File | What it covers |
|---|---|
| [RbxDox/characters.md](RbxDox/characters.md) | Humanoid, R6/R15, pathfinding, basic appearance, Animation Editor, custom locomotion (fly, sprint, dash, double-jump) |
| [RbxDox/player-lifecycle.md](RbxDox/player-lifecycle.md) | PlayerAdded/Removing, CharacterAdded/Removing, Died, Idled (built-in AFK), custom AFK tracking, reconnect, BindToClose |
| [RbxDox/spawn-mechanics.md](RbxDox/spawn-mechanics.md) | Spawn-point selection (random/team/smart/teammate), spawn protection (ForceField), pre-spawn UI, loadout-at-spawn, death-cam → respawn flow |
| [RbxDox/resource-bars.md](RbxDox/resource-bars.md) | Unified resource pattern (HP/stamina/mana/shield/XP/ammo/durability), regen rules, blocking flags, bar UI primitive (segmented + gradient), color thresholds, damage-flash ghost bar |
| [RbxDox/leaderstats.md](RbxDox/leaderstats.md) | `Player.leaderstats` Folder — top-right scoreboard, IntValue/StringValue children, formatting, sort key |
| [RbxDox/avatar-customization.md](RbxDox/avatar-customization.md) | `HumanoidDescription`, accessories, layered clothing, body colors, emotes (built-in + custom) |
| [RbxDox/tools.md](RbxDox/tools.md) | `Tool` class — equippable items, Handle/Grip, Equipped/Activated/Unequipped, server-authoritative damage |
| [RbxDox/vehicles.md](RbxDox/vehicles.md) | `Seat` and `VehicleSeat`, motorized HingeConstraints (wheels), suspension, network ownership for the driver |
| [RbxDox/interactions.md](RbxDox/interactions.md) | `ProximityPrompt` and `ClickDetector` — player→world interaction primitives |
| [RbxDox/input-handling.md](RbxDox/input-handling.md) | UserInputService, ContextActionService, mouse/keyboard/gamepad/touch |
| [RbxDox/camera.md](RbxDox/camera.md) | `Camera.CFrame`, CameraType, custom cameras, mouse picking |
| [RbxDox/ui-system.md](RbxDox/ui-system.md) | ScreenGui, Frame, UDim2, AnchorPoint, layouts, AutomaticSize |
| [RbxDox/menu-systems.md](RbxDox/menu-systems.md) | Menu architecture, MenuManager (single open + modal stack), tabs, transitions, hotkey toggles, settings persistence, confirmation dialogs, loading screens, tooltips, HUD vs menu, mobile UI |
| [RbxDox/tweens-animation.md](RbxDox/tweens-animation.md) | TweenService, easing, AnimationTrack on rigs |
| [RbxDox/sound.md](RbxDox/sound.md) | `Sound`, `SoundService`, sound groups, 2D vs 3D positional, parenting rules |
| [RbxDox/chat.md](RbxDox/chat.md) | `TextChatService` — channels, slash commands, `ShouldDeliverCallback`, replacing legacy Chat |

### Persistence & Networking
| File | What it covers |
|---|---|
| [RbxDox/datastores.md](RbxDox/datastores.md) | `UpdateAsync`, retry/backoff, session locking, OrderedDataStore, `BindToClose` |
| [RbxDox/save-slots.md](RbxDox/save-slots.md) | Multi-slot DataStore pattern, slot picker UI, active slot tracking, deletion, ProfileService integration |
| [RbxDox/offline-processing.md](RbxDox/offline-processing.md) | Idle accumulation, `os.time()` discipline, max-offline cap, daily login, prestige multipliers |
| [RbxDox/memorystores.md](RbxDox/memorystores.md) | HashMap / SortedMap / Queue, cross-server coordination |
| [RbxDox/messaging-http.md](RbxDox/messaging-http.md) | MessagingService pub/sub, HttpService outbound, secrets |
| [RbxDox/teleport.md](RbxDox/teleport.md) | `TeleportService` — `TeleportAsync`, `ReserveServer`, `TeleportData`, matchmaking handoff |
| [RbxDox/monetization.md](RbxDox/monetization.md) | `MarketplaceService` — gamepasses, devproducts, Premium, idempotent `ProcessReceipt` |

### Gameplay Systems
| File | What it covers |
|---|---|
| [RbxDox/npcs.md](RbxDox/npcs.md) | NPC composition (Model + Humanoid + prompt + script), friendly vs hostile, dialog (linear + branching), state machine, vendor and quest-giver integration |
| [RbxDox/enemy-spawning.md](RbxDox/enemy-spawning.md) | Spawner Instances, weighted pools, max-concurrent caps, activation/despawn radii, wave systems, group/pack spawning, boss spawners, time-of-day gating |
| [RbxDox/enemy-difficulty.md](RbxDox/enemy-difficulty.md) | Difficulty tiers (Easy/Normal/Hard/Nightmare), per-stat multipliers, per-area difficulty, player-count scaling, dynamic difficulty (DDA), endless scaling, loot/XP scaling per tier |
| [RbxDox/shops.md](RbxDox/shops.md) | Catalog shape, server-validated purchase flow, Robux + currency, vendor NPC, daily rotation, multi-tab, bulk purchase |
| [RbxDox/inventory.md](RbxDox/inventory.md) | Inventory data shape (separate from Backpack), stack counts, server-authoritative mutations, capacity, pickups, trading |
| [RbxDox/equipment.md](RbxDox/equipment.md) | Equipment slots (head/chest/weapon/etc.), stat aggregation, set bonuses, loadout presets, paper-doll UI |
| [RbxDox/crafting.md](RbxDox/crafting.md) | Recipe registry, ingredient validation, stations, craft timer, unlocks, queue, discovery |
| [RbxDox/progression.md](RbxDox/progression.md) | XP/level curve, level-up effects, attribute points, prestige/rebirth, multiplier-stack discipline |
| [RbxDox/skill-trees.md](RbxDox/skill-trees.md) | Tree data shape, point allocation, prerequisite gates, respec, stat aggregation from nodes |
| [RbxDox/combat.md](RbxDox/combat.md) | Hitbox vs hitscan, central damage function, damage types, combos, blocking/parry, knockback, stamina |
| [RbxDox/status-effects.md](RbxDox/status-effects.md) | Buffs/debuffs as time-limited modifiers, stack rules (Refresh/Stack/Independent), tick, stat aggregation, immunity |
| [RbxDox/quests.md](RbxDox/quests.md) | Quest definitions, per-player state, objective progression, quest-giver/turn-in NPCs, quest log UI, daily quests, achievements |
| [RbxDox/multi-stage-quests.md](RbxDox/multi-stage-quests.md) | Sequential stages, per-stage world state (`onEnter`/`onExit`), branching paths, dialog interludes, quest chains and sagas, mid-quest persistence, instanced vs shared world state |
| [RbxDox/time-trials.md](RbxDox/time-trials.md) | Start / progress / end gates, high-precision server timer, splits, personal best tracking, OrderedDataStore leaderboard, restart, ghost replay |
| [RbxDox/round-based.md](RbxDox/round-based.md) | Round state machine (Lobby → Countdown → Active → Ending), voting, mini-game registry, scoring, late-joiners, waves |
| [RbxDox/placement.md](RbxDox/placement.md) | Ghost preview, grid snap, server-validated placement, save/load layouts, tycoon dropper/collector, tower defense |
| [RbxDox/pets.md](RbxDox/pets.md) | Pet inventory, equip slots, follow behavior, multiplier stacking, hatching (RNG), fusion, auto-equip |
| [RbxDox/guns-fps.md](RbxDox/guns-fps.md) | Gun definition (ammo, reload, recoil, spread), ADS, hitscan vs projectile, kill feed, tracers |
| [RbxDox/horror-effects.md](RbxDox/horror-effects.md) | Sanity meter, monster proximity distortion, jumpscare framework, flashlight tool, ambient dread audio, hide mechanic |
| [RbxDox/trading-and-gifting.md](RbxDox/trading-and-gifting.md) | Mutual trade flow with confirmation latching (anti-scam), item gifting, Robux gifting via MarketplaceService cross-account purchase |
| [RbxDox/environmental-effects.md](RbxDox/environmental-effects.md) | Tag-driven effect blocks: kill/damage/heal/buff/coin/speed/float/invisibility/teleport/checkpoint, with one shared dispatcher |
| [RbxDox/vip-systems.md](RbxDox/vip-systems.md) | VIP gamepass detection (cached), VIP shop / area / discount / daily rewards / multi-tier (Bronze/Silver/Gold) / trial / visual flair |
| [RbxDox/player-badges.md](RbxDox/player-badges.md) | Above-head BillboardGui badges (VIP crown, Prestige stars, Level, Gifter, Owner/Admin), priority-sorted, MaxDistance perf, respawn cleanup |
| [RbxDox/growth-systems.md](RbxDox/growth-systems.md) | Plant lifecycle (seed → sprout → mature → fruiting), per-plot state, L-system stage builders, watering, harvest, persistence + offline accrual |
| [RbxDox/live-ops.md](RbxDox/live-ops.md) | Admin allowlist + commands, cross-server events via MessagingService, coordinator election, scheduled events, toast notifications, giveaways, live-config / Open Cloud control plane |

### Quality & Tools
| File | What it covers |
|---|---|
| [RbxDox/performance.md](RbxDox/performance.md) | MicroProfiler, common bottlenecks, design patterns, Streaming |
| [RbxDox/studio-mcp.md](RbxDox/studio-mcp.md) | Catalogue of Roblox Studio MCP tools and how to use them |
| [RbxDox/community-libraries.md](RbxDox/community-libraries.md) | Curated third-party libraries (ProfileService, Knit, Roact, Promise, Rojo, Wally, Cmdr, ZonePlus, etc.) |

## Routing — when asked to do X, read Y first

Grouped by topic for quick scanning. Genre shortcuts (obby, simulator, tycoon, RPG, etc.) are at the bottom.

### Architecture & code
| Task | Start with |
|---|---|
| Anything client ↔ server | `client-server-model.md` → `events-signals.md` → `security.md` |
| New module / shared code | `scripts-and-modules.md` → `luau-types.md` → `luau-style.md` |
| Class / object with state + behaviour | `oop-patterns.md` → `luau-types.md` |
| Naming, casing, or "what should I call this?" | `luau-style.md` |
| Performance issue | `performance.md` first, always |
| Tag-driven Instance setup (designer-tagged behaviour buckets) | `tags.md` → `attributes.md` |

### Studio + procedural construction
| Task | Start with |
|---|---|
| Spawn / move / rotate parts in the live Studio | `studio-mcp.md` → `parts-and-models.md` → `cframes-vectors.md` |
| Organise objects in the world (Folder vs Model, parenting) | `workspace-hierarchy.md` → `parts-and-models.md` |
| Build a structure (house, tower, fence, stairs, arch, simple geometry) | `procedural-construction.md` → `parts-and-models.md` → `studio-mcp.md` |
| Generate trees, rocks, bushes, organic decoration | `procedural-construction.md` (Algorithmic organics) — but **try Creator Store assets first** via `studio-mcp.md` |
| "Make a nice <thing>" with no clear spec | Ask user for size + materials + style references first; convert to spec; then `procedural-construction.md` (Iteration workflow) |

### UI / menus / HUD
| Task | Start with |
|---|---|
| Add a UI element | `ui-system.md` → `tweens-animation.md` → `studio-mcp.md` |
| Build a menu (main, pause, settings, inventory, shop, etc.) — system-level | `menu-systems.md` → `ui-system.md` → `input-handling.md` |
| Tab/page navigation, modal dialog, tooltip, loading screen | `menu-systems.md` |
| Settings UI (volume, FOV, key bindings, graphics) with persistence | `menu-systems.md` (Settings persistence) → `datastores.md` |
| HUD design (health bar, ammo, minimap layout) | `menu-systems.md` (HUD vs Menu) → `resource-bars.md` → topic cards (`leaderstats.md`, `guns-fps.md`, `status-effects.md`) |
| Top-right scoreboard / leaderstats / "Stage: 12 Wins: 3" | `leaderstats.md` |
| Health / stamina / mana / shield / XP bars and pools | `resource-bars.md` → `combat.md` (uses) → `progression.md` (XP) |
| Toast notifications, "player X did Y" broadcasts | `live-ops.md` (Notifications) → `ui-system.md` → `tweens-animation.md` |
| Above-head player badge (VIP crown, prestige star, level, role) | `player-badges.md` → `vip-systems.md` / `progression.md` (sources) |

### Player & character
| Task | Start with |
|---|---|
| Player lifecycle, AFK detection, reconnect handling | `player-lifecycle.md` → `datastores.md` (saves) → `community-libraries.md` (ProfileService) |
| Spawn flow (spawn points, protection, death-cam, respawn timer, loadout select) | `spawn-mechanics.md` → `player-lifecycle.md` → `world-mechanics.md` (SpawnLocation) |
| Hook up player input | `input-handling.md` → `events-signals.md` |
| Custom camera or cinematic | `camera.md` → `tweens-animation.md` |
| Custom locomotion (fly, sprint, dash, jetpack, double-jump, climb) | `characters.md` (Custom locomotion) → `physics-constraints.md` |
| Avatar customization, clothing, accessories, emotes | `avatar-customization.md` → `characters.md` |

### World, areas, environment
| Task | Start with |
|---|---|
| Lighting, atmosphere, post-effects | `lighting.md` |
| Day/night cycle, weather, seasons, in-world game clock | `time-and-weather.md` → `lighting.md` → `vfx.md` |
| Biomes, areas, regions, zones, world subdivision | `areas-and-biomes.md` → `tags.md` → `lighting.md` → `time-and-weather.md` |
| Discrete stages / levels (obby, level-based games) | `areas-and-biomes.md` (Discrete stages section) → `leaderstats.md` → `spawn-mechanics.md` |
| Per-area music / lighting / weather / enemy pool / spawn pool | `areas-and-biomes.md` (Per-area state) → `sound.md` / `lighting.md` / `time-and-weather.md` / `npcs.md` / `spawn-mechanics.md` |
| World map, minimap, area discovery / fog-of-war | `areas-and-biomes.md` (Map integration, Discovery) → `ui-system.md` |
| Spawn point, conveyor belt, moving platform, jump pad | `world-mechanics.md` → `collisions.md` → `tags.md` |
| Tagged pickup (item on the ground that grants on touch) | `world-mechanics.md` (Pickups section) → `tags.md` → `attributes.md` |
| Mining/farming/gathering/resource nodes | `world-mechanics.md` (Resource nodes) → `tags.md` → `inventory.md` |
| Kill block / damage tile / heal pad / coin pickup / speed pad / teleport pad / checkpoint | `environmental-effects.md` → `world-mechanics.md` → `tags.md` |
| Effect block (touch = float / invisible / status effect) | `environmental-effects.md` → `status-effects.md` |
| Selective barriers (enemies-only walls, faction doors, phantom units) | `collisions.md` (Selective barriers section) → `tags.md` |

### Items, inventory, monetization
| Task | Start with |
|---|---|
| Tool / equippable item (sword, gun, wand, fishing rod) | `tools.md` → `events-signals.md` → `security.md` |
| Player interaction with world (door, vendor, pickup, lever) | `interactions.md` → `events-signals.md` → `tweens-animation.md` |
| Inventory of items (consumables, materials, loot) | `inventory.md` → `datastores.md` |
| Equipment / gear slots / loadouts / set bonuses | `equipment.md` → `inventory.md` → `progression.md` |
| Crafting / recipes / workbench | `crafting.md` → `inventory.md` → `interactions.md` |
| Shop, store, vendor — buying with currency or Robux | `shops.md` → `monetization.md` (Robux) → `datastores.md` (currency) → `npcs.md` (vendor) |
| Robux purchases / gamepass / dev product / Premium | `monetization.md` → `security.md` (server-authoritative grant) |
| Player-to-player trading (mutual exchange) | `trading-and-gifting.md` → `inventory.md` → `security.md` |
| Gifting items or Robux to another player | `trading-and-gifting.md` (Gifting sections) → `monetization.md` (for Robux) |
| VIP system (shop, area, discount, daily reward, multi-tier) | `vip-systems.md` → `monetization.md` (gamepass primitive) → `shops.md` |
| Pets, companions, eggs, hatching | `pets.md` → `progression.md` (multiplier) |

### Combat, NPCs, enemies
| Task | Start with |
|---|---|
| Combat — damage, combos, blocking, knockback (PvP or PvE) | `combat.md` → `tools.md` → `security.md` |
| Buffs, debuffs, DoT, stun, slow, regen | `status-effects.md` → `combat.md` |
| Gun, rifle, shooter mechanics (ammo, reload, recoil, ADS) | `guns-fps.md` → `tools.md` → `combat.md` |
| Friendly NPC (vendor, quest-giver, dialog) | `npcs.md` → `interactions.md` → `characters.md` |
| Hostile NPC (enemy, mob, boss) | `npcs.md` → `characters.md` → `oop-patterns.md` (state machine) → `security.md` (network ownership) |
| Enemy spawn points / spawners / waves / max-concurrent caps | `enemy-spawning.md` → `npcs.md` → `tags.md` → `areas-and-biomes.md` |
| Enemy difficulty (Easy/Normal/Hard/Nightmare) / stat scaling / DDA | `enemy-difficulty.md` → `enemy-spawning.md` (apply on spawn) → `combat.md` (read multipliers) |

### Progression, quests, time trials
| Task | Start with |
|---|---|
| XP, levels, attribute points, rebirth/prestige, multiplier stack | `progression.md` → `leaderstats.md` |
| Skill tree / talent tree / branching upgrades | `skill-trees.md` → `progression.md` |
| Quest, mission, objective, achievement | `quests.md` → `npcs.md` (giver/turn-in) → `datastores.md` (persistence) → `tags.md` (tracked targets) |
| Multi-stage / multi-act questline with branching, world state per stage, cutscenes | `multi-stage-quests.md` → `quests.md` (foundation) → `npcs.md` |
| Dialog system (linear or branching, NPC conversations) | `npcs.md` (Dialog patterns) → `ui-system.md` |
| Time trial — start gate, checkpoints, end gate, leaderboard, personal best | `time-trials.md` → `world-mechanics.md` → `datastores.md` (OrderedDataStore) → `leaderstats.md` |

### Vehicles & assemblies
| Task | Start with |
|---|---|
| Vehicle (car, plane, boat, hovercraft) | `vehicles.md` → `physics-constraints.md` → `security.md` (network ownership) |
| Seat, chair, mount (player sits, doesn't drive) | `vehicles.md` (Seat section) |
| Placement (tycoon droppers, sandbox building, tower defense) | `placement.md` → `raycasting.md` → `inventory.md` |
| Particles / VFX / beams / trails | `vfx.md` → `parts-and-models.md` → `performance.md` |
| Play a sound | `sound.md` |

### Persistence & networking
| Task | Start with |
|---|---|
| Persist player data | `datastores.md` → `security.md` (consider `community-libraries.md` → ProfileService) |
| Multiple save slots / character slots per player | `save-slots.md` → `datastores.md` → `player-lifecycle.md` |
| Offline rewards / idle accumulation / passive income / daily login | `offline-processing.md` → `datastores.md` → `player-lifecycle.md` |
| Cross-server pool / leaderboard | `memorystores.md` (durable: `datastores.md`) |
| Cross-server announcement / webhook | `messaging-http.md` |
| Teleport between places / reserved server / matchmaking handoff | `teleport.md` → `memorystores.md` (if matchmaking) |

### Live-ops & operations
| Task | Start with |
|---|---|
| Round-based gameplay (party games, fighting arena, tower defense waves) | `round-based.md` → `oop-patterns.md` (FSM) |
| In-game chat or chat commands | `chat.md` → `security.md` (admin allowlists) |
| Admin commands (kick, give, ban, event-start) | `live-ops.md` → `chat.md` (TextChatCommand) → `security.md` (allowlist) |
| Cross-server / game-wide events, giveaways, coordinated boss spawns | `live-ops.md` → `messaging-http.md` → `memorystores.md` (coordinator lock) |
| Scheduled real-world events (daily reset, weekly raid, holiday window) | `time-and-weather.md` (scheduledDailyAt) → `live-ops.md` (cross-server fan-out) |
| Live config / hot-reloadable feature flags / remote control | `live-ops.md` (Live config) → `datastores.md` → `messaging-http.md` (Open Cloud) |

### Genre / domain shortcuts
| Genre / domain | Start with |
|---|---|
| Obby — checkpoints, stages, scoreboard | `world-mechanics.md` → `leaderstats.md` → `datastores.md` |
| Simulator / idle game (click-for-resource, prestige, pets, multipliers) | `progression.md` → `pets.md` → `leaderstats.md` → `offline-processing.md` |
| Tycoon (droppers, conveyors, buy-buttons, expanding base) | `placement.md` → `world-mechanics.md` → `interactions.md` → `leaderstats.md` |
| Farming / planting / growing crops / seeds → plants | `growth-systems.md` → `procedural-construction.md` (L-system) → `inventory.md` → `offline-processing.md` |
| Horror — sanity, jumpscares, flashlight, dread | `horror-effects.md` → `lighting.md` → `sound.md` |

## Style notes for code in this project

For full naming and casing rules see [RbxDox/luau-style.md](RbxDox/luau-style.md). Project-specific defaults:

- Prefer `--!strict` for ModuleScripts in `ReplicatedStorage`.
- Cache services at the top of every script: `local Players = game:GetService("Players")`.
- Use `:WaitForChild` for any replicated descendant access from a `LocalScript`.
- Keep `Workspace` clean: organise builds under named Folders/Models (PascalCase), not loose at the root.
- Name Remotes with their action: `Remotes.Damage`, `Remotes.Buy`, not `RemoteEvent1`.

## Updating this documentation

These reference cards intentionally distill the upstream docs rather than mirror them. If a Roblox API meaningfully changes (new constraint type, deprecated service, new MemoryStore quota), update the relevant card and check the upstream link still resolves. The upstream source for the official docs is <https://github.com/Roblox/creator-docs> under `content/en-us/`.
