---
kind: game
title: 'AI and save internals: the 101-state StateMachine, A* pathfinding, MapKnowledge, hordes, action states, PlayerDB and the nascent ECS'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); javap/jar-verified'
platform: windows
engine: java
route: loader-api
tools: ["javap (JDK)", "zombie.ai.* package map", "zombie.savefile.* PlayerDB"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Lua_(API)"
tags: [b42, ai, statemachine, pathfinding, mapknowledge, horde, actionstate, save, playerdb, ecs, internals]
---
# AI and save internals: StateMachine, pathfinding, MapKnowledge, PlayerDB, ECS (42.21.0)

> The last internals layer: how a PZ character actually thinks and persists. `zombie.ai` is a real
> hierarchical state machine (101 state classes) over A* pathfinding with learned map knowledge; saves go
> through per-player SQL DBs; and the "new B42 ECS" turns out to be a 6-class nascent system, not the
> big rewrite it looked like.

## Setup
- javap/jar-verified 42.21.0. Builds on the cheat-sheet note (Lua-callable surface) and both creature
  notes (mod-facing AI seams).

## Route and why
Loader API for behaviour; internals knowledge for knowing which seams exist.

## 1. `zombie.ai` - the state machine everything runs on
Jar-verified census:
- `StateMachine` (+ `SubstateSlot`) - hierarchical states with substates; every `IsoGameCharacter` owns
  one. `IsoZombie.initializeStates`/`actionStateChanged` are the zombie/player registration points.
- `zombie.ai.states` - **101 state classes**, player AND zombie AND animal sharing the framework:
  `IdleState`, `PathFindState`/`AnimalPathFindState`, `AttackState`/`LungeState`/`LungeNetworkState`/
  `AttackNetworkState`, `BumpedState`, `PlayerHitReactionState`/`ZombieHitReactionState`/
  `AnimalHitReactionState`/`PlayerHitReactionPVPState`, `FakeDeadZombieState`/`FakeDeadAttackState`,
  `BurntToDeath`, `ClimbOverFenceState`/`ClimbOverWallState`/`ClimbThroughWindowState`/
  `ClimbSheetRopeState`/`ClimbDownSheetRopeState`, `OpenWindowState`/`CloseWindowState`,
  `CollideWithWallState`, `CrawlingZombieTurnState`, `GrappledThrownIntoContainerState`/
  `GrappledThrownOutWindowState`/`GrappledThrownOverFenceState`, `PlayerAimState`/`PlayerActionsState`/
  `PlayerEmoteState`/`PlayerExtState`/`PlayerFallingState`/`PlayerFallDownState`/`PlayerGetUpState`/
  `PlayerKnockedDown`/`PlayerOnBedState`/`PlayerOnGroundState`/`PlayerDraggingCorpse`,
  `FishingState`/`FitnessState`, `ForecastBeatenPlayerState`, `GenericDefaultState`...
- `zombie.ai.astar` - `AStarPathFinder`/`AStarPathFinderResult`/`IPathfinder`: real A* over the tile
  graph. `pathToCharacter`/`pathToLocationF`/`pathToLocation`/`pathToTrough` are the Lua entry points.
- **`MapKnowledge` + `KnownBlockedEdges`** - zombies *learn the map*: they remember blocked edges
  (closed doors, fences) and repath around them instead of re-bumping. That's why vanilla zombies aren't
  wall-stuck; your custom creatures get it free on the zombie/animal chassis.
- **`ZombieGroupManager`** - horde/group membership; `VirtualZombieManager.createHordeFromTo` (cheat
  sheet) seeds groups through it. `WalkingOnTheSpot`, `AIBrainPlayerControlVars`.
- `zombie.ai.sadisticAIDirector` (3 classes) - the drama director (spawner pacing, event injection);
  `getNumVisible/ChasingZombies` stats (cheat sheet) are its instruments.
- The action layer beside it: `zombie.characters.action.ActionState`/`ActionStateContainer`/
  `ActionStateSnapshot`/`IActionStateChanged` - the "currently animating action" channel that
  `actionStateChanged` and `actionStateVariables` feed; TimedActions layer on top in Lua.

## 2. The mod-facing AI seam (what Lua can and can't do)
- **Can**: pick targets (`spotted`, `clearAggroList`), issue paths (`pathTo*`), set locomotion
  (`setWalkType`, `setSpeedTypeFromWalkType`, `setCanWalk`, `setCrawler`), fire state-relevant variables
  (`setVariable` into animsets), report semantic events (`reportEvent("wasBumped")`-family), hear/make
  noise (`RespondToSound`/`respondToSound`/`WorldSoundManager`), invert hostility (see beyond-chassis
  note), and intercept action changes via `OnZombieUpdate`-family reads of `actionStateChanged` +
  `IActionStateChanged` snapshots.
- **Can't (from Lua)**: add new `State` subclasses, change A* costs, edit the state machine itself -
  the machine is Java; your "new behaviours" are *choreographies of existing states* driven by a Lua
  controller (the COF/CDDA pattern), or full bypass (Lua-driven movement via `setX/Y` + own pathing,
  which loses collisions/MP - last resort).
- The practical taxonomy: **behaviour within a state** (attack clip choice, walk type, target) = Lua;
  **behaviour between states** (when to lunge, when to fake-death, when to flee) = Lua controller
  pushing state triggers; **the states themselves** = engine-fixed.

## 3. Saves and persistence internals
- `zombie.savefile`: **`PlayerDB`/`ClientPlayerDB`/`ServerPlayerDB`** - per-player SQL databases
  (world saves hold character state in SQLite, not flat files - `DBResult`/`DBTicket` join these to the
  network auth side), `SavefileNaming`, `SavefileThumbnail`, `AccountDBHelper`.
- Character/world persistence flow: `IsoZombie.save`/`load`, `IsoAnimal.save`/`load`/`unloaded`,
  `InventoryItem.getByteData`/`loadCorpseFromByteData` (corpse-as-item!), `IsoObject.getModData` +
  `transmitModdata`, `SGlobalObjectSystem` (Lua) for systems, `Lua`/mod world data under
  `Zomboid/Saves/<save>/`.
- Mod implication: anything you want kept must live in ModData (item/object/character/global-object
  systems) or your own file under the save dir - plain Lua globals die at save boundaries, and
  chunk-unload = `unloaded` call, not a save you can hook per-object.

## 4. The ECS that isn't (yet)
`zombie.characters.ecs` = `ECSComponent`, `ECSEntity`, + `componentmods/` (`ECSFrameStep`,
`ECSGameLoadingStateEnter`, `ECSInGameStateEnter`) - **6 classes total**. It's a small entity/component
scaffold wired to lifecycle events, not the full ECS rewrite "B42 ECS" rumours imply. `IsoZombie`
`registerECSComponents`-adjacent hooks exist; mod-facing ECS component authoring is NOT a thing yet -
file under "watch for 43", don't build on it.

## Gotchas
1. **States aren't Lua-registered** - a "new AI" is choreography over stock states, not new state classes;
   design around it (the CDDA/COF brains are proof it suffices).
2. **`MapKnowledge` is per-zombie** - recycled zombies may keep stale knowledge; another reason for the
   identity-clearing pattern.
3. **`ActionStateSnapshot`/`IActionStateChanged`** are the safe way to *observe* transitions - polling
   `getVariable` strings per tick is the fragile alternative.
4. **PlayerDB is SQL** - don't hand-edit save files; use `um backup` style whole-save snapshots.
5. **`unloaded` != saved-and-gone** - animals unload with chunks; identity must survive the round trip
   via `load`, not ModData marks alone.
6. **Multi-state choreography has a tick budget** - one `pathTo*` per think tick per creature; spamming
   repaths thrashes A* (the astar is real, and not free).
7. **`ZombieGroupManager` is Java-internal** - hordes from Lua go through `createHordeFromTo`; per-member
   group behaviour is yours to script.

## Verification
- javap/jar-verified 42.21.0: all class names/counts cited (101 states, 6 ECS classes, AI package list).
- Verified in the wild: state-machine choreography is exactly what COF/CDDA controllers do;
  `createHordeFromTo` exists in VZM.
- Not verified in-game: `MapKnowledge`/`ZombieGroupManager` Lua reachability (likely nil - Java-internal),
  PlayerDB schema, per-state entry conditions (decompile-answerable per state).

## Open questions
- Per-state entry/exit conditions and the full 101-state list semantics (each is a small decompile target).
- `AStarPathFinder` cost function (fences/windows/climbables weighting) - undocumented.
- ECS growth trajectory for 43+.
- `ForecastBeatenPlayerState` and other oddball states - vanilla uses them for hit-anticipation;
  mod-reusable?
