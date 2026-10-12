---
kind: game
title: 'Build 42 modding anatomy: mod folders, media assets, Lua, scripts, tooling and Workshop publishing'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); structure verified against the install and two live mods (Workshop 3813406470, a local animal mod)'
platform: windows
engine: java
route: loader-api
tools: ["Project Zomboid Lua (Kahlua) mod API", "PZ-Umbrella EmmyLua stubs", "demiurgeQuantified/ZomboidDecompiler (Vineflower)", "javap", "TileZed/WorldEd community forks", "Blender + Paddlefruit rig / DirectX X importer"]
anti_cheat: 'none. Single-player is unprotected; no client patching exists or is needed - mods are Lua + data loaded by the game itself. Hosted dedicated servers run the same mod set, so stay out of servers you do not control.'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Modding"
- "https://pzwiki.net/wiki/Mod_structure"
- "https://github.com/PZ-Umbrella/Umbrella"
- "https://github.com/demiurgeQuantified/ZomboidDecompiler"
- "https://www.projectzomboid.com/modding/"
tags: [b42, mod-structure, lua, kahlua, scripts, animsets, models, sounds, sandbox-options, translations, texturepacks, maps, multiplayer, workshop, tooling, orientation]
---
# Build 42 modding anatomy: mod folders, media assets, Lua, scripts, tooling and Workshop publishing

> An orientation note: how a Project Zomboid Build 42 mod is laid out on disk, what every `media/` subfolder does,
> how the Lua (Kahlua) API surface works, which community tools to reach for, and how a mod gets onto the Steam
> Workshop. Verified against a 42.21.0 Steam install plus two real mods (a 23-monster NPC pack and a custom
> animal). Pair with the deeper notes: `b42-20-modlist-preset-and-world-load-errors.md` (loader/preset mechanics),
> `cdda-enemy-pack-ants-...` (custom animals + custom tiles), and the zombie-outfit monster note in this folder.

## Setup
- Project Zomboid **42.21.0** from Steam (`steam_appid.txt` = 108600), Windows 10. Game build and revision are in
  `%USERPROFILE%\Zomboid\version.txt` (`42.21.0 4a0e9546ec`).
- Install: `<SteamLibrary>\steamapps\common\ProjectZomboid\`. Key entries: `projectzomboid.jar` (all game code,
  Java), `jre64\` (bundled JRE), `media\` (all vanilla data - readable, the reference for every format),
  `Workshop\ModTemplate\` (official upload template), `mods\` (install-local mods).
- User data: `%USERPROFILE%\Zomboid\` - `mods\` (per-user mods, checked in the Mods screen), `Workshop\` (upload
  staging), `console.txt` (current/last session log; holds several sessions - read from the last
  `loading <modid>`), `Logs\`, `Saves\`, `Lua\pz_modlist_settings.cfg` (mod presets), `db\` (save DB).
- Workshop downloads: `<SteamLibrary>\steamapps\workshop\content\108600\<workshopId>\mods\<modFolder>\`.
- No anti-cheat, no loader to install. The game IS the mod loader. `-debug` on the launch options enables the
  debug menu, Lua file reload (F11-style hot reload for anims/scripts) and extra logging.

## Route and why
Loader API, always. PZ mods are data + Lua loaded natively by the game; there is no BepInEx/SMAPI-style loader
and none is needed. Java bytecode patching is technically possible (the jar is unobfuscated) but is not
Workshop-distributable and almost never required - everything the community ships is Lua + media.

## How the game works (what we had to learn)

### Mod layout (Build 42)
```
Contents/mods/<modId>/
  common/media/...        assets shared by all game versions
  42/mod.info             one folder per supported build line (41/, 42/, 42.1/...)
  42/poster.png           mod-list thumbnail
  42/media/...            version-specific assets
