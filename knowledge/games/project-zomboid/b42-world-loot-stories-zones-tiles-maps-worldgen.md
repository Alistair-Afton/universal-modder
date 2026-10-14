---
kind: game
title: 'Build 42 world simulation: loot distributions, random stories, metazones, WorldGen, basements, climate and meta events'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); verified against media/lua/server + jar class census'
platform: windows
engine: java
route: loader-api
tools: ["Project Zomboid Lua (Kahlua) mod API", "javap / ZomboidDecompiler for the zombie.* packages"]
anti_cheat: 'none (server-side simulation modding)'
status: in-progress
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/ProceduralDistributions"
- "https://pzwiki.net/wiki/Distributions"
tags: [b42, loot, distributions, proceduraldistributions, world-stories, metazones, metagrid, zones, worldgen, basements, climate, weather, meta-events, helicopter, ai-director, java-internals]
---
# Build 42 world simulation: loot distributions, random stories, metazones, WorldGen, basements, climate and meta events

> The systems that make the *world* tick - loot tables, randomized world stories, the zone metagrid, B42's Lua
> world generator, the new basements API, climate/seasons, and the Sadistic AI Director meta events. All of it
> is plain Lua tables or Java methods a mod can extend. This is the layer for mods that change *what the world
> does*, not what things look like.

## Setup
- PZ **42.21.0** Steam/Windows. All systems below live in `<install>/media/lua/server/` (readable) and
  `projectzomboid.jar`'s `zombie.*` packages (decompilable, see Route).

## Route and why
Loader API again - every system here is data (Lua tables) or exposed Java calls. For the internals that aren't
exposed cleanly, decompile with ZomboidDecompiler and *describe*; never ship patched classes.

### The Java package map (for decompile-driven questions)
Class counts per package in `projectzomboid.jar` (42.21.0) - where to look when Lua docs run out:

| Package | ~Classes | Owns |
|---|---|---|
| `zombie/core/` | 773 | renderer, sprites, textures, core engine |
| `zombie/iso/` | 661 | the live world: IsoCell, IsoGridSquare, IsoObject, IsoPlayer, rooms, fire, weather objects |
| `zombie/network/` | 628 | MP packets, sync, server logic |
| `zombie/scripting/` | 407 | the script parser (all the block types) + entity scripting |
| `zombie/characters/` | 339 | IsoGameCharacter stats, BodyDamage, traits, skills, safety/combat |
| `zombie/entity/` | 328 | the B42 entity-component system behind `entity` script blocks |
| `zombie/util/` | 207 | collections, io, misc |
| `zombie/audio/` | 166 | FMOD wrapper, emitters, GameSounds |
| `zombie/randomizedWorld/` | 164 | world stories, abandoned cars, burnt houses, the helicopter |
| `zombie/worldMap/` | 153 | the annotated map / symbols system |
| `zombie/vehicles/` | 146 | BaseVehicle, parts, physics |
| `zombie/ai/` + `zombie/pathfind/` | 252 | zombie states, pathfinding |
| `zombie/debug/` | 124 | the -debug menus (study it for dev-tools ideas) |
| `zombie/commands/` | 79 | admin/server commands (`/additem`, `/godmod`...) - mirror for custom commands |
| `zombie/inventory/` | 76 | items, containers, ItemPicker |
| `zombie/erosion/` | 57 | plant/crack growth over time |
| `zombie/gameStates/` | 46 | menu/loading/game states |
| `zombie/tileDepth/` | 41 | isometric depth |
| `zombie/radio/` + `zombie/chat/` | 68 | broadcast channels, chat |
| `zombie/popman/` | 35 | PZPopMan - zombie population director |
| `zombie/Lua/` | 31 | LuaManager, event dispatch, Kahlua binding |
| `zombie/savefile/`, `zombie/fileSystem/` | 43 | saves, chunk io |
| `zombie/basements/` | 12 | B42 basements |
| `zombie/seams/`, `zombie/seating/`, `zombie/pot/`, `zombie/spnetwork/`, `zombie/gizmo/`, `zombie/vispoly/`, `zombie/asset/`, `zombie/combat/`, `zombie/statistics/`, `zombie/sandbox/`, `zombie/buildingRooms/`, `zombie/globalObjects/`, `zombie/profanity/`, `zombie/text/` | ~120 | seams between chunks, seats, pottery UI?, SP networking, editor gizmos, visibility polys, asset db, melee combat, stats, sandbox opts, room detection, global objects, word filters, localization |
| native DLLs | - | `PZBullet64` (physics), `PZClipper64` (geometry), `PZPathFind64`, `PZPopMan64`, `PZPathFind`, `ZNetJNI64`/`ZNetNoSteam64` (net), `fmod*`, `jassimp64` (model import), `Lighting64` |

