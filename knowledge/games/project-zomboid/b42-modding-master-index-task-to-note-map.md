---
kind: game
title: 'Master index: every Project Zomboid modding task mapped to the right notes, APIs and recipes (Build 42)'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); index of the 25-note B42 docset'
platform: windows
engine: java
route: loader-api
tools: ["the b42-* note series in this KB"]
anti_cheat: 'none'
status: in-progress
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Modding"
tags: [b42, index, master, map, recipes, tasks]
---
# Master index: task -> notes -> APIs (Build 42 docset)

> The reading-order note. Twenty-one deep notes exist; this one answers "where do I start for X" for
> every request class - from "add an item" to "port a whole enemy from another game" to "build a
> total conversion". Read the row for your task, then the notes it names; each note declares what was
> jar-verified vs in-game-verified vs untested.

## Setup
- Requires only the notes in this KB (`knowledge/games/project-zomboid/b42-*`); each note stands alone
  but assumes the anatomy + runtime foundations.

## Route and why
Loader API throughout - the docset documents the stock mod route; this index is purely navigational.

## How the game works
Covered per-layer in the notes; this file doesn't teach the game, it points at the note that does.

## Build steps
Read the task row -> the named notes -> their Build steps. A generic order is below.

## The map by task

### Creatures & characters
| I want to... | Read | Key APIs/classes |
|---|---|---|
| Port a whole enemy from another game (humanoid or animalistic) | `b42-porting-creatures-from-other-games` then the chassis note | zombie `dressInPersistentOutfitID` / animal `addAnimal` |
| Make a hostile monster that hunts players | `b42-custom-monsters-dressed-zombie-route` | IsoZombie + invisible whole-body clothingItem + AnimSets/zombie |
| Make an animal/insect/pack creature | `cdda-enemy-pack-ants-...` (animal chassis) + `b42-porting-creatures` | AnimalDefinitions, breeds, `goAttack`/`fleeTo` |
| Make a friendly NPC / companion / talking survivor | `b42-creatures-beyond-zombies-animals` then `b42-survivor-npc-frameworks` (Bandits/ALife anatomy) | `setVariableBoolean` identity + ZA task API + `setBumpType` locomotion |
| Make a turret/mimic/stationary creature | same note (thumpable chassis) | IsoThumpable `Thump`/`WeaponHit`/`changeSprite` |
| Make a multi-part boss / swarm / rider+mount | same note (hybrids) | N synced objects + shared ModData HP |
| Reskin/rewrite the player (new identity, voice, traits) | `b42-character-challenge-gameplay-hooks` | `SurvivorSwap.applyCharacter` |
| Add a custom skill/perk/trait/profession | `b42-custom-skills-perks-traits` | `PerkFactory.AddPerk` + `CustomPerks.initLua` + script blocks |
| Custom animations for anything | `b42-animation-internals` + `b42-presentation-edge-cases` | AnimSets + animNodes + `setVariable` + AnimEvents |

### Items, objects & systems
| I want to... | Read | Key APIs/classes |
|---|---|---|
| Add items/recipes/weapons/food/clothing/vehicles/traits... | `b42-script-system-items-recipes` (the census) | script blocks in `media/scripts` |
| Build a workstation (furnace, drying rack, alchemy, fluid tank) | `b42-entity-system-gameentities` | `entity` blocks + crafting/fluids/logistics components |
| Make a persistent interactive machine/system | `b42-interactive-world` + entity note | `SGlobalObjectSystem` / `entity`+`LuaComponent` |
| Custom liquids, poison drinks, fuel networks | entity note (fluids/components) | `FluidContainer` API, `ResourceChannel` |
| Modify hunger/metabolism/temperature/exercise | `b42-body-simulation-survival` | `Nutrition`/`Metabolics`/`Thermoregulator`/`Fitness` |
| New needs/moodles/illnesses | same note (needs section) | `Stats` + ModData + fake-moodle UI |
| Farming/fishing/foraging/traps/power/water | same note (loops table) + forage schema | `CFarmingSystem`/`forageSystem` data |
| Custom loot/loot tables/world stories/zone types | `b42-world-simulation` | `Distributions`, `Randomized*`, `register*Zone` |
| World events, helicopters, drama pacing | same note | `getNumChasingZombies`, SadisticAIDirector |
| Custom biomes/wilderness/procedural world | `b42-map-files-formats` + world-sim | `worldgen` Lua table + `WorldGenOverride.lua` |
| Maps, tiles, moveables, stashes, annotated maps | `b42-interactive-world` + map-files note | tiledef props, `StashUtil`, MapSymbol defs |

