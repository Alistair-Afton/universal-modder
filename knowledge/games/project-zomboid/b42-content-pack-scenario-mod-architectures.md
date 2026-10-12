---
kind: game
title: 'Content-pack and scenario-mod architectures: BanditsWeekOne outbreak sim, BeyondOneWeek function-patching, CDALifePatch bridges, .alife custom data formats'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); mods: BanditsWeekOne (3403180543), BeyondOneWeek (3405043636), ProjectALifeAftermath (3806063445), ProjectALifeJimmy (3806944055), CDALifePatch (3807738026), ALifeKnoxFractured (3813871509)'
platform: windows
engine: java
route: loader-api
tools: ["workshop items 3403180543/3405043636/3806063445/3806944055/3807738026/3813871509", "getModFileReader", "function-wrapping"]
anti_cheat: 'none'
status: in-progress
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://steamcommunity.com/sharedfiles/filedetails/?id=3403180543"
- "https://steamcommunity.com/sharedfiles/filedetails/?id=3405043636"
- "https://steamcommunity.com/sharedfiles/filedetails/?id=3813871509"
tags: [b42, scenario, content-pack, outbreak, radio, patching, wrapper, compat-bridge, custom-format, getModFileReader]
---
# Content-pack and scenario-mod architectures (42.21.0)

> The six scenario/content mods from the same download batch show four distinct architectures:
> **BanditsWeekOne** (a whole pre-outbreak society sim with custom radio/TV broadcasts), **BeyondOneWeek**
> (a mod that patches ANOTHER mod by wrapping its global functions), **CDALifePatch** (a pure
> cross-mod bridge), and **ALifeKnoxFractured** (a custom `.alife` data-file format read via
> `getModFileReader` - mods authoring their own file formats). Each is a different answer to "how do
> you ship content on top of a framework".

## Setup
- File inspection of the listed workshop items (all target 42.x; BWO/BOW ship `42.18`/`42.20`/`42`
  version dirs alongside `common/`).

## Route and why
Loader API - and for BOW, deliberately-intrusive Lua (documented as such by its own authors).

## 1. BanditsWeekOne: a scenario sim, not a skin pack
615 textures, 460 sounds, 230 Lua, 124 anims, 60 AnimSets, 50 script files, `sandbox-options.txt`,
per-version `registries.lua` - and three custom broadcast networks:
- `media/sound/Broadcasts/{BWOParty,BWORadio,BWOTelevision}/` + `OutputFilesoundtable.lua` - a
  full broadcast pipeline: staged `.wav` assets + a generated soundtable + radio/TV registration
  (the RadioData.xml/ISDynamicRadio layer from the presentation note, at production scale). The
  outbreak unfolding is told through in-world media.
- `registries.lua` per version dir = the framework-content handshake: the pack registers its
  NPC types/outfits/events into Bandits' runtime per game build (why version dirs exist - registry
  fields drift between builds).
- Gameplay content is *emergency-services simulation*: cops/SWAT/medics/firemen/hazmat callouts
  (`BWOEvents.CallCops/CallSWAT/CallMedics/CallFireman/CallHazmats`, `BWOPopControl` with per-service
  `On`/`Cooldown` knobs) - the week-one scenario is a scheduled event system driving the Bandits
  NPC framework, plus `findVehicleSpot2`-style emergency-vehicle spawning.

## 2. BeyondOneWeek: the mod-that-patches-a-mod pattern
Only ~9 Lua files, and they're surgical: `FDIntrusiveChanges.lua` opens with a banner that is the
pattern's own warning label - *"THIS SCRIPT CONTAINS OVERRIDDEN VERSIONS OF THE FUNCTIONS IN BWO"*.
The mechanics:
- `FDUtils.PrependFunction(BWOEvents.CallHazmats, chanceFunc)` - **wrapping another mod's global
  functions**: prepend a gate (`RandPer(50)`) so not every callout fires, or replace functions
  wholesale (`CallSWATReplace` reimplements the SWAT spawn with new odds and a debug chat line).
- `DynamicEpidemic.lua` (commented draft): scheduled wrapper application - after day 7, wrap
  `CallHazmats` to double `SandboxVars.BanditsWeekOne.HazmatCooldown` each invocation.
- The rules that make it survivable: (a) wrap, don't clone - `PrependFunction` composes around the
  original so updates pass through when the wrapper still matches the signature; (b) loudly flag
  override files (their banner) because they rot when the host mod updates; (c) gate with
  `SandboxVars` knobs rather than hardcoding; (d) `chanceFunc` composition beats if/else edits -
  your patch is a *decision layer* on top, not a fork underneath.
- This is the production answer to "can I tune another mod without forking it" - and also why the
  interop note says defensively re-read globals at call time: the global table is the seam both
  APIs and patches use.

## 3. CDALifePatch: the pure bridge
4-6 files, zero assets: `server/CDALifePatch.lua` + `CDALifePatchAlert.lua`,
`client/CDALifePatchAlertClient.lua`, `shared/CDALifePatch/Compatibility.lua`, sandbox options.
The shared file is a reference-grade bridge:
- `normalizeRelation` - maps one mod's relation vocabulary onto another's ("allied"->"friendly",
  "suspicious"->"careful").
