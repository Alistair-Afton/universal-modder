---
kind: game
title: 'Build 42 character & gameplay hooks: SurvivorSwap, perks/skills, body/health, challenges, explosions, render hooks'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); APIs verified in media/lua of the install'
platform: windows
engine: java
route: loader-api
tools: ["Project Zomboid Lua (Kahlua) mod API"]
anti_cheat: 'none'
status: in-progress
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Lua_(API)"
tags: [b42, lua, challenges, gamemode, survivorswap, traits, perks, skills, health, bodydamage, moodles, nutrition, voice, xp, explosions, render-hooks, characters]
---
# Build 42 character & gameplay hooks: SurvivorSwap, perks/skills, body/health, challenges, explosions, render hooks

> The hooks for "play as someone else", "change how the game is played", and "make your own game mode":
> the vanilla `SurvivorSwap` table (a full character-rewrite API TIS ships for debug), the perks/skills/stats
> surface, BodyDamage, moodles, nutrition/fitness, the challenge gamemode system (`addChallenge`), explosion
> and fire APIs, and where Lua can draw. This is the mod for total-conversion ambitions.

## Setup
- PZ **42.21.0** Steam/Windows. Sources: `media/lua/shared|client|server` of the install.

## Route and why
Loader API. Everything below is vanilla Lua calling exposed Java - the same things the "crazy" total-overhaul
workshop mods do.

## How the game works

### Rewriting a character: SurvivorSwap
`media/lua/shared/NPCs/SurvivorSwap.lua` is a TIS-shipped table that rewrites an `IsoPlayer` wholesale -
it powers the debug "Survivor Swap" context menu and is the exact recipe for a "play as <anyone>" mod:
```lua
SurvivorSwap.applyCharacter(playerObj, data)  -- fields:
-- forename, surname, profession, traits = {...} (clears + sets known traits)
-- perks = { [Perks.X] = level, ... }  via playerObj:setPerkLevelDebug (Strength/Fitness default 5)
-- weight (nutrition), female, voiceType, voicePitch, voicePrefix (auto VoiceMale/VoiceFemale)
-- hairColor/beardColor (ImmutableColor), skinTexture index, beard, hair model, then resetModel()
```
`SurvivorSwap.Survivors`/`Loadouts`/`Vehicles` are data tables you extend; entries show under the debug menu
and the function works from any mod Lua - the whole body/identity of a player is writable at runtime.

### Traits, perks, XP, stats
- Trait read: `playerObj:getCharacterTraits():getKnownTraits()` (or `playerObj:HasTrait("id"` older API) -
  write `traits:clear()` + `traits:add("base:x")` as SurvivorSwap does.
- Perks: `Perks.<Name>` enum values + `Perks.getMaxIndex()`/`Perks.fromIndex(i)` iteration +
  `PerkFactory.getPerk(perk)`. Levels via `playerObj:setPerkLevelDebug(perk, n)` and
  `playerObj:getPerkLevel`. **Custom skill**: `PerkFactory.AddPerk` exists in the Java API (10-int XP
  curve + parent/passive variants) and `media/perks.txt` (`perk Name { parent, translation, passive,
  xp1 ... xp10 }`, loaded by `CustomPerks`) is the data path - see the custom-skills note.
  (community mods added skills in B41 - verify the exact signature in Umbrella stubs / javap; the perk list
  is built early, so register at `OnGameBoot` before the char screen).
- XP: `playerObj:getXp():AddXP(perk, amount)` (+ `getXPGlobal`, multipliers); `client/XpSystem` +
  `server/XpSystem` folders are the Lua side. `ISPerkLog` logs gains.
- Stats: `playerObj:getStats()` exposes moodle-adjacent live values (hunger, thirst, fatigue, drunkenness,
  `getNumVisibleZombies()`/`getNumChasingZombies()` - the drama fields, see world-sim note).
- `getDescriptor()` = name/profession/voice; `getHumanVisual()` = appearance; `resetModel()` re-builds the
  avatar after edits.

### Body, health, moodles
- `playerObj:getBodyDamage()` - per-part health (`BodyParts` enum - **the part list is a fixed enum; you
  cannot add new body parts**, animals have their own simpler system), `DamageFromAnimal`, infections,
  `setInfected`, fractures, bandages, `ReduceGeneralHealth` etc.
- **Moodles** (`Moodles`/`MoodleType`) are a fixed enum too - custom moodle *types* can't be registered;
  mods fake them with a custom UI icon (ISUI) + a value in mod data.
- `getNutrition()` - weight, calories, lipids/proteins/carbs (SurvivorSwap sets weight directly).
- `getFitness()` + `shared/Definitions/FitnessExercises.lua` - exercise definitions are Lua data (custom
  exercises possible).
- Knockdown/drunk-trip: the bump sequence from the CDDA note (`setBumpType("stagger")`, `setBumpFall(true)`,
  `setBumpFallType`, `reportEvent("wasBumped")`); `playerVoiceSound("PainFromBite")` for voice lines.

### Explosions, fire, destruction
- `IsoFireManager.explode(getCell(), square, radius)` - the vanilla challenge `AReallyCDDAy` blows up a
  random room at spawn with this one call. `IsoFireManager` also owns `StartFire`/`startSmoke`.
