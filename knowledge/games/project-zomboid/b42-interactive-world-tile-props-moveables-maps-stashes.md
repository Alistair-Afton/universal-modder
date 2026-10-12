---
kind: game
title: 'Build 42 interactive world: tiledef properties, moveables, object taxonomy, map mods, world-map symbols and annotated-map stashes'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); verified against install media + StashUtil/MapSymbolDefinitions Lua'
platform: windows
engine: java
route: loader-api
tools: ["Project Zomboid Lua (Kahlua) mod API", "TileZed/WorldEd community forks", "in-game WorldMap editor (media/lua/client/ISUI/Maps/Editor)"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Tile_properties"
- "https://pzwiki.net/wiki/Mapping"
tags: [b42, tiles, tiledef, tile-properties, moveables, iso-objects, maps, map-mods, spawnpoints, worldmap, symbols, stashes, annotated-maps, lootable-maps]
---
# Build 42 interactive world: tiledef properties, moveables, object taxonomy, map mods, world-map symbols and annotated-map stashes

> The "world is made of objects" layer: how a sprite becomes a stove, a chair, a lootable or a barricade (tiledef
> properties + moveables), how map mods plug into the world, how to draw on the in-game map, and the stash
> system that makes annotated treasure maps spawn on zombies. Closest to "level design" modding gets.

## Setup
- PZ **42.21.0** Steam/Windows; `media/` of the install + `media/lua/shared/StashDescriptions/`,
  `Moveables/`, `Definitions/MapSymbolDefinitions.lua`, `client/ISUI/Maps/`.

## Route and why
Data + Lua. Tile behaviour is properties on tiles (set in `.tiles`/`.txt` tiledefs or at runtime); interactions
are IsoObjects + Lua; the map layer is fully scriptable.

## How the game works

### Tile properties = gameplay, not just art
Every tile in a `.tiles` file has a key/value property set (readable sidecar `.tiles.txt` -
`newtiledefinitions.tiles.txt` ships next to the binary and shows the schema):
```
tile {
    xy = 0,0
    Facing = S
    Material = MetalPlates          -- hit sound/debris material
    MaterialType = Metal_Large      -- build/scrap logic
    Material2/Material3 = ...       -- extra scrap yields
    BlocksPlacement =               -- blocks building on the square
    CanScrap =                      -- sledgeable into materials
    container = stove               -- loot container of type 'stove'
    solid =                         -- blocks movement
    attachedFloor =                 -- binds to floor sprite
    alwaysDraw =
}
```
Properties seen across vanilla tiledefs: `container` (turns the tile into an ItemContainer of a named type -
that's how ovens/wardrobes hold loot), `Door`, `doorN/doorW` + `open`/`closed` sprite pairs, `floor`,
`WallN/WallW`, `solid`/`solidtrans`/`collideN/W`, `water`, `bed`, `sittable`, `Table`, `vegitation`,
`stackReplace`/`ReplaceTiles`, `curtain`, `tree`, `bush`, `connectedWall`, `TileOverlay`, erosion hooks,
`surface` (vehicle/footstep surface), `FloorHeight`, `stair`. The `.txt` sidecar is the authoritative list of
what vanilla tiles set - grep it before inventing a property. Custom tiles get these via TileZed or
hand-written `.pack`/`.tiles` (format in the CDDA note).

### IsoObject taxonomy (what sits on squares)
`zombie/iso` owns the object classes a square can hold - the ones mods touch most:
`IsoDoor`, `IsoWindow` (+ `IsoCurtain`, `IsoBarricade`), `IsoThumpable` (buildables/zombie-attackable),
`IsoGenerator`, `IsoStove`/`IsoFireplace`, `IsoCompost`, `IsoMannequin`, `IsoWorldInventoryObject` (ground
items), `ItemContainer`, `IsoLightSource`, `IsoJukebox`, `IsoDeadBody`, `IsoFire`/`IsoSmokeSource`,
`IsoTrap`, `IsoWaveSignal` (radios/TVs - see broadcast note), `IsoCarBatteryCharger`. Create with
`IsoObject.new(cell, sq, sprite)` + `sq:AddTileObject` (CDDA note); special classes have their own
constructors (`IsoThumpable.new` etc.).

### Moveables: why you can pick up a chair
`shared/Moveables/ISMoveableDefinitions.lua` + `ISMoveableSpriteProps.lua` - the system that decides which
world sprites are liftable, with what tool, skill and chance (breakable pickup!), and what item they become:
- `ISMoveableSpriteProps.fromObject(obj)` reads the sprite's props; `ISMoveableDefinitions` maps sprite names
  -> `{ item, weight, requiredTool, requiredSkill, breakChance }`.
- `ISMoveablesAction` = the timed action; `ISMoveablesIconToolTip` shows requirements.
- Making custom furniture carryable = entries for your sprite + the tile's moveable props; the same system
  powers scrap (`CanScrap` + Material fields decide yields).

### Map mods
- `media/maps/<MapName>/`: `map.info` (`title`, `description`, `lots = Muldraugh, KY` to parent onto the world
  map so cells stitch in), `spawnpoints.lua` (per-profession spawn coords), `thumb.png`, plus the binary
  chunks: `.lotheader`, `*.lot` (buildings), `.chunkdata_X_Y.bin`, `.bin`, `world_<x>_<y>.lotpack` etc.
  - binary formats; produced by TileZed/WorldEd (community forks for B42 - see anatomy note).
- Cell replacement only: B42 cells are 256x256; you can't add a lone building inside an existing vanilla cell
  without replacing the whole cell (pzwiki mapping page).
- `mod.info` `map=` lines + `tiledef=`/`pack=` for the tilesets the map needs.

### The in-game map & symbols
- `client/ISUI/Maps/`: `ISMap.lua`/`ISWorldMap`/`ISMiniMap`/`ISMapSymbols`/`ISWorldMapSharing` + a full
  **`Editor/`** (WorldMapEditorMode_* incl. `WorldMapEditorMode_Stashes.lua`) - the dev map-annotation editor
  ships in the client; spawnable from debug for authoring map annotations in-world.
- `MapSymbolDefinitions.getInstance():addTexture("Name", "media/ui/LootableMaps/map_x.png", "Category")` -
  register your own stamps (categories: Symbols, Loot, Locations). Symbols carry color+scale.
- An item becomes a readable/annotatable map via `item` fields (`Map = ...`, `ItemType = base:map`-style) +
  `media/ui/LootableMaps/` textures; `ISMap.lua` is the paper-map UI (pan/zoom/draw symbols).
- `zombie/worldMap` (153 classes) is the underlying system incl. the map item's symbol storage.

### Annotated-map stashes (the buried-treasure system)
`shared/StashDescriptions/*StashDesc.lua` + `StashUtil.lua` - how "a zombie drops a map with an X on it"
works:
```lua
local s = StashUtil.newStash("MulStashMap1", "Map", "Base.MuldraughMap", "Stash_AnnotedMap")
s.spawnOnlyOnZed = true          -- or on ground/containers
-- s.daysToSpawn = "0-30"; s.zombies = 5; s.traps = "1-5"; s.barricades = 50
s.buildingX, s.buildingY = 10660, 9758    -- target building
s.spawnTable = "GunCache1"                -- loot table name
s:addContainer("GunBox", "floors_interior_tilesandwood_01_60", nil, "livingroom")
s:addStamp("X", nil, 10657, 9757, r, g, b)            -- symbol or nil
s:addStamp(nil, "Stash_MulMap1_Text1", 10673, 9755, ...) -- text annotation (Stash.json keys)
```
Stashes can spawn zombies/traps/barricades at the target, place a named container with a loot table, and stamp
symbols/text onto the map item. `WorldStashDesc.lua` + `WpStashDesc.lua` are the global/weapon-shop examples;
debug UI `StashDebug.lua` visualizes them. Custom treasure-hunt mods = a new `*StashDesc.lua` + Stash.json
translations + (optionally) new map-symbol textures.

## Build steps
1. Interactive tile: give your `.tiles` entry the property set (container type, sittable+seating, material,
   facing), then place via map mod or `IsoObject`.
2. Carryable furniture: `ISMoveableDefinitions` entry for the sprite + tile props.
3. Map mod: TileZed cells -> `media/maps/<name>/` + `map.info` (`lots = Muldraugh, KY` to stitch into the
   world) + `spawnpoints.lua` + `mod.info` `map=` line.
4. Treasure map: `StashUtil.newStash` in a shared file + symbol textures via `MapSymbolDefinitions`.
5. Custom device: IsoObject + SGlobalObjectSystem pair (see character note) for persistence.

## Verification
- Tile props: `-debug` tile/object inspector shows live properties on any square; `console.txt` for
  container-type misses.
- Stashes: `StashDebug` UI + spawn a stash map on a test zombie (`-debug` tools) and read it.
- Map mod: the Mods screen lists map mods with `map=`; WorldEd map preview + `console.txt` chunk errors.
- Not verified: hand-written `.lot` chunks (use TileZed); `ISUI/Maps/Editor` completeness for publishing
  authored annotations.

## Gotchas
1. **Tile property typos are silent.** Unknown keys are ignored - a stove that isn't `container = stove`
   exact is just decoration. Copy property names from `newtiledefinitions.tiles.txt`.
2. **Multi-sprite objects need all parts.** A 2x1 bed is two tiles sharing `Facing`/group props; moveables
   multi-sprite radius (`multiSpriteFloorRadius = 3`) decides what joins the pickup.
3. **`lots` is what stitches your map into the world.** Missing `lots = <map>` in map.info = standalone
   challenge map that doesn't border Knox Country.
4. **Stash containers need a real room name** (`"livingroom"` above) that exists in the target building's
   RoomDef, or the spawn targets a random room instead.
5. **Map symbol textures must be registered before the map UI builds** - `addTexture` at `OnGameBoot`.
6. **Annotated-map stamps use world coords** - verify buildingX/Y land inside the building (`-debug` coord
   readout), off-by-one = X on the neighbor's house.
7. **`spawnOnlyOnZed` vs container spawns** - loot maps on corpses need the zombie loot tables to roll them
   (SuburbsDistributions/procedural entries, see world-sim note).
8. **Cell-replacement mods conflict permanently** - two mods touching cell (x,y) can't merge; document the
   cells you replace in the description.
9. **Moveables breakChance rolls on pickup** - a "guaranteed" custom furniture needs `breakChance = 0` plus
   skill gating; otherwise players smash it.
10. **`StashUtil` runs where the map item generates** - stash creation code belongs in shared/server paths
    that run during loot creation, not client UI code.

## Assets
- Map symbol PNGs: `media/ui/LootableMaps/` (small glyphs, categories Symbols/Loot/Locations).
- Tile sheets/`.pack`/`.tiles`: see CDDA note for the hand-writable formats.

## Open questions
- The full tiledef property dictionary (the `.txt` sidecars show usage, not the enum; `javap` on
  `zombie.core.properties` / `IsoFlagType` enumerates it).
- Whether `ISUI/Maps/Editor` can export annotations for shipping (looks dev-internal).
- `.lot`/`.chunkdata` field layouts (binary; TileZed writes them).
- Custom RoomDef types (room loot categories) - how far Lua can register new room names vs map-file data.