- `isALifeBody` - detects the other mod's actors via **ModData marks** (`data.ProjectALifeOwned ==
  true`) with `hasModData()` nil-guards (avoids allocating tables on ordinary zombies) and pcall
  wrapping ("any error counts as not an A-Life body").
- `getOwnerStance(entity, owner)` - stance resolution with explicit "unknown" fallback, never
  `getPlayer()`-guessing.
Cross-mod communication is ModData flags + vocabulary translation + defensive probing - the exact
interop pattern, deployed between two NPC frameworks (CDDA ants <-> Project A-Life) so ants don't
get treated as ordinary zombies and faction relations translate cleanly.

## 4. ALifeKnoxFractured: a custom file format on `getModFileReader`
2 Lua files + `common/knox_fractured/{factions.alife,profiles.alife}` - the content is *data in a
mod-defined format*:
- `getModFileReader(modID, relativePath, false)` (verified used) - read arbitrary files out of your
  own mod directory at runtime; a `getModFileWriter` companion exists in the API surface.
- `ProjectALife.CatalogParser.parseText` parses `.alife` into catalog records; `Bootstrap.lua`
  deep-copies (`copy`) and registers them into `ProjectALife.Catalog` - "**Original additive catalog
  pack. No base records or save files are replaced.**" - a content mod that adds to a framework's
  catalog without touching its files.
- The architecture: framework ships parser+catalog (`alife_runtime`), packs ship `.alife` files +
  a bootstrap that registers them. New factions/profiles = new packs, no framework updates. This is
  the most elegant content-extensibility pattern in the batch - and generalizes: your own
  `.mypack` files + a parser + `getModFileReader` = user-authored content pipelines.
- `Pack.installed` guard + `Pack.MOD_ID`/`VERSION` fields - packs self-identify for interop.

## 5. The Aftermath/Jimmy variant packs
`ProjectALifeAftermath/{42,common/equipment}`, `...AftermathG93`, `...AftermathMFS`, `ProjectALifeJimmy`
(170+179 Lua, custom AnimSets/anims_X/ui/sandbox) - scenario packs on the same runtime: shared
`common/equipment` data + version dirs. Jimmy ships real content volume (per-NPC brains, anims, UI)
proving the runtime hosts multiple scenario mods simultaneously (framework + N packs, not one
monolith).

## The four content architectures, compared
| | Mechanism | When to use |
|---|---|---|
| Scenario sim (BWO) | content registers into framework + own event/broadcast systems | a campaign/world state layered on NPCs |
| Function-wrap patch (BOW) | `PrependFunction`/replace host mod's globals | tuning/extending an existing mod without forking |
| Pure bridge (CDALifePatch) | ModData flags + vocabulary translation, both directions | two frameworks must coexist safely |
| Custom format pack (Knox) | `.alife` files + `getModFileReader` + catalog registration | data-driven packs non-programmers can author |

## Gotchas
1. **Function-wrapping is load-order-sensitive** - your wrap must run AFTER the host mod's shared/
   files; use late-init (`OnGameTimeLoaded`) or `require=` if BOW-style patching (see interop note).
2. **Overrides rot silently** - the host mod updates, your copied function is stale; flag every
   override file (BOW's banner) and keep the wrapper thin.
3. **`getModFileReader` paths are relative to your mod root** - `getModFileReader("MyID", "data/x",
   false)`; a missing file returns nil reader, handle it (their `cannot open` path).
4. **ModData marks are the cross-mod identity standard** - `data.ModOwned = true` + `hasModData()`
   guards before `getModData()` (nil + allocation safe, per CDALifePatch).
5. **Vocabulary translation needs an unknown fallback** - never invent a stance; `unknown` is safer
   than wrong (their `getOwnerStance` contract).
6. **Broadcast content needs the soundtable too** - wavs alone don't broadcast; the OutputFile
   soundtable + radio channel registration complete the pipeline.
7. **Per-version registries exist because frameworks break builds** - BWO carries `42.18`+`42.20`
   registry copies; pin content to the builds you tested.
8. **Additive packs should never replace base records** - the Knox "no base records replaced"
   rule is what keeps N packs coexisting.
9. **`Pack.installed` guards** - shared files can double-load on save reload; idempotent bootstrap
   or you register content twice.
10. **`getSpeakColour`/`addLineChatElement`** exist for NPC speech text (BOW uses them for debug) -
    the chat-element API is how NPCs "say" things in-world.

## Verification
- Verified by file inspection: all file names, the `PrependFunction` pattern (verbatim from
  `FDIntrusiveChanges.lua`), the BWO broadcast folders + soundtable, `registries.lua` per version,
  CDALifePatch's `normalizeRelation`/`isALifeBody`/`getOwnerStance` (verbatim logic), Knox's
  `getModFileReader` + `.alife` + `CatalogParser.parseText` usage, `Pack.installed`, Aftermath
  `common/equipment`.
- Not verified in-game: running the packs, `getModFileWriter` companion API usage, `.alife` grammar.

## Open questions
- Full `.alife` schema (factions/profiles grammar - the parser source is in `alife_runtime`).
- BWO `registries.lua` field differences across `42.18`/`42.20` (version drift surface).
- `BWOPopControl` per-service knobs (`On`/`Cooldown` verified in BOW's wrapper) - full surface in
  BWO's server code.
