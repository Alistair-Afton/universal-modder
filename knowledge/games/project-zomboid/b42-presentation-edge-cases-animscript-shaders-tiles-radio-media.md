---
kind: game
title: 'Build 42 presentation edge cases: animscript combat XML, shaders, spriteModels, seating, radio/TV, recorded media, music, fonts'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); verified against install media/ tree'
platform: windows
engine: java
route: loader-api
tools: ["Project Zomboid Lua (Kahlua) mod API", "media/ text formats"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Animation"
- "https://pzwiki.net/wiki/Recorded_media"
tags: [b42, animscript, actiongroups, animset-root, shaders, glsl, spritemodel, seating, cutaways, radio, recorded-media, tv, music, fonts, effects, tracers, gibs, voicestyles, presentation]
---
# Build 42 presentation edge cases: animscript combat XML, shaders, spriteModels, seating, radio/TV, recorded media, music, fonts

> The obscure media files and presentation systems most modders never open: the second (older) animation
> language in `animscript/`, real GLSL shaders sitting in the install, attaching 3D models and seat anchors to
> tiles, wall cutaway masks, bullet tracer/effects parameter files, the radio/TV broadcast schema, the recorded
> media (CDs/VHS) table, the dynamic music system, and bitmap fonts. Everything here is moddable data.

## Setup
- PZ **42.21.0** Steam/Windows. Paths are `media/` inside the install unless noted.

## Route and why
Loader API / data-only. None of this needs Lua skill beyond table edits - it's for mods that change how the
game *looks, sounds and talks*.

## How the game works

### Two animation systems, not one
- The modern stack (AnimSets/ActionGroups/animNodes - covered in the anatomy note) has a sibling:
  `media/animscript/combat.xml` - an older **AnimStates XML** language with named states containing
  `<Commands>` (`<PlayAnim><Anim>Bob_X</Anim><Repeat>0</Repeat>`) and `<Out>` transitions
  (`<NextState><BlendAnimState/><BlendTime>0</BlendTime>`), plus `AllowChangeAngle`/`MoveCancel`. It drove the
  old combat/cover state machine; still loaded - check before assuming AnimSets is the only graph.
- `media/actiongroups/<entity>/` mirrors AnimSets: per-entity folders plus `.xml` roots
  (`zombie.xml`, `player.xml`, `cow.xml`...) whose `<animNodeFilter>` declares
  `folderPrefixes`/`namePrefixes` (zombie = `Bob`,`Zombie`; that's why human anims need `Bob_`/`Kate_`
  prefixes). `Master_Bones.xml`/`Master_Variables.xml`/`Defaults.xml` define the shared skeleton and the full
  variable list animNode conditions can read - the master list of `setVariable` targets.
- `media/animstates/` has test files (`testanimstates.xml`) - the format sandbox.

### Shaders are plain GLSL files in the install
`media/shaders/*.frag/.vert` (~50 pairs): `aim_outline_*`, `CutawayAttached/NoDepth`, `CircleStencil`,
`DeadBodyAtlas`, `addHole`, lighting passes. `README-Shaders.txt` documents the downlevel path (GLSL 330 ->
GL 2.1 by string rewriting: `#version` swap, layout->glBindAttribLocation, in/out->attribute/varying).
Whether *mod* shaders load is unverified (treat as engine-owned), but reading them explains the render
effects you see (aim outlines, corpse atlas, wall cutaways) - and a copied file at the same relative path in a
mod *should* override by the load-path rule (untested).

### Attaching things to tiles without TileZed
- `media/spriteModels.txt`: `spriteModel { tileset { name, tile { xy, modelScript = Base.X,
  translate, rotate, scale } } }` - renders a real 3D `model` on a tile sprite (vanilla: radio on a counter).
- `media/seating.txt`: `seating { tileset { name, tile { xy, position { id = S|E|N|W, translate } } } }` -
  seat anchor points per furniture tile, which is what makes a chair sittable.
- Cutaway/transparency masks are plain PNGs: `wallcutaways.png`, `windowframe_cutaways*.png`,
  `mask_transparency_player.png`, `mask_transparency_cursor.png` - the wall-fade shape when a player walks
  behind a building. Mod walls get cutaway behavior by matching tiledef properties + these shared masks.
- `tileDepthTextureAssignments.txt`, `tileGeometry.txt`, `depthmaps/`, `lightmaps/`, `heightmaps/`,
  `geomTextures/` - per-tile depth/geometry/light data for the isometric renderer (jumbo trees get their own
  `.tiles` + `.txt` maps).
- `effects/*.txt` = flat key-value parameter files (`ProjectileRed`, `ProjectileLength`,
  `ProjectileFadeRate`...) - bullet tracers per caliber and similar particle effects are *data, not code*.
- `gibs/`, `binmap/`, `binary.dat`, `seams.txt` - gore chunks, binary map lookup, chunk-seam data. Mostly
  engine-owned; `seams.txt` is the road/zone seam table.

### Radio and TV broadcasts (`media/radio/RadioData.xml`)
One XML file drives every broadcast channel:
- `<Voices><VoiceEntry ID="guid"><FinalColor r g b/></VoiceEntry>` - speaker colors.
- `<Adverts>` and channel `<ScriptEntry>` blocks: `name`, `startdelay`, `timestampmode` (Static/relative),
  `loopmin/loopmax`, containing `<BroadcastEntry>` items with `timestamp/endstamp`, `type`
  (`ActivateBroadcast`, line types), `day` (day-indexed schedules - that's how broadcasts "run out" over the
  apocalypse) and per-line `<LineEntry>` voice references.
- Combined with `media/lua/server/radio/ISDynamicRadio.lua` (dynamic channels - the weather channel reads the
  ClimateManager) and `client/RadioCom/` UI. A custom channel = a ScriptEntry + translations (`RadioData.json`,
  `DynamicRadio.json`).
- `media/profanity/` = the word filter lists.

### Recorded media (CDs, VHS tapes, broadcasts-on-disc)
`media/lua/shared/RecordedMedia/recorded_media.lua` - generated Lua table:
```lua
RecMedia["<guid>"] = {
  itemDisplayName = "RM_<guid>", title = "RM_<guid>", subtitle = nil,
  author = "RM_<guid>", extra = "RM_<guid>",
  spawning = 0,                 -- loot-table weight flag
  category = "CDs",             -- CDs, VHS, ...
  lines = { { text = "RM_<guid>", r, g, b, codes = "BOR-1" }, ... },
}
```
All display strings are `RM_<guid>` translation keys (see `Translate/EN/` `RM`/Print media files), each
spoken line has an RGB color and a `codes` field (broadcast/effect codes like `BOR-1`, `LEAR-` for skill
broadcasts). `media/lua/shared/RecordedMedia/ISRecordedMedia.lua` plays them; items link by media GUID. A mod
adds `RecMedia` entries + translation keys + an item with the media id - instant custom CD/VHS.

### Dynamic music
`media/lua/client/Music/MusicChoices.lua` + `media/music/` + `examplemod`'s `MusicChoices.lua` pattern:
tracks are scored against a `drama` value (`SadisticMusicDirector` computes it from
`getNumVisible/ChasingZombies`), picked by `compareByDrama`/`compareByAmbientDrama` comparators, with a
last-played penalty. `media/music/luatriggered/` = cue tracks fired from Lua; the menu theme/preface are
replaceable files (`theme2.ogg`, `preface.ogg`) - the examplemod literally swaps the title theme by shipping
same-named files.
- `voiceStyles/` and `media/videos/` (Bink `.bik`) = survivor voice sets and in-game video (TV uses these).

### Fonts and text
`media/*.fnt` + `*_0.png` = AngelCode bitmap fonts (`mainfont.fnt`, `zombienewtext*`, `zombiecodetext.fnt` -
the UI/console faces). `media/font|fonts/` = TTFs. `getText*` renders with them; a mod replaces glyphs by
shipping an updated `.fnt`+PNG pair (e.g. for CJK mods - check how vanilla's CN/JA fonts ship separately).
`zombie/text/` (19 classes) handles localization mechanics; translation JSONs are `media/lua/shared/
Translate/<LANG>/*.json` (LANG dirs seen: AR CA CH CN CS DA DE EN ES ES_CL ...).

### Misc small formats
- `WorkshopTags.txt` - the allowed tag list for `workshop.txt` `tags=`.
- `SpritePaddingSettings.xml`, `inventory/`, `ui/` (XUI widgets), `newuitests/` - UI skin/test data.
- `gamecontrollerdb.txt` - SDL controller mappings (add your pad).
- `jumbo_trees*.tiles` - multi-tile tree sets with matching `.txt` descriptors.
- `mask_transparency_*` - see cutaways above.

## Build steps
1. Tracer/effect tweak: copy `effects/bullet_tracer_effect_base_bullets_9mm.txt`, edit the key-value params.
2. Custom broadcast channel: ScriptEntry in `media/radio/` (mod path mirrors), voice GUIDs + colors, lines in
   `Translate/EN/RadioData.json`.
3. Custom CD: `RecMedia["<new-guid>"]` entry + `RM_` keys + item script referencing the guid.
4. Custom music: `media/music/*.ogg` + a `MusicChoices`-style shared table hooked into the drama picker, or
   `luatriggered/` + `playerVoiceSound`-style trigger calls.
5. Sittable furniture: `seating.txt` entry per tile + the tile's `sittable` properties in tiledef.
6. 3D-on-tile: `spriteModels.txt` entry + `model` script for the mesh.

## Verification
- Radio: tune a radio/TV in game; dynamic channel code is in `server/radio/` so `console.txt` errors point at
  your ScriptEntry.
- RecMedia: play the item on a boombox/TV; lines without matching `RM_` keys print the raw key.
- Music: `console.txt` MusicDebug lines + the `-debug` music panel.
- Tile attachments: place the object via `-debug` tile picker and walk around it.
- Not verified: mod shader overrides, custom `.fnt` on non-Latin scripts, `codes` values beyond the stock set.

## Gotchas
1. **Anim file dropped silently.** The `animNodeFilter` in the AnimSet root (`Bob`/`Kate` prefixes,
   `animationsMesh` folder) gates what `anims_X` files get scanned - wrong prefix = file never loads, no error.
2. **`animscript/combat.xml` is a different language** from AnimSet animNodes - `<PlayAnim>`/`<Out>` states vs
   `m_Conditions` nodes. Copy the right one.
3. **RecMedia strings must be `RM_<guid>` keys** or the game prints raw keys on screen; `spawning` flag
   controls loot eligibility.
4. **Broadcast `day` indexes schedule across apocalypse days** - entries outside the world's day range never
   air; `loopmin/loopmax` repeats.
5. **Shader files are version-mangled at load** (330->120 rewrite on old GL) - a mod shader must follow the
   same dual-syntax style to run everywhere.
6. **Seats need BOTH the tiledef sittable properties AND a `seating.txt` anchor** - one without the other =
   chair you can't sit in.
7. **Radio voice colors live in `VoiceEntry` GUIDs** - a LineEntry referencing an unknown voice falls back;
   register voices per channel.
8. **`codes` on media lines aren't decorative** - `LEAR*` codes grant skill XP (that's how life-living TV
   works); wrong/missing codes = flavor-only media.
9. **Fonts are bitmap atlases** - a `.fnt` without its `_0.png` (or wrong glyph page) renders boxes; CJK needs
   the dedicated font set, not a patched Latin one.
10. **`luatriggered` music never plays by itself** - something must call the trigger; check `MusicChoices`/
    how vanilla triggers combat cues.
11. **Effects files are hot data** - bad float formats (missing digits) fail the parse silently; keep the
    `Name=0.x` style exactly.
12. **Videos are Bink (`.bik`)** - ffmpeg can't emit it natively; reuse stock videos or use RAD's own tools
    (licensed) if you truly need new TV video.

## Assets
- Music/VO: `.ogg` Vorbis; voice sets in `voiceStyles/`.
- Fonts: Hiero/AngelCode `.fnt` + PNG atlases.
- Video: Bink only.

## Open questions
- Mod shader loading (override-by-path plausible, untested).
- Full `codes` vocabulary on media lines (BOR/LEAR/...).
- Whether `animscript/` still drives live combat states in 42.21 or is vestigial.
- `.pzby`, `binmap`, `binary.dat` field layouts (binary, undecoded - see world-sim note).