- Structures: `sledgeDestroy`, `IsoThumpable` objects take damage; fire spread is `zombie.iso`'s fire system
  (`client/FireFighting`/`server/FireFighting` Lua).

### Custom game modes: challenges
`media/lua/client/LastStand/` - challenges are Lua tables registered by `addChallenge(table)`:
```lua
CDDA = {}; CDDA.Add = function() addChallenge(CDDA) end
CDDA.OnGameStart = function() -- runs on new game with this challenge
    local pl = getPlayer(); ... IsoFireManager.explode(getCell(), tile, 100000)
end
CDDA.OnInitWorld = function()
    SandboxVars.Zombies = 1; SandboxVars.DayLength = 3; SandboxVars.StartMonth = 12; ...
end
```
Hooks seen: `OnGameStart`, `OnInitWorld`, plus the hourly/daily tick and player-spawn variants (read
`WinterIsComing.lua`, `28MinutesLater.lua`, `TopOfTheWorld.lua` for the full set). Each ships a same-named
`.png` thumbnail. Sandbox vars are just overwritten - a challenge is a scripted scenario, not a mode flag.
`LastStandPlayerFile.lua` handles save naming.

### Render/input hooks for crazy UI/world effects
- `getRenderer()` returns the Java renderer (used by vanilla `ISBuildingBlueprintManager.RenderUI` to blit
  sprites to world coords via `ISCoordConversion.ToWorld/ToScreen` - that file is also the canonical
  screen<->world coordinate example).
- UI render events (`OnPostUIDraw`/`OnPreUIDraw`-family) draw custom overlays; ISUI panels do everything
  custom HUDs need. `OnPostRender`-style events exist for world-space drawing (check Umbrella for exact names
  per build).
- Input: `shared/keyBinding.lua` (bindable keys), `Events.OnKeyPressed`/`OnCustomUIKey`,
  `gamecontrollerdb.txt` for pads, `JoyPad/` client folder.
- Screen readers: `getMouseX/Y` + `ISCoordConversion` = aim-anywhere custom interactions.

### Global object systems (persistent interactive objects)
`client/Map/CGlobalObjectSystem.lua` + `server/Map/SGlobalObjectSystem.lua` + `MapObjects/` - the
client/server pair behind objects that persist and sync (rain collectors, generators, etc. are built on
this GlobalObject framework). For "a new kind of placed machine that keeps running", subclass an
SGlobalObjectSystem/CGlobalObjectSystem pair instead of hacking IsoObjects.

## Build steps
1. Play-as mod: extend `SurvivorSwap.Survivors` entries or call `applyCharacter` on `OnGameStart`.
2. Challenge mod: `client/LastStand`-style table + `addChallenge` + thumbnail png + hooks.
3. Custom skill: `PerkFactory.AddPerk` at `OnGameBoot` (verify signature), translation `Perk_` keys, wire XP
   gains through `getXp():AddXP`.
4. World device: SGlobalObjectSystem pair; study `MapObjects/` + the rain-barrel/generator Lua.
5. Explosion/setpiece: `IsoFireManager.explode` + timed via `ISTimer` (luaexamples).

## Verification
- Challenges appear in the game-mode picker with their png; `console.txt` shows hook errors.
- SurvivorSwap entries show under the debug Survivor Swap context menu immediately.
- Perk changes reflect in the char screen/skills panel; `ISPerkLog` prints XP.
- Not verified: `PerkFactory.AddPerk` exact call shape on 42.21 (Java signature - check Umbrella/javap
  before building a skill mod); GlobalObjectSystem subclassing end-to-end.

## Gotchas
1. **Body parts and moodle types are fixed enums** - you can damage/heal existing parts and fake moodles with
   UI, not add new ones.
2. **`setPerkLevelDebug` works but is "debug"-named** - it persists correctly; the name is a warning, not a
   limitation.
3. **Traits added at runtime need the char screen refreshed** - SurvivorSwap calls
   `getPlayerInfoPanel().charScreen.refreshNeeded = true`; skip it and the UI shows stale traits.
4. **Challenges can't "inherit" vanilla scenario setup implicitly** - write `OnInitWorld` completely; missing
   sandbox vars fall to defaults, not the preset the player picked.
5. **`explode` is square-based and huge-radius-capable** - the vanilla call uses radius 100000; clamp yours.
6. **GlobalObject systems must be registered on both sides** - S* on server, C* on client, paired by name;
   one-sided registration = object works in SP only.
7. **`resetModel()` required after visual edits** - hair/skin/outfit changes don't redraw until called.
8. **XP added on the client in MP desyncs** - server-authoritative since ~42.13; grant XP server-side.

## Assets
- Challenge thumbnails: `client/LastStand/<name>.png`.
- Voice: `voiceStyles` + `desc:setVoicePrefix/Type/Pitch` pick from existing styles; new voice styles are data
  files (see presentation note).

## Open questions
- Exact `PerkFactory.AddPerk` contract on B42 (arguments, when perks serialize, MP sync).
- Whether `SurvivorSwap.Loadouts`/`Vehicles` have engine consumers beyond the debug menu.
- GlobalObjectSystem: which lifecycle hooks are required vs optional (documented only by reading the rain
  barrel/generator Lua deeply).