### Presentation & media
| I want to... | Read | Key APIs/classes |
|---|---|---|
| Custom models/textures/atlases | `b42-modding-anatomy` (assets) + porting note | `model`/`clothingItem` blocks, textureChoices |
| New sounds/SFX/voice/music | anatomy + presentation + weapons notes | `sound` blocks, `WorldSoundManager`, `playerVoiceSound` |
| Shaders, sprites on tiles, seating, cutaways | `b42-presentation-edge-cases` | shaders/, spriteModels.txt, seating.txt |
| Radio channels, recorded media, music cues, fonts | same note | RadioData.xml, RecMedia, MusicChoices |
| Custom UI windows/HUD/radials/minigames | `b42-ui-framework-ispanel` | ISCollapsableWindow + ISUI3DModel + XUI |
| Tracers, particles, screen feedback | `b42-weapons-ballistics-effects` | media/effects params, blood/gore calls |
| Audio systems (parameter clips, ambient, loops, voice) | `b42-audio-internals` | GameSound/FMOD params, ObjectAmbientEmitters, limiters |
| Weather/climate (storms, seasons, temperature, fog) | `b42-weather-climate-internals` | `ClimateManager`, `ThunderStorm`, `ClimateValues` |

### Combat & gameplay
| I want to... | Read | Key APIs/classes |
|---|---|---|
| New weapons/melee/ranged/throwables | `b42-weapons-ballistics-effects` + script census | item weapon fields |
| Custom damage types/attacks/armor-piercing | same note (damage funnel) | `DamageFromWeapon`/`DamageFromAnimal`/`hitConsequences` |
| Explosions/fire/incendiary | same note | `IsoFireManager.explode` |
| Distraction/sonar/noise mechanics | same note (WorldSound) | `WorldSoundManager.addSound` + `respondToSound` |
| Dismemberment/gore/dismembering hits | same note | `ModelInstance.dismember`, `splatBlood` |
| Custom ballistics/aim assists | same note | `BallisticsController` getters |
| Port a whole vehicle | same note (vehicle recipe) | `VehicleScript` + `BaseVehicle` (587 methods) |

### Structure, multiplayer & workflow
| I want to... | Read | Key APIs/classes |
|---|---|---|
| Ship a mod (folders, mod.info, Workshop) | `b42-modding-anatomy` | `media/` layout, workshop.txt |
| Understand events/timed actions/outfits/spawns | `b42-lua-runtime-events` | Events.*, ISTimedActionQueue |
| Make it work in multiplayer | `b42-multiplayer-internals` + runtime MP section | `sendClientCommand`/`transmit*`/ownership rules |
| Custom game mode/challenge/scenario | `b42-character-challenge` | `addChallenge`, sandbox writes |
| Depend on another mod / expose an API | `b42-mod-interop` then `b42-content-pack-scenario` (4 shipped architectures) | `require=`, shared globals, namespacing |
| Patch/extend another mod, or ship a scenario pack | `b42-content-pack-scenario` | `PrependFunction` wraps, ModData bridges, `getModFileReader` + custom formats |
| "Does the engine expose X?" | `b42-lua-callable-java-cheat-sheet` | javap-verified method clusters |
| Iterate/test/debug fast | `b42-modder-toolchain` | -debug, 32 debug windows, console.txt, um win |
| Reverse-engineer a behaviour | cheat-sheet workflow + toolchain | javap -> Umbrella -> ZomboidDecompiler |
| Total conversion (UI+mode+systems+creatures) | the column above, top to bottom | composition of the docset |

## Reading order for a first mod
1. `b42-modding-anatomy` - how a mod is shaped.
2. `b42-script-system` - the data layer.
3. `b42-lua-runtime` - the behaviour layer.
4. Your task's row above -> the deep note.
5. `b42-modder-toolchain` before your first in-game test.

## Verification
Every row in the task tables cites a note that passed `um kb check`; the index itself was generated
against the live docset (21 b42-* notes on 42.21.0). The ledger below is the docset-wide summary -
per-claim verification lives in each note's own Verification section.

## Honesty ledger
- **Jar/file-verified** (names/facts from `projectzomboid.jar` or shipped files): all class/method counts,
  method names, folder censuses, file anatomy.
- **In-game-verified** (running in a shipped mod or vanilla): the three creature provenance routes,
  state-machine choreography, `createRealZombieNow`, ModOptions, ModData, transmit* sync, debug suite.
- **Flagged-but-named** (open questions with the exact decompile target): binary cell schema
  (`IsoChunk`/`ChunkSaveWorker`), per-component entity grammar (`ComponentScriptFactory`), `AddPerk` int
  semantics, per-state entry conditions (`zombie.ai.states.*`), remote-debugger attach, dedi 42 flags.

## Gotchas
1. **Read the task row's notes in order** - each deep note assumes the anatomy/runtime foundations.
2. **Check a note's Verification section before trusting a method** - jar-verified names are real;
   untested call semantics are flagged inline, don't build on unflagged assumptions.
3. **The docset targets 42.21.0** - method/class names drift between 42.x builds; re-javap on upgrade.
4. **Multiple chassis solve the same "creature" differently** - pick by what the engine gives free
   (the porting note's decision table), not by what looks easiest at first.

## What's deliberately NOT here
Decompiled source, proprietary assets, workshop content without permission - the KB describes behaviour
in original words; your ports need licensed or generated assets (the porting note's legal section).
