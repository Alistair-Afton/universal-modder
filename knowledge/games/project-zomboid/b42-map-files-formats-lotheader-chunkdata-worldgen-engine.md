---
kind: game
title: 'Map files and the worldgen engine: lotheader/chunkdata binaries, map folder anatomy, the 101-class procedural worldgen runtime, areas/zones internals'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); jar/file-system-verified'
platform: windows
engine: java
route: loader-api
tools: ["TileZed/WorldEd community forks", "media/maps/<town>/ folder anatomy", "worldgen Lua tables", "javap (JDK)"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Mapping"
tags: [b42, maps, lotheader, chunkdata, worldgen, binary-formats, zones, areas, towns, procedural]
---
# Map files and the worldgen engine (42.21.0)

> The last flagged territory: what map data actually IS on disk, and the procedural engine that can
> replace it. Census results: a town folder is a grid of binary cells (`N_M.lotheader` + `chunkdata_N_M.bin`)
> plus Lua control files, and `zombie.iso.worldgen` is a **101-class procedural world generator** with
> biomes/roads/rules/veins/zones/blending - vanilla wilderness is generated, not painted.

## Setup
- File/jar-verified on 42.21.0 (`media/maps/<town>` + `zombie.iso.*` census). For making maps the
  supported way see the interactive-world note (TileZed/WorldEd, map.info, spawnpoints, cell rules).

## Route and why
Map creation = external tools; map *semantics* = this note. Don't hand-write binaries.

## 1. A town folder, file-verified (`media/maps/Muldraugh, KY`)
- **`N_M.lotheader`** (one per 300x300-cell region - the .lot header): binary per-cell tile/stream
  data (building interiors, tile IDs, properties). `IsoChunk`/`IsoChunkLevel`/`ChunkSaveWorker`
  (jar-verified) load/save them.
- **`chunkdata_N_M.bin`** (one per region): the binary cell payload - tile grids, object IDs,
  meta. The `.lot`+header+chunkdata trio IS the compiled map.
- **`WorldGenOverride.lua`** (per town!): worldgen parameters overriding biome/feature tables for this
  town's procedural edges.
- **`basements.lua`**: basement prefab registrations (the `Basements.getAPIv1()` + `.pzby` binary pair
  from the world-sim note).
- `map.info`, `objects.lua`, `spawnpoints.lua`, `thumb.png` (in mod/MP maps - see the maps note).
- Top level `media/maps/` = town folders + `challengemaps/` - each challenge can ship its own map set.

## 2. The worldgen engine (`zombie.iso.worldgen`, 101 classes, jar-verified)
- Core: `WorldGenReader`/`WorldGenParams`/`WorldGenChunk`/`WorldGenTile`/`WorldGenSimplexGenerator`/
  `WorldGenTiming`/`WorldGenUtils`/`StaticModule`/`PrefabStructure`/`ChunksCache`/`WorldGenDebug`.
- Sub-packages: `biomes`, `roads`, `rules`, `veins` (ore/loot veins in wilderness), `zones`,
  `blending` (edge blending between generated and authored cells), `maps`, `attachments`, `enums`,
  `utils`, `zombie`.
- Mod-facing: the Lua `worldgen` table (world-sim note) declares biomes/features/prefabs/veins/roads;
  these classes are the engine that consumes it. `WorldGenOverride.lua` is the per-town override file -
  a map mod can ship one to reshape the wilderness around its cells.
- Implication: "generate a whole new county procedurally" is a data + engine feature, not a fantasy -
  the same system that makes vanilla wilderness can make yours, gated on how much of the schema the
  `worldgen` table exposes (the flagged semantic gap, now mapped to named classes).

## 3. Zones and areas internals (`zombie.iso.areas`, 47 classes)
`IsoMetaGrid`/`registerZone`-family write here: `Zone`, `ZoneType` enums, `IsoRoom`, `IsoBuilding`,
`IsoRegions`-family - the named-area layer everything else queries (forage zones, room tones,
mannequins, water zones, story eligibility). Zones are world metadata (metagrid = survives unload);
squares are world content (binary cell = unloadable).

## 4. What the binaries are (and why you still don't write them)
- `.lotheader`/`.bin` = serialized tile/object streams consumed by `IsoChunk` loaders - the format
  shifts between builds (the 42.21 `.bin` layout is not documented anywhere public). Community tools
  (TileZed forks) are the only maintained readers/writers - write maps there, let the tools emit
  binaries.
- The mod-relevant parts are NOT the binary layout but: the cell grid (300x300, `lots=` stitching rules),
  `map.info`, `spawnpoints.lua`, `objects.lua` (zone/property tables), `basements.lua`,
  `WorldGenOverride.lua`, thumbnail/story eligibility. All text or tool-authored.
- If you must inspect binary semantics: decompile `IsoChunk`/`ChunkSaveWorker`/`LotHeader`-family
  (jar-verified names above) - that IS the format spec; never commit decompiled code.

## Gotchas
1. **`.lotheader` and `chunkdata` come in pairs** - one without the other = corrupt cell; tools emit both.
2. **256-cell replacement rule** (maps note) applies at load - whole-cell replace only, no partial merges
   of binary data.
3. **`WorldGenOverride.lua` is per-town** - it shapes the wilderness ring around authored cells, not the
   cells themselves.
4. **`basements.lua` + `.pzby`** pair the Lua registration with the binary prefab - the prefab format is
   binary too (same "tools only" rule).
5. **`IsoChunk` is the load unit** - "the square changed" means a chunk was touched; chunk-level writes
   are what `transmitAddObjectToSquare` actually syncs.
6. **Worldgen runs on empty cells** - it fills un-authored space; your authored cells are exempt (the
   blending package handles edges).
7. **`StaticModule`/`PrefabStructure`** are the "drop a built thing into generated space" path - for
   scattered structures the engine supports it natively.
8. **Don't copy `media/maps` into a mod** - maps ship via `map.info` + mod folders; vanilla town folders
   are reference material, not templates to clone wholesale.

## Verification
- File/jar-verified 42.21.0: the Muldraugh folder listing (`.lotheader`/`chunkdata`/`WorldGenOverride.lua`/
  `basements.lua`), `zombie.iso.worldgen` 101-class census, `IsoChunk`/`IsoChunkLevel`/`ChunkSaveWorker`
  names, `zombie.iso.areas` 47-class census.
- Verified in the wild: TileZed/WorldEd forks produce these files; `basements.lua` pairs with the
  `Basements` API; `WorldGenOverride` matches the `worldgen` Lua table system.
- Not verified: the `.bin`/`.lotheader` byte layout (tool authors' domain), whether mod `worldgen`
  tables feed `WorldGenSimplexGenerator` directly end-to-end.

## Open questions
- `.lotheader`/`chunkdata_*.bin` exact byte schema (decompile `IsoChunk` loaders when needed - the
  classes are the spec).
- `worldgen` Lua table -> `WorldGenParams` field mapping (the semantic gap, now class-named).
- `veins`/`blending`/`rules` package semantics (procedural rules grammar).