```
- The game picks the version folder matching the running build; `common/` is always merged in. A mod with only
  `common/` works but cannot version-gate media. `mod.info` lives in the version folder in B42 (keep it
  lowercase `mod.info` for Linux).
- `mod.info` keys seen in the wild: `name`, `id`, `description=` (repeatable lines), `poster=`, `icon=`,
  `modversion`, `versionMin`, `require=`, `incompatible=\ModA,\ModB`, `loadModAfter/Before`, `pack=`,
  `tiledef=<file> <100-8189>`, `map=`, `models=`. Full list: pzwiki `mod.info` / ScriptsDocs.
- Lua overrides by relative path: a mod file at `media/lua/shared/<same path as vanilla>` replaces the vanilla
  file (load order between mods decides conflicts). Name your own files uniquely and put them in a subfolder
  named after the mod (`media/lua/shared/MyMod/...`).

### The media/ subfolders (what goes where)
| Folder | Contents |
|---|---|
| `lua/{shared,client,server}` | Lua. `shared` loads first (boot, both sides), then `client` or `server`. SP loads client+server; MP client loads shared+client; MP dedicated loads shared+server. |
| `scripts/` | `.txt` script files in the `module X { item Y { ... } }` block syntax: items, recipes, models (`model`, `animationsMesh`), sounds (`sound`), vehicles, fixing, evolution recipes. Modules: `Base` or your own; `imports { Base }` reuses vanilla blocks. |
| `models_X/` | Meshes. `Skinned/` for rigged; `.x` (text DirectX), `.fbx`, `.glb` all load. |
| `anims_X/` | Animation files (`*.x`, `.fbx`, `.glb`) in folders per `animationsMesh` entry. |
| `AnimSets/<name>/<state>/<file>.xml` | Animation state graph XML (see below). Replaces/adds nodes to an entity's AnimSet by path. |
| `actiongroups/` | ActionGroup XMLs: the higher-level state machine that picks AnimStates. |
| `clothing/clothingItems/*.xml` | Clothing item definitions: model path, GUID, textureChoices, masks, attachment bone. |
| `fileGuidTable.xml` | One `<guid>` per file that needs a stable identity (every clothing item XML). Vanilla uses it too; entries are `<path>` + `<guid>`. |
| `textures/` | PNGs: item icons (`Item_*.png`), clothing/body textures, UI. `Item_<name>.png` auto-binds to `item <name>` icons. |
| `texturepacks/*.pack` | Custom tile sprite packs (format documented in the CDDA note). |
| `sound/` | Audio: `.ogg`/`.wav`, bound by `sound` script blocks. |
| `sandbox-options.txt` | Custom sandbox options (`option Ns.Name { type, default, page, translation }`). |
| `lua/shared/Translate/<LANG>/*.json` | Translations: `IG_UI_`, `Sandbox_`, `Tooltip_`, `ItemName_`, `Recipes_` keys. |
| `maps/<mapName>/` | Map chunks (`.lotheader`, `.bin`, `.chunkdata_*`), `map.info`, `spawnpoints.lua`, `thumb.png`. |
| `music/`, `videos/` | Custom music tracks (see examplemod `MusicChoices.lua`) and Bink videos. |
| `ui/XogSatellite*` / `xui/` | UI textures / XUI layouts. |

### Lua (Kahlua)
- Kahlua is a Java Lua ~5.1. Lua calls exposed Java methods directly (`z:getHealth()`); Java *fields* are NOT
  visible - only public methods (this burned the CoF mod once; their comment says it plainly).
- Globals injected by the game: `getCell()`, `getWorld()`, `getPlayer()`, `ZombRand`, `Iso*` classes,
  `ArrayList`, `addZombiesInOutfit`, `instanceof`, `Events`, `SandboxVars`, `ModData`, `sendClientCommand` /
  `sendServerCommand`, `isClient()`, `isServer()`, `getText`, `require` (module path from media/lua root, e.g.
  `require 'COF_MonsterOutfits'` - `shared/` first).
- Events: `Events.OnGameBoot`, `OnTick`, `OnZombieUpdate`, `OnZombieCreate`, `OnClientCommand`,
  `OnServerCommand`, `OnLoad`, `OnSave`, `OnWorldSound`, `OnDeadBodySpawn`, hundreds more - Umbrella stubs list
  them all with signatures.
- Persistence: `ModData.getOrCreate(key)` global tables (plain data only) and `obj:getModData()` per-object.
  Neither auto-syncs in MP: `ModData.transmit/request` or manual commands.
- Multiplayer: since ~42.13 the server owns damage/item logic; write state changes server-side
  (`server/` folder or `isServer()` paths) and `sendServerCommand` to clients; clients `sendClientCommand` up.
  `Events.OnClientCommand.Add(fn)` on server, `OnServerCommand` on client. Args = plain data only.
- `stdlib.lua`/`stdlib.lbc` and all vanilla Lua ship readable in `media/lua/` - grep it before inventing an API.
  `media/luaexamples/` has timed-action/UI examples.

### Animations (the non-obvious stack)
- Three layers: **AnimSet** (per-entity set of AnimStates) -> **ActionGroup** (state machine over states) ->
  **animNode XML** (`m_AnimName`, `m_Looped`, `m_BlendTime`, `m_Conditions` on animation variables of types
  BOOL/STRING/float set via `character:setVariable("Name", value)`).
- Mod AnimSets under `media/AnimSets/<entity>/<state>/*.xml` merge into the named entity's graph; a node gated
  on a custom variable (`COF_IsSawrunner = true`) is how one zombie species gets a unique moveset.
- In B42 only `anims_X` subfolders named by the model's `animationsMesh` script entry are scanned; human rigs
  scan `Bob/` and `Kate/` with `Bob_`/`Kate_` filename prefixes.
- `.glb` (recommended, hot-reloadable, needs the Paddlefruit rig), `.fbx`, `.x` (vanilla, poorly supported by
  modern DCC). Blender: Paddlefruit community rig + `PY-BobBatchExportGLB.py` batch exporter, 30 fps. Import
  vanilla `.x` via the "DirectX X Importer" Blender extension or ZomboidAssetConverter. AnimZed (TIS's own
  animation tool) was announced for release after B42 stable.

### Reading the game code
- `projectzomboid.jar` is unobfuscated Java. Fastest: `javap -cp projectzomboid.jar -p <class>` (JDK, or the
  jdeps path used in the CDDA note) and a small reflection lister to enumerate what Lua can call.
- Best: `demiurgeQuantified/ZomboidDecompiler` (Vineflower + Rosetta parameter names) - point it at the install,
  get `output/source/zombie/...` readable decompiles. Keep output OUTSIDE any repo you publish.
- JavaDocs: `projectzomboid.com/modding/` (official, e.g. `zombie/network/GameClient.html`). Umbrella
  (PZ-Umbrella org) ships EmmyLua/LuaLS stubs generated from Rosetta data - IDE autocomplete for the entire
  API. `pz-lua-stubgen` regenerates stubs; `pz-rosetta-source` is the raw dataset.
- Lua API moves between 42.x builds (`IsoGridSquare.Is` -> `has` between 42.20 and 42.21). Re-verify method
  names per build.

### Workshop publishing
- Staging folder: `Zomboid/Workshop/<id>/{workshop.txt, preview.png (256x256), Contents/mods/<modFolder>/...}` -
  the install ships `Workshop/ModTemplate` as the example (includes a two-map mod with `map.info` and
  `spawnpoints.lua`).
- `workshop.txt`: `version=1`, `title`, repeatable `description=` lines, `tags`, `visibility=public|private`.
- Upload via the in-game Workshop menu (needs `-debug`? No - the vanilla "Workshop" main-menu entry handles
  create/update). Updating re-uploads the same folder; keep `id` stable.
- Tags that matter: `Build 41` / `Build 42` workshop tags drive filtering; `versionMin` in mod.info gates
  loading.

### Maps
- B42 cells are 256x256 tiles (B41: 300x300). `media/maps/<MapName>/` per map; map mods insert new cells or
  replace whole cells (no partial-cell edits). Official TileZed/WorldEd B42 builds unreleased at research time;
  use community forks (Alree's or "Community Edition" by Crater). Buildings need BuildingEd; loot zones/room
  defs via ItemZed-era docs + `.lot` files. TileZed also produces `.tiles` tiledefs and `.pack` texturepacks,
  but both formats are documented (CDDA note) and writable by hand.

## Build steps
1. `Zomboid/mods/<id>/42/mod.info` + `media/` skeleton; enable in the Mods screen. Or stage under
   `Zomboid/Workshop/<id>/Contents/mods/` to test the upload layout early.
2. Lua in `media/lua/shared|<server>|<client>/MyMod/`; translations in `shared/Translate/EN/*.json`; sandbox
   options in `media/sandbox-options.txt`.
3. Data content via `media/scripts/*.txt` blocks (items, models, sounds) - copy the vanilla block shape from
   `media/scripts/` in the install.
4. Launch with `-debug`; read `Zomboid/console.txt` (and `Logs/` for older runs) after every session; hot-reload
   scripts/anims from the debug menu during the session.
5. Package: copy to `Zomboid/Workshop/<id>/`, fill `workshop.txt` + `preview.png`, upload in game.

## Verification
- `console.txt` is the oracle for load errors: script parse errors, missing textures (`ERROR: texture`),
  Lua stack traces with mod path lines. Multi-session file - search from the last `loading <modid>`.
- Debug menu (`-debug`): Lua debugger, animator debug, iso-region/zombie debug views, item spawner.
- The reflection-name check from the CDDA note (every `obj:method(` call checked against a javap/reflection
  dump) still catches API drift the IDE stubs miss.
- Not verified here: dedicated-server play of every mechanic (the CoF mod's `isServer()/isClient()` split is
  documented but we did not host a server in this session).

## Gotchas
1. **Lua sees only methods, never Java fields.** Symptom: `attempted index ... of non-table: null`. Cause:
   reading `manager.choices` on a Java object. Fix: find the accessor (`getChoices()` style) or another
   exposed path; the CoF factory comment documents exactly this failure on `VirtualZombieManager`.
2. **Mod silently not loading.** Cause: `mod.info` in `common/` but no version folder, wrong folder name vs
   running build, or a hidden `.txt` extension. Fix: `42/mod.info`, check the Mods screen, show file
   extensions.
3. **Old errors in console.txt.** It accumulates sessions; always read from the last `loading <modid>` line.
4. **`require` resolves from `media/lua/` root including `shared/` prefix rules** - `require 'X'` finds
   `shared/X.lua` from any side, but a same-named file in `client/` shadows differently per context. Keep one
   file per name.
5. **Sandbox labels containing `%` break the options screen** (Java `String.format`). Write "percent".
6. **Icon not showing.** `Item_Foo.png` must sit in `media/textures/` AND match `item Foo`'s module scoping;
   non-Base modules want `Item_<Module>_<Name>.png` conventions depending on version - check how the vanilla
   icon path resolves for your module.
7. **MP-only spawn bug pattern**: client-side code spawning zombies/items does nothing (or duplicates). Cause:
   `server/` Lua never runs on a client. Fix: spawn on server (`isServer()` or `createRealZombieNow` from
   server), transmit to clients.
8. **Overwriting vanilla files by accident.** Any mod file at a vanilla relative path replaces it
   (media/lua, AnimSets, scripts). Cause: generic filenames. Fix: mod-prefixed filenames and subfolders.
9. **Clothing item invisible/missing model.** Cause: `fileGuidTable.xml` missing the clothing XML's GUID entry,
   or `m_MaleModel` path not matching `models_X` layout. Every clothingItem needs a stable GUID.
10. **Anim node never plays.** Cause: `anims_X` folder name != `animationsMesh` entry's `animationDirectory`,
    or missing filename prefix (`Bob_`), or the animNode's conditions never true. Check
    `getAdvancedAnimator()` debug + the anim debug UI in `-debug`.
11. **Workshop update doesn't appear.** Cause: preview.png wrong size/format (256x256 PNG), or the mod moved
    out of `Contents/mods/`. The upload root is the folder containing `workshop.txt`.
12. **`Math.atan2` vs `math.atan2` portability**: Kahlua is 5.1-era; helpers written for Lua 5.3+ break.
    Alias what you use.

## Assets
- 2D/icons: any PNG pipeline; `Item_*.png` convention binds automatically.
- 3D: `.glb` for new work; import vanilla `.x` via Blender's DirectX X Importer or ZomboidAssetConverter.
- Audio: `.ogg` Vorbis preferred, `.wav` fine; `sound` script blocks define categories (Zombie, Animals, ...),
  3D falloff distances, max instances, loops, multiple `clip {}` variants per sound name.
- See the CDDA note for a full fal-generated pipeline (image->3D->decimate->weight to pig skeleton->.x text
  writer, ElevenLabs SFX normalized to vanilla loudness).

## Cost and time
Research session ~half a day: install + two live mods read end to end, pzwiki + tool repos surveyed.

## Open questions
- AnimZed and official B42 mapping tools release status (announced "after 42.20 stable hotfixes").
- Whether `.glb` hot-reload covers AnimSet XML too, or anims only.
- Server-authoritative rebalances since 42.13: the exact list of character/damage operations clients can no
  longer perform.
