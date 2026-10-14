---
kind: game
title: 'Custom humanoid monsters as dressed zombies: the Cry of Fear NPC mod anatomy (Build 42)'
game: Project Zomboid
games_also: ["Cry of Fear"]
game_version: 'Project Zomboid 42.21.0 (Steam, Windows 10); mod version 0.68.25, modversion 42.20+, Workshop item 3813406470'
platform: windows
engine: java
route: loader-api
tools: ["Project Zomboid Lua (Kahlua) mod API", "Steam Workshop (for source study)"]
anti_cheat: 'none (single-player focus; mod has partial MP support via the normal Lua networking API)'
status: in-progress
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links: []
tags: [b42, zombies, custom-monster, npc, clothing, bodylocation, animsets, animnode, animation-variables, skinned-mesh, spawning, persistent-outfit, multiplayer, case-study]
---
# Custom humanoid monsters as dressed zombies: the Cry of Fear NPC mod anatomy (Build 42)

> How the *Cry of Fear Monsters* workshop mod (23 monsters: Sawrunner, Faster, Taller, Citalopram, Drowned,
> Flygare, ...) actually works on Build 42: every monster is a **zombie wearing an invisible whole-body clothing
> item whose model is a custom skinned mesh**, driven by per-monster Lua controllers, custom AnimSet nodes keyed
> on animation variables, and a persistent-outfit identity system that survives the engine's zombie recycling.
> Anatomy study of Workshop 3813406470 (v0.68.25, `versionMin=42.20`) - read for technique, not copied. The
> alternative "monster = Build 42 animal" route is documented in `cdda-enemy-pack-ants-...md`; this note is the
> decision guide between them.

## Setup
- PZ **42.21.0**, Steam, Windows 10. Workshop item **3813406470** installed via normal subscription; source at
  `<SteamLibrary>\steamapps\workshop\content\108600\3813406470\mods\CryOfFearMonsters\{common,42}`.
- The mod ships with dense development comments (the authors log their own bug history in-file - worth reading
  directly; quotes below are paraphrased).
- 13k lines of Lua: `42/media/lua/shared/` (factory, outfits, identity helpers, net, sandbox-backed host
  settings), `client/` (one controller per monster + movement/presentation/spawning/vehicles/net), `server/`
  (`COF_ServerMonsters.lua`). Data: `common/media/{models_X,anims_X,clothing,scripts,sound,textures,
  fileGuidTable.xml,registries.lua}` and `42/media/{AnimSets,sandbox-options.txt,lua}`.

## Route and why
Loader API - nothing but the game's own Lua + asset pipeline. Two viable monster chassis in B42:

| | Zombie body (this mod) | Animal body (CDDA ants note) |
|---|---|---|
| Base object | `IsoZombie` with a custom clothing body | `IsoAnimal` (subclass of IsoPlayer) |
| You get for free | Hunt/attack AI vs players, hit reactions, deaths, ragdoll, crawling, fake-dead, vehicle collision, netcode | Pathing, eating, fleeing, growth/stages, butchering, saving |
| You must build | The whole look (body item, model, animset nodes) + identity + spawn system | The whole brain (attack AI, aggression) in Lua |
| Best for | Humanoid monsters that hunt players | Creatures with own behaviour, wildlife, packs |

CoF chose zombies because its monsters ARE humanoid hunters - the zombie state machine (target player, lunge,
bite, thump, stagger) is exactly the needed AI.

## How the game works (what the mod shows)

### The dressed-zombie chassis
- **One new BodyLocation**: `common/media/registries.lua` is a one-liner
  `ItemBodyLocation.register("COF:MonsterBody")`. At `OnGameBoot` a shared Lua file fetches the
  `BodyLocations` "Human" group, `getOrCreateLocation(ItemBodyLocation.get(...))` and
  `moveLocationToIndex` to the `UNDERWEAR_BOTTOM` slot index - putting the monster body at the
  lowest body-layer position so its skinned model replaces the zombie's visible body while normal
  clothing layers still render over it.