## How the game works

### Loot: Distributions + ProceduralDistributions
- `media/lua/server/Items/Distributions.lua` (+ `Distribution_*Junk.lua`, `SuburbsDistributions.lua`) defines
  the `Distributions`/`SuburbsDistributions` tables: per room type (`all`, `kitchen`, `gunstore`...), per
  container type, weighted item lists.
- `ProceduralDistributions.list` = named tables `{ rolls = N, items = {"Item",weight,...},
  junk = { rolls, items } }` - container types reference these by name via `procedural` entries; rooms get the
  base `Distributions` tables plus procedural overlays.
- **Modding it**: a shared/server Lua file appends entries (`table.insert(ProceduralDistributions.list["GunStoreGuns"].items, "MyGun")` and its weight) or adds whole tables. `ItemPickerJava.getLootModifier(item)` is the rarity modifier hook; the `RemoveStoryLoot` sandbox option blacklists story clutter.
- `ScriptManager.instance:FindItem("Base.X")` validates an item id at runtime (how vanilla catches typos in
  loot tables - use it in your init to log bad entries).

### Randomized world stories
- `zombie/randomizedWorld/` (Java, 164 classes) + `server/RandomizedWorldContent/StoryTable_Initialization.lua`
  + `StoryClutter/`: house/street scenarios placed when the world generates - boarded windows, corpses with
  loot, burnt houses, crashed cars, table settings (the clutter tables above).
- Stories attach to rooms at world-gen time (like the nest-site lesson from the CDDA note: pick the site early,
  build when the square loads). `RandomizedWorldBase` subclasses get called per candidate room; criteria are
  room type + zone.
- Story clutter items go through `StoryTables.addItemToClutterTable` which validates against
  `ScriptManager` and the sandbox blacklist.

### Metazones / the zone metagrid
- `getWorld():getMetaGrid()` is the global zone registry that works on *unloaded* world - the same API the
  CDDA note used for town-edge detection.
- `server/metazones/metazoneHandler.lua` shows the moddable registration calls, driven by map-provided data:
  `getWorld():registerAnimalZone(name,type,x,y,z,w,h,properties)`,
  `registerMannequinZone(...)`, `registerRoomTone(...)`,
  `getMetaGrid():registerGeometryZone(name,type,z,geometry,points,properties)` (polygon zones),
  plus spawn-origin and **basement spawn location** handlers.
- `AnimalsPathConfig.lua` and `BiomeMapConfig.lua` show zone-driven animal/biome data: animal habitats are
  metazone regions with property maps - custom monsters can get natural habitats this way.

### Basements (new in B42)
- `Basements.getAPIv1()` (Lua) exposes `registerBasementSpawnLocation(mapID, name, type, x, y, z, w, h,
  properties)` - basement entry points are registered per map id.
- `media/basement_access/*.pzby` = binary prefab building chunks (header `PZBY`, a tileset-name string table,
  then per-tile records) used for basement exteriors/interiors/stair access pieces. Format is compact binary;
  read it with a script if you need to inspect (strings are length-prefixed).
- `zombie/basements/` (12 classes) is the runtime; `ProceduralBasements/` in shared Lua too.

### WorldGen (B42's Lua world generator)
- `server/WorldGen/` with `Biomes.lua`, `Features.lua`, `Roads.lua`, `Veins.lua`, `Selection.lua`,
  `attachments/Priorities.lua`, `biomes/`, `features/`, `prefabs/` and a global `worldgen` table of
  `{biomes,features,selection,prefabs,veins,roads,attachments,similar,priorities}`.
- This is how wilderness/biome content is generated at world creation (or how TIS generates it) - the tables
  are moddable: add biomes/features/prefabs like adding distributions. The metazone `BiomeMapConfig` feeds it.
