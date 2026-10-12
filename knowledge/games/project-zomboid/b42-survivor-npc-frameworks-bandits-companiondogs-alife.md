---
kind: game
title: 'Survivor NPC frameworks in the wild: how Bandits programs zombies as people, CompanionDogs rides the raccoon chassis, and ProjectALife modularizes NPCs'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); mods: Bandits (workshop 3268487204), CompanionDogs (3740052292), ProjectALifeNPCs (3803984183)'
platform: windows
engine: java
route: loader-api
tools: ["Bandits workshop mod (3268487204)", "CompanionDogs (3740052292)", "ProjectALifeNPCs (3803984183)"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://steamcommunity.com/sharedfiles/filedetails/?id=3268487204"
- "https://steamcommunity.com/sharedfiles/filedetails/?id=3740052292"
- "https://steamcommunity.com/sharedfiles/filedetails/?id=3803984183"
tags: [b42, npc, survivors, bandits, companion, ai-framework, zombie-chassis, raccoon, tasks, programs, interop]
---
# Survivor NPC frameworks in the wild: Bandits, CompanionDogs, ProjectALife (42.21.0)

> The beyond-chassis note theorized "a zombie with inverted AI = a friendly NPC". Three shipped mods
> prove and far exceed it: **Bandits** runs full survivor NPCs on zombies via a task/program framework
> (49 action modules, 9 behavior programs, bases, looting, fishing, generators), **CompanionDogs**
> builds a pet on the raccoon animal chassis, and **ProjectALife** shows modular NPC architecture with
> per-mod compat adapters. This note is the anatomy of the real thing.

## Setup
- Inspected workshop items `3268487204` (Bandits, 265 Lua/523 txt+xml, `42.20`+`42.21`+`common`
  version dirs), `3740052292` (CompanionDogs, 113 Lua), `3803984183` (ProjectALifeNPCs, 248 Lua/695
  txt+xml, `alife_runtime/`). Builds on the beyond-chassis + cheat-sheet notes.

## Route and why
Loader API - all three ship pure Lua+data (plus assets); no Java patching.

## 1. Bandits: the "programmed person on a zombie" framework
- **Identity**: `zombie:setVariableBoolean("Bandit", true)` - not an outfit check. The identity layer
  is `BanditZombie.Cache` (full objects) + `CacheLight`/`CacheLightB`/`CacheLightZ` (position/angle/
  isBandit/brain/rid only) - a perf-sorted cache split zombies vs bandits, rebuilt on unload via
  `IsReanimatedForGrappleOnly`/`isRagdoll`/`isAlive` early-outs in `OnZombieUpdate`.
- **Brain**: `BanditBrain.Get(zombie)` - one persistent brain table per bandit (the memory/goals/tasks
  store) looked up by `BanditUtils.GetZombieID`.
- **Actions** (`shared/ZombieActions/`, 49 modules): a uniform task API -
  `ZombieActions.X.onStart/onWorking/onComplete(zombie, task)` where `task` is a plain table. The
  library covers a whole survivor life: `ZAGoTo`/`ZAMove`/`ZAClimbFence`, `ZAAim`/`ZAShoot`-family,
  `ZAEquip`/`ZAEquipWeapon`/`ZALoad`/`ZARack` (gun handling!), `ZABandage`, `ZALootItems`/
  `ZALootWeapons`/`ZAPickUp`/`ZAPickUpBody`/`ZADrop`/`ZAPlaceItem`/`ZAPutInContainer`, `ZABuildFloor`,
  `ZABuryCorpse`/`ZAFillGrave`/`ZACleanBlood`, `ZAFishing`, `ZAGeneratorFix`/`ZAGeneratorRefill`/
  `ZAGeneratorToggle`, `ZAOpenWindow`/`ZALightToggle`, `ZADestroy`, `ZADie`, `ZAPush`, `ZASmack`,
  `ZADie`... - the NPC "verbs" are scripted action objects, one file each.
- **Programs** (`shared/ZombiePrograms/`): state machines that *emit task lists* - each state function
  returns `{status=..., next="State", tasks={...}}`. Shipped programs: `ZPBandit`, `ZPCamper`,
  `ZPCompanion`, `ZPCompanionGuard`, `ZPDead`, `ZPDefend`, `ZPLooter`, `ZPRoadblock`, `ZPThief`.
  `ZPCompanion.Main` is the readable example: checks `BanditPost.At(bandit,"guard")` -> switches
  program, `BanditPlayer.GetMasterPlayer` -> follow behavior, picks `walkType` (Walk/Run/SneakWalk/
  WalkAim) from master state + distance, tracks `endurance` deltas.
- **Locomotion trick**: `ZAGoTo` does `zombie:setVariable("BanditWalkType", ...)` +
  `zombie:setBumpType("IdleToRun"|"IdleToWalk")` + `pathToLocationF` + `faceLocationF` - the
  `setBumpType`/`BumpDone` anim-variable channel is how a *zombie chassis plays humanoid locomotion*
  (their custom actiongroups/AnimSets map those bump types to survivor anims). This is the key
  discovery: you don't need the zombie's own walk states - drive the player-style anim variables.
- **Ownership**: `BanditUtils.IsController(zombie)` gates `pathToLocationF` - only the owning client
  steers; others receive the sim (the popman ownership model applied correctly).
- **Peripheral systems**: `BanditPost` (guardpost/bases via world markers), `BanditBases`,
  `BanditWeapons` (armament/ammo bookkeeping incl. `IsOutOfAmmo`), `BanditLoot`, `BanditNames`,
  `BanditNotifications`, `BanditEventMarkers`, `PlayerDamageModel`, `TimedActions/`, `BanditModOptions`
  (PZAPI.ModOptions), `BanditCompatibility` (IsReanimated checks + version gating per `42.20`/`42.21`
  folders).
- **Client structure**: `BanditUpdate` (the tick), `BanditActionInterceptor`, `BanditClientCommands`,
  `BanditMenu`, `BanditPermanent`, `BanditPlayer(Base)`, `BanditProjectile`, `BanditRandom`,
  `BanditSquareLoader`, `BanditTest` - note the `isServer()` early-return in `OnZombieUpdate`: brains
  run on clients, synced via commands.

## 2. CompanionDogs: the animal-chassis companion
- Chassis: **the raccoon** - `media/AnimSets/raccoon/` gets merged animNodes (`cdNetRun`,
  `cdNetWalk`, `cdLimpBack`, `cdLimpFront`, `cdLimpFrontRun`, `runPathfind`, `defaultPathfindGlide`...)
  - custom dog anims merged into the host animset with the `cd` prefix convention, exactly the
  dressed-zombie pattern applied to animal AnimSets.
- UI layer: `CompanionDogs_RadialCommands` (ISRadialMenu!), `CompanionDogs_StatusUI`,
  `CompanionDogs_StrayScanUI` (a scan/stray-tracking window), `CompanionDogs_BagWindow` (pack
  inventory - the "pack mule" pattern from the beyond-chassis note realized),
  `CompanionDogs_ContextMenu`.
- Takeaway: a companion needs (a) the animal brain driving follow/guard via behavior calls, (b) a
  command UI (radial), (c) status/bag windows, (d) anims merged into the host set. The raccoon is
  quadruped-shaped and small - ideal dog host (vs pig for big creatures).

## 3. ProjectALife: the modular NPC suite
- Structure: `client/ProjectALife/{Admin,Combat,Compat,Interface,...}` + `common/alife_runtime/catalog`
  - an NPC framework organized by subsystem, plus a shared runtime catalog.
- **`Compat/` adapters are the standout**: `ALifeAdvancedTrajectoryAdapter`,
  `ALifeAutoAllAdapter`, `ALifePointBlankAdapter`, `ALifeWanderingZombiesAdapter`,
  `ALifeForagingGuard` - one file per other-mod integration = the compat-shim pattern from the
  mod-interop note at production scale. Detect -> adapt -> isolate.
- Combat: `ALifeHordeRelations` (faction/aggro relations between NPCs and hordes), `ALifeTracer`
  (real tracers - the effects-param layer in use), plus interface (`ALifeContextMenu`,
  `ALifeEncounterCompass`), admin (`ALifeDebugSpawner`), and the `Aftermath`/`Jimmy`/`KnoxFractured`
  sibling mods layering scenarios over the same `alife_runtime` - a framework + content-mod split
  worth copying for big projects.

## The unified recipe for "a person in the world"
1. Chassis: zombie for humanoids (Bandits), animal for quadrupeds (CompanionDogs) - the decision
   table holds.
2. Identity: `setVariableBoolean` flag + brain table keyed by stable ID; handle recycle/unload.
3. Motion: custom walk types via `BanditWalkType`-style variable + `setBumpType` for anim transitions
   + `pathToLocationF`/`faceLocationF` for movement + `IsController` gating for MP.
4. Behaviour: task library (`onStart/onWorking/onComplete` modules) + program state machines
   (`{status,next,tasks}` returns) - the Bandits ZA/ZP architecture generalizes to any NPC.
5. Presentation: merged AnimSets/actiongroups with your prefix; clothingItems/outfits for bodies.
6. World presence: bases/posts via world-marker systems (`BanditPost`), loot/inventory management,
   context menus + radial commands + status windows.
7. Scale: subsystem folders + a `Compat/` adapter directory + a runtime catalog (the ALife layout).

## Gotchas
1. **`isServer() return` in OnZombieUpdate** - Bandits brains run client-side; dedi brains need the
   server/ path or command relays (MP note).
2. **`setVariableBoolean` identity persists in the anim map** - cheap, but a recycled zombie keeps the
   flag until a controller clears it (the same identity-recycle class of bug as outfits).
3. **49 actions is the real cost of "an NPC"** - the brain is easy; the verbs (equip, load, loot,
   open windows, fix generators) are the work. Budget for the library.
4. **`IsController` gating is mandatory in MP** - every client can see the zombie; only one may steer
   it (`pathTo*` calls gated, presentation broadcast).
5. **Raccoon AnimSet merges are fragile across builds** - 42.20 vs 42.21 folders exist in Bandits for
   exactly this reason; version your anim merges per build dir.
6. **Brains-per-tick cost** - Bandits' CacheLight (position-only tables) exists because reading
   IsoZombie fields per tick at population scale is the hot path; copy the light-cache pattern.
7. **`setBumpType` drives more than bumps** - it's the locomotion-transition channel; `BumpDone` is
   the completion signal (the CDDA knockdown trick generalized).
8. **Program returns are declarative** - `next="Guard"` switches program; logic errors surface as
   stuck states, not crashes - log program transitions while debugging.
9. **Framework+content split pays off** - ALife's `alife_runtime` + per-scenario mods (Jimmy, Knox)
   is how you scale past one NPC type.
10. **Adapters isolate fragility** - one file per external mod (`Compat/`), detected at runtime;
    don't scatter `if OtherMod` checks through the main logic.

## Verification
- Verified by file inspection on the listed workshop items (42.21 targets): file/folder names, the
  ZA/ZP architecture, the `setVariableBoolean("Bandit")` identity, `setBumpType`/`BanditWalkType`
  locomotion, `IsController` gating, `isServer()` early-outs, raccoon anim merges, ALife Compat
  adapters, `alife_runtime` layout.
- Not verified in-game this session: running the mods (download inspection only), MP behaviour of
  Bandits brains, `ZA*` module internals beyond signatures shown.

## Assets
- Bandits: custom `actiongroups/`, `AnimSets/`, `anims_X/`, `clothing/`, `sound/`, `textures/`, `ui/`
  in `common/media` - the survivor anim/visual bank.
- CompanionDogs: `models_X`, `textures`, `sound`, `AnimSets/raccoon` merges, `ui`.
- ALife: `models_X`, `AnimSets`, `actiongroups`, `clothing`, `sound`, `textures` + `alife_runtime`.

## Open questions
- `ZA*` action module full catalogue semantics (49 files - each a small read; the naming is the API).
- Whether Bandits' identity also uses persistent outfits under the variable (belt+braces) or relies
  on the flag alone across chunk reload.
- ALife `alife_runtime/catalog` exact role (data registry? spawn catalog?).