- **The body is a clothing item**: `media/scripts/COF_<Monster>Body.txt` defines an invisible
  `ItemType = base:Clothing`, `ClothingItem = COF_<Monster>Body`, `BodyLocation = COF:MonsterBody`,
  `WorldRender = false`, `hidden = true`, `Tags = base:noragdoll`. Because it occupies a body location the
  zombie wears, it replaces the visible body with its own skinned model.
- **The clothingItem XML** (`media/clothing/clothingItems/COF_<Monster>Body.xml`):
  `<m_MaleModel>/<m_FemaleModel>` point into `models_X` (`Sawcrazy/Sawcrazy`), one fixed `<m_GUID>`, all 16
  `<m_Masks>` slots set (hides the underlying body parts), `<m_Static>false`, one
  `<textureChoices>` atlas. A matching `...CorpseBody.xml` exists per monster so dead bodies keep the look.
- **`fileGuidTable.xml`** maps every clothing-item XML path to its GUID - required for clothing persistence;
  generate a v4-style GUID per file and never reuse.
- **Identity = outfit name, not mod data.** Each monster has a persistent outfit (`COF_Sawrunner`...) made of
  its body item. `z:dressInPersistentOutfit(outfit)` applies it; `z:getOutfitName()` reads it back;
  `z:isPersistentOutfitInit()` tells whether the game has actually dressed it yet.

### Why the identity system is the hardest part (their own bug history)
- The engine **recycles zombie objects**. A recycled zombie keeps its old `getOutfitName()` until re-dressed
  (which only happens when it comes into view). If controllers claim zombies by outfit name, a recycled zombie
  gets claimed by the WRONG monster's controller, then re-claimed - two controllers stripping each other's body
  item (naked zombie) and playing competing sounds.
- Mod-data marks are also unreliable: a monster that leaves and re-enters the loaded area can lose its mark,
  and unmarked types get spawned again (their log: "13 Sawcrazies and endless Human Flowers in an hour").
- Their fix (`COF_AIdentity.lua`): on `OnZombieCreate`, if `isPersistentOutfitInit()` is false, clear the stale
  `getHumanVisual():setOutfit(nil)` and immediately `dressInPersistentOutfitID(getPersistentOutfitID())` so the
  controller owns it before it is ever seen. The high half of `getPersistentOutfitID()` is the outfit index -
  a table learned from every dressed monster maps index -> monster name, so undressed zombies are identified
  before dressing.
- Per-frame dispatch on `Events.OnZombieUpdate`: every controller registered via `I.onUpdate(name, fn)` runs on
  each zombie each update; a weak-keyed `previousIdentity` table detects identity changes so old controllers
  get one cleanup pass (stop looping sounds, drop live tables) when a zombie is recycled into a different
  monster.

### Custom animations on the zombie animset
- `42/media/AnimSets/zombie/<state>/COF_*.xml` files merge into the vanilla zombie AnimSet: states seen include
  `attack`, `walktoward`, `idle`, `hitreaction`, `falldown*`, `getup`, `lunge`, `pathfind`, `eatbody`,
  `fakedead`, `bumped`, `vehicleCollision-*`, `corpseDragged*` - the full vanilla zombie state list is the
  template to mirror.
- Each `animNode` picks `m_AnimName` (a clip in `anims_X/<monster>/`) gated by `m_Conditions` on animation
  variables, e.g. `COF_IsSawrunner == true` AND `COF_SawrunnerAttacking == false`. Lua sets them:
  `z:setVariable("COF_IsSawrunner", true)` at claim time; attack loops toggle `COF_<M>Attacking`.
- Vanilla variables matter too: `ZombieBiteDone` gates bite application - controllers set it false at attack
  start and true to cancel; custom attack clips are selected by setting a `COF_<M>AttackClip` STRING variable
  that the animNode conditions read.
- `42/media/COF_AnimTemplates/` and `COF_SawcrazyTemplates/` are working copies of the vanilla AnimSet XML the
  authors keep in the mod for reference/generation - NOT loaded (only `media/AnimSets/` is).