- Under-documented publicly; read `Features.lua`/`Biomes.lua` for the schema (it's self-describing Lua).

### Climate, seasons, erosion
- `server/Climate/ClimateMain.lua` - the weather/climate driver; `getClimateManager()` (Java) exposes rain,
  fog, thunder, temperature, wind; `client/erosion/` + `server/Seasons`/`erosion` drive visual season +
  overgrowth on a `zombie/erosion` backend (ErosionManager). Erosion tiles come from
  `media/tiledefinitions_erosion.tiles` - modded tiles can participate in erosion if they carry the right
  properties.
- `OnClimateTick`-family events exist for weather-driven mods; weather channels on the radio
  (`server/radio/ISWeatherChannel.lua`) read the same manager.

### Meta events / the AI Director
- `server/NPCs/SadisticAIDirector/SadisticMusicDirector.lua` - named "Sadistic" after its predecessor
  (`SadisticAIDirector` governs offscreen drama: nearby screams, gunshots, the helicopter event, house alarms
  that pull hordes). The music half is client-side and reads
  `playerObj:getStats():getNumVisibleZombies()/getNumChasingZombies()` to pick track intensity - so those two
  stat fields exist for your own "danger meter".
- Meta events live in `zombie/randomizedWorld` + the AI director package; they're world-level, fire through
  `OnWorldSound`/`WorldSoundManager` for the audible parts, and helicopter-style events are scheduled by game
  time + sandbox settings (`MetaEvent` sandbox options). Custom meta events = a server Lua loop that schedules
  world sounds + spawns, using the same primitives.

### Admin/server commands
- `zombie/commands/` (79 classes) implements `/additem`, `/godmod`, `/createhorde`, etc. Custom slash commands
  come free via `Events.OnClientCommand` handled server-side + chat parsing - or for dedicated servers, the
  RCON/command channel. `client/ServerCommands.lua` lists vanilla client->server verbs.

## Build steps
1. Loot mod: `server/` Lua, append to `ProceduralDistributions.list` + `SuburbsDistributions`; validate every
   id with `ScriptManager.instance:FindItem`.
2. Story mod: study a small `zombie/randomizedWorld` class via decompile for the base class contract, or use
   `StoryClutter`-style Lua tables if your story is loot-only.
3. Zone-driven feature: register zones through `getWorld()`/`getMetaGrid()` from `server` at `OnInitWorld`/
   map-data load; query anywhere with `getZonesAt` (unloaded-friendly).
4. WorldGen content: extend `worldgen.features`/`biomes` tables in a `server` file (runs before world build).
5. Meta event: server-side scheduler + `addWorldSound`/spawn calls + sandbox option to tune frequency.

## Verification
- Loot: `-debug` loot-window / item viewer shows procedural tables; `console.txt` prints the
  "item script does not exists" warnings for bad ids at world load.
- Zones: `-debug` iso-region/metazone visualizers draw registered zones.
- WorldGen/stories: generate a new world with only your mod; check `console.txt` story/gen logs.
- Basements API: `javap`/decompile `zombie.basements` for the full v1 surface (only the registration call was
  verified here from metazoneHandler).
- Not verified: custom `worldgen.prefabs` end-to-end; radio channel authoring beyond data schema; dedicated
  server for any of it.

## Gotchas
1. **Loot entry typo = silent missing item.** `FindItem` returns null; vanilla only prints. Validate your ids
   at boot and `print` failures yourself.
2. **Procedural table edits must run before containers fill.** Rooms are looted when first loaded - patch
   tables in `OnInitWorld`/shared init, not on demand.
3. **Zone queries on unloaded squares** return nil for `getCell():getGridSquare` but `getMetaGrid():getZonesAt`
   still works - plan around it (CDDA gotcha, same API).
4. **World stories only exist at world creation.** A story mod can't retrofit into an old save's already-
   generated chunks; new cells only.
5. **`getNumVisibleZombies` is client-only truth** - on a dedicated server read population via the server
   zombie lists instead.
6. **Erosion ignores mod tiles without the right tiledef properties** - copy erosion-capable property sets
   from `tiledefinitions_erosion.tiles`.
7. **Basement registration needs the mapID of an installed map** - registering for `Muldraugh, KY` affects the
   vanilla map; wrong mapID silently does nothing.
8. **SadisticAIDirector is SP-shaped.** On dedicated servers the drama logic mostly doesn't run; replicate
   with your own scheduler if MP needs it.
9. **`.pzby` is binary, not text** - don't hand-edit; if you need custom basement pieces, build the structure
   in game and study how the format stores tile ids (or wait for tooling).
10. **StoryTables clutter runs through the sandbox blacklist** - `RemoveStoryLoot=true` can strip your story's
    custom items unless `ItemPickerJava.getLootModifier` marks them exempt (modifier != 0).

## Assets
n/a - simulation-layer note.

## Open questions
- Full `worldgen` schema (features/prefabs/veins semantics) - only the table skeleton is mapped here; the
  `biomes/`, `features/`, `prefabs/` subfolders have the real definitions.
- The complete `Basements` v1 API surface.
- Whether mods can register entirely new randomized world story *classes* in Lua (needs a
  `RandomizedWorldBase` subclass - probably Java-only; the Lua clutter tables are the reachable half).
- `.pzby` field-by-field layout (header + tileset table + tile records confirmed; record layout not decoded).