- Animation files: `common/media/anims_X/<Monster>/COF_*.x` (text DirectX here; .glb/.fbx work too).

### Spawning
- Normal population path: `addZombiesInOutfit(x, y, z, count, outfit, femaleChance, crawler, ...)` dressed in
  the monster's outfit - works only when zombie population isn't None.
- Population-None path: `VirtualZombieManager.instance:createRealZombieNow(x, y, z)` then set everything by
  hand (`setFemaleEtc`, `dressInPersistentOutfit`, `setCrawler`, `setKnockedDown`, `setFakeDead`,
  `setGodMod`, `setSitAgainstWall`, `setHealth`). Two traps documented in-file: only callable server-side
  (`isClient()` early-out; MP clients get an empty list), and Java *fields* aren't reachable from Lua (an old
  version read `manager.choices`, got nil, spawned nothing - methods only).
- A whole spawn system sits on top: `COF_MonsterSpawns` (weighted rarity per monster, independent of zombie
  population), `COF_MonsterSites`, a Sawrunner "night event" (distant rev wakes sleepers, lurks, chase music,
  leaves at dawn), `COF_SpawnDiagnostics` for logging, and debug right-click test spawns.

### Sounds, UI, multiplayer
- `media/scripts/COF_<M>Sounds.txt`: `sound` blocks, `category = Zombie`, `is3D`, `distanceMin/Max`,
  `maxInstancesPerEmitter`, `loop`, multiple `clip {}` files per sound = random variants. Controllers
  start/stop loops per zombie; identity changes must kill loops (see dispatch cleanup above).
- Client-only features: `COF_DrownedStruggleUI`, `COF_HealthDisplay`, `COF_HostSettingsUI` - all under
  `client/`, none of it server-visible.
- MP support is explicit but bounded: `shared/COF_Net.lua`, `client/COF_NetClient.lua`, `COF_NetSync`,
  `COF_MultiplayerCorpses`, `server/COF_ServerMonsters.lua`; controllers early-return on `isServer()` and the
  factory early-returns on `isClient()`. "A player's game in multiplayer never creates zombies" - all creation
  is server-side, presentation syncs down.
- Sandbox options: `sandbox-options.txt` pages per rarity group (`None/Rare/Uncommon/Normal/High` enum via
  `numValues` + `valueTranslation`, booleans per creature), read via `SandboxVars.COF_Creatures.*` and mirrored
  into `COF_HostSettings` for MP host control.

## Build steps (your own dressed-zombie monster)
1. BodyLocation: `ItemBodyLocation.register("MyMod:MonsterBody")` in `common/media/registries.lua`; at
   `OnGameBoot` insert it into the Human group near `UNDERWEAR_BOTTOM`.
2. Model: skinned mesh in `common/media/models_X/<Monster>/` (`.glb`/`.fbx`/`.x`), clothingItem XML with all
   `<m_Masks>` 0-15 set + fixed GUID + `textureChoices` atlas in `media/textures/`; add the GUID to
   `fileGuidTable.xml`; corpse variant too.
3. Item: invisible clothing script block bound to `BodyLocation = MyMod:MonsterBody`; outfit XML/definition
   `MyMod_Monster` containing only that item.
4. AnimSet: copy the vanilla zombie state XML list, write `animNode`s keyed on `MyMod_Is<Monster>` variables;
   clips in `anims_X/<Monster>/` matching your `animationsMesh`.
5. Controller: `client/` Lua claimed via `OnZombieUpdate` dispatch keyed on `getOutfitName()`/`persistent
   outfit ID`; set `COF_Is*`-style variables at claim; drive attacks with `ZombieBiteDone` + attack-clip
   string variables; sounds via `playSound`/`emitter` on the zombie.
6. Spawner: server-side only - `createRealZombieNow` + `dressInPersistentOutfit` for population-None, or
   `addZombiesInOutfit` when the vanilla population is on.
7. Recycling safety: clear stale outfit on `OnZombieCreate` before the zombie is dressed; keep identity
   derivation from outfit, not mod data; give every controller a cleanup path on identity change.

## Verification
- The mod itself is the evidence (installed and its structure read end-to-end); its in-file comments document
  the user-visible symptoms of each failure mode (naked zombies, wrong sounds, runaway spawns).
- For your own build: `Zomboid/console.txt` after each session; `-debug` animator view to watch animNode
  selection; a debug context-menu spawn entry (their `COF_SpawnDiagnostics` pattern) to iterate.
- Not verified: dedicated-MP correctness of the net layer; whether `dressInPersistentOutfitID` timing changed
  after 42.21.

## Gotchas
1. **Recycled zombies carry stale outfit names** -> wrong controller claims them. Fix: on `OnZombieCreate`,
   clear `getHumanVisual():setOutfit(nil)` when not yet `isPersistentOutfitInit()`, re-dress from
   `getPersistentOutfitID()`.
2. **Mod-data marks don't survive chunk unload/reload reliably** -> duplicate spawns. Fix: identity from the
   outfit/persistent-ID (engine-side truth), count monsters via identity not marks.
3. **Two controllers on one zombie** -> body item stripped by the other controller = naked zombie playing the
   wrong sounds. Fix: single dispatch point, per-zombie previous-identity tracking, cleanup pass on change.
4. **Looping monster sounds survive the monster.** Fix: kill emitters/loops in the identity-change cleanup,
   not just on death.
5. **`addZombiesInOutfit` silently does nothing when zombie population is None.** Fix:
   `VirtualZombieManager.instance:createRealZombieNow` (server only) + manual property set.
6. **Lua cannot read Java fields** (`manager.choices` = nil). Only public methods. Fix: find the accessor; the
   game creates the square-in-list itself inside `createRealZombieNow`.
7. **Client-side spawning in MP spawns nothing** (or local phantoms). Fix: all creation server-side;
   `isClient()`/`isServer()` guards in shared files.
8. **Attack anims never trigger.** Vanilla bite only fires when `getShouldAttack()` passes (~0.72 tiles) and
   `ZombieBiteDone` state cooperates. Fix: drive custom clips through your own attack variables + a clip-select
   string variable, and set `ZombieBiteDone` yourself.
9. **Missing `fileGuidTable.xml` entry -> clothing loses identity across saves/MP.** Every clothingItem XML
   needs a unique stable GUID.
10. **Monster turns back into a normal zombie after reload.** Cause: body item removed but zombie persists.
    Fix: corpse body variant + controller re-asserts body item in its tick when identity holds.
11. **Whole-body look but vanilla hands/head poke through.** Fix: all 16 `<m_Masks>` entries set; check
    `m_AttachBone` empty and `m_Static=false` on the clothingItem.
12. **New animset replaces movement instead of adding to it.** (Same failure as the CDDA animal copy.) Fix:
    add nodes into `media/AnimSets/zombie/` keyed on variables; never replace the base set.

## Assets
- Meshes: `models_X/<Monster>/` skinned meshes + `textures/` atlases (textureChoices per clothingItem).
- Anims: `anims_X/<Monster>/` (this mod uses `.x`; `.glb` is the recommended modern path - see the anatomy
  note's animation section).
- Sounds: `sound/COF/<monster>/*.wav` bound by `sound` script blocks, category Zombie, 3D falloff tuned per
  alert/idle/attack/death; variants via multiple `clip {}` entries.
- The mod (by Svide, Workshop item 3813406470) uses Cry of Fear's original models/animations/sounds,
  credited to Team Psykskallar; no licence or permission is stated on the Workshop page, so treat the
  assets as unlicensed source-game content - study the structure, don't copy files.

## Open questions
- How much of the net layer is strictly needed for listen-server co-op vs dedicated.
- Whether a hybrid is viable: animal chassis with a zombie-hunter brain (animals can fight; CDDA note proves
  hostility, zombies-vs-animal NPEs documented there).
- AnimZed's release may obsolete the hand-written animNode XML workflow.
