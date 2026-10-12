---
kind: game
title: 'Porting whole creatures from other games into Build 42: three proven routes (original assets, converted assets, generated assets)'
game: Project Zomboid
games_also: ["Cry of Fear", "Half-Life", "Cataclysm: Dark Days Ahead"]
game_version: 'Build 42.21.0 (Steam, Windows 10); routes verified in three live ports: CoF Monsters 0.68.25, HLMonsters 0.1.0, CDDA Ants'
platform: windows
engine: java
route: loader-api
tools: ["Project Zomboid Lua (Kahlua) mod API", "jassimp64 (assimp model loader in the game)", "Blender + DirectX X importer", "Crowbar/pymeshlab for source formats", "fal (Meshy, ElevenLabs)"]
anti_cheat: 'none (single-player creature mods)'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://steamcommunity.com/sharedfiles/filedetails/?id=3813406470"
tags: [b42, creatures, monsters, cross-game-port, models, skinned-mesh, animations, animsets, ai, sounds, legal, houndeye, cry-of-fear]
---
# Porting whole creatures from other games into Build 42: three proven routes

> The synthesis note for "bring an enemy from game X into Zomboid". All three proven asset routes exist in the
> wild today: **Cry of Fear's monsters** ported with their *original models, animations and sounds*
> (Workshop 3813406470), **Half-Life's Houndeye** ported from HL's `.mdl` source assets through conversion
> (HLMonsters), and **CDDA's ants** reimplemented with AI-generated 3D + sound (no source assets at all).
> Pick a chassis (zombie vs animal), an asset provenance, and an AI strategy - the pieces are all documented.

## Setup
- PZ **42.21.0** Steam/Windows. Chassis notes this builds on:
  `b42-custom-monsters-dressed-zombie-route.md` (zombie chassis),
  `cdda-enemy-pack-ants-...md` (animal chassis + custom tiles + the full fal asset pipeline),
  `b42-modding-anatomy-...md` (formats), `b42-lua-runtime-...md` (behaviour).

## Route and why

### Step 1 - pick the chassis
| | Dressed zombie (CoF) | B42 animal (HLM/CDDA) |
|---|---|---|
| Source creature is | humanoid hunter | animal/quadruped/insect/pack creature |
| Free AI | stalk/attack players, grapple, stagger, crawl, fake-death, vehicle hits | walk/flee/eat/drink/grow/herd/milk/egg paths |
| Free visuals | Bob/Kate human skeleton + zombie animset | the host animal's skeleton+animset (`animationsMesh`) |
| Hard part | identity/recycling + per-monster anim conditions | the whole attack brain in Lua |
| Skeleton to skin to | `Bip01` (Bob/Kate human rig) | pig/turkey/deer... whichever fits the anatomy |
| Detail note | `b42-custom-monsters-dressed-zombie-route.md` | `cdda-enemy-pack-ants-...md` |

Rule of thumb that actually held: if the creature wants to *hunt the player like a zombie*, use a zombie
(the AI is free); if it wants *its own behaviours* (pack tactics, sonic blasts, colonies), use an animal and
write the brain.

### Step 2 - get the body: three proven asset provenances
**A. Original game assets (the Cry of Fear route).** CoF ships its monsters' *own* models/animations/sounds
with the porters' rights cleared (the mod credits the CoF asset licenses - you must do the same or you can't
publish). Layout: `models_X/<Monster>/` skinned meshes + `anims_X/<Monster>/` clips + atlas textures +
`sound/` wav banks. If the source game can export skinned meshes+anims to `.fbx`/`.glb`/`.x`, they load via
the game's bundled **assimp (`jassimp64`)** - `model`/`animationsMesh` script blocks take those formats
directly; no conversion to PZ's `.x` is required.
**B. Converted source assets (the Half-Life route).** HLMonsters's houndeye: HL `.mdl` -> decompile/convert
(Crowbar or similar to `.smd`/`.fbx`) -> skin to a PZ-compatible skeleton -> ship as text `.x`
(`media/models_X/Skinned/Houndeye.x`) + `media/anims_X/houndeye/he_*.x` clips + `animationsMesh` entry
(`animationDirectory = houndeye, animationPrefix = he_`). The AnimSet it uses is a *copy of the turkey's*
(`media/AnimSets/turkey/` state folders with `he_` xml entries) - the animal chassis drives a ported
quadruped with vanilla-shaped states.
**C. Generated assets (the CDDA route).** No source assets at all: fal image -> image-to-3D (Meshy) ->
pymeshlab decimate -> weight onto an existing skeleton (the pig's) -> text `.x` writer -> `animationsMesh =
Pig_BoarAndSow`. Every stage is automated and documented in the CDDA note's Assets section; SFX via
ElevenLabs normalized to vanilla loudness.

### Step 3 - animation plumbing
- Zombie chassis: your clips merge into `AnimSets/zombie/<state>/` animNodes gated on
  `z:setVariable("Mod_Is<Monster>", true)` conditions; attack clips selected by a STRING variable
  (`Mod_<M>AttackClip`) and `ZombieBiteDone` timing (see dressed-zombie note for the full node anatomy).
- Animal chassis: copy the host's AnimSet folder (`AnimSets/turkey/` -> your states) OR reuse the host's set
  wholesale via `animationsMesh` on the model script; per-state folders = attack/idle/walk/pathfind/death/
  hitreaction/eat. Extra behaviours go in Lua, not the animset (CDDA gotcha #1: custom animset copies break
  everything - prefer reusing the host set).
- Prefix rules: `anims_X/<animationDirectory>/<prefix>_*.x|glb|fbx` where directory+prefix come from your
  `animationsMesh` block; humanoids scan `Bob`/`Kate` prefixes.

### Step 4 - the brain
- **Zombie chassis:** one Lua controller per monster (`client/COF_<M>.lua` pattern): claim on identity
  dispatch -> set `Is*` variables -> per-tick think (target checks, `ZombieBiteDone` resets, attack-clip
  selection, voice/loop sounds, `COF_MonsterInjury`-style hit bookkeeping, `COF_MonsterTear` armor ripping,
  `COF_MonsterVehicles` car interactions, `COF_RemotePresentation` MP presentation sync).
- **Animal chassis:** per-type `onThink` in a server Lua file: provoke/`goAttack`/pathToLocation for
  movement, `hitConsequences`+`setHitReaction` for aggro (the fightingOpponent trick), Lua-side damage for
  anything animals can't reach (zombies!), colony/state in `ModData`. See the CDDA note - every trap is
  documented.
- **Performance patterns both use:** staggered thinks (15-tick windows / per-second sweeps), per-window
  position snapshots on an 8-tile grid, sound-clip global limiters, weak-keyed identity caches. Never do
  O(n²) work in `OnZombieUpdate`/`OnAnimalUpdate`.

### Step 5 - spawning & sandbox
Population-None-proof spawning: server-side `VirtualZombieManager.instance:createRealZombieNow` +
`dressInPersistentOutfit` (zombies) or `addAnimal`+`addToWorld` (animals); natural spawning via your own
weighted scheduler + metazone habitats (`registerAnimalZone`) for animals. Sandbox options per creature:
enable toggles + rarity enums (`sandbox-options.txt` + `Sandbox.json`).

### Legal hygiene (the part that decides if you can ship)
- **Original assets**: only with a license - CoF's mod exists because the porters had permission. Don't
  extract-and-ship commercial assets without it.
- **Converted assets**: same rule - converting HL's `.mdl` doesn't change its copyright; HLMonsters is
  distributed as the porter's own remake or under the source game's mod policy. If unsure, describe the
  creature and generate your own assets (route C).
- **Your own/generated**: cleanest - document provenance in the README.

## Build steps (a complete port, start to finish)
1. Chassis decision (table above). Write down which free systems you're counting on.
2. Asset route: original (with permission) / converted (smd->fbx->re-skin->`.x` or keep `.glb`) / generated
   (fal pipeline).
3. Skeleton: reuse an existing PZ rig whenever possible - skinning to `Bip01` or the pig skeleton buys you
   the whole vanilla animset for free. A fully custom skeleton means authoring every state.
4. Model+anim script blocks, clothingItem (zombie) or AnimalDefinitions (animal), sounds table, textures.
5. Brain: per-monster controller (zombie) or onThink brain (animal) with the gotchas lists applied.
6. Spawn system + sandbox options + debug right-click spawns for iteration.
7. Iterate with `-debug` + `console.txt`; the animal/zombie gotcha lists are the bug catalog.

## Verification
- All three reference mods run on 42.x (CoF targets 42.20+, HLM 42.0+, ants tested through 42.21.0).
- Mesh/anim check before in-game: play host-skeleton anims on your `.x` offline (CDDA note's mesh oracle).
- In-game: `-debug` animator + spawn commands + `console.txt`.
- Not verified: assimp's per-format quirks on *every* exporter (test your exact `.glb` early - assimp
  version differences bite), custom-skeleton (non-reused) animsets end-to-end.

## Gotchas
1. **Wrong skeleton choice = months of anim work.** Re-skin to `Bip01` (humanoids) or the closest animal rig
   instead of importing the source skeleton; the source skeleton also requires re-authoring every AnimState.
2. **Source anims don't map 1:1 to PZ states.** PZ splits by state folder (attack/idle/walktoward/...); your
   source clip list needs renaming into that shape - batch-rename on export, not by hand.
3. **Zombie controller claims get recycled** - the whole dressed-zombie identity section applies; identity
   comes from outfit, never mod data.
4. **Animals can't be zombie targets without an NPE** - cross-chassis fights must be Lua-driven damage (CDDA
   gotcha #5).
5. **`.glb` vs `.x` animation curves**: assimp loads both but the AnimSet's `m_AnimName` must match the clip
   name inside the file - renamed files without renamed internal clips fail silently.
6. **Texture atlas vs textureChoices**: zombie bodies need `textureChoices` entries in the clothingItem xml
   pointing at your atlas; animal breeds need `texture`/`textureMale` fields.
7. **Sounds too loud/stack**: port the *number* of clips, then cap concurrent playback (CDDA gotcha #14) -
   imported SFX banks are usually louder than PZ mixes.
8. **MP**: zombie chassis monsters need the server-spawn + presentation-sync split (COF_Net pattern); animal
   brains run on `server/` naturally but sounds/loops still need client-side presentation.
9. **The port isn't done when it renders** - hit reactions, death, ragdoll/corpse look, blood, vehicle
   interactions, hitbox extents (`extents`/`physicsChassisShape` equivalents on characters) are each their
   own task; budget for them.
10. **Publishing original game assets will get your mod pulled** - if you can't prove the license, ship the
    code and let users BYO assets, or use generated assets.

## Assets
- `.glb`/`.fbx`/`.x` all load via jassimp64; `.x` (text) is what hand-writers emit (CDDA pipeline).
- Atlases: PNG in `textures/`; clothing `textureChoices` or breed `texture` fields.
- Audio: `.ogg`/`.wav` + `sound` script blocks (category + 3D falloff + variants).

## Open questions
- Whether jassimp64 loads skinned `.fbx` with embedded anims per-clip or one timeline (affects export
  strategy - the proven path is one clip per file).
- Custom skeleton (not reused) minimum-viable AnimSet size - untested; every shipped port reuses a rig.
- `zombie.popman` (PZPopMan) hooks for horde-level monster populations vs self-scheduled spawns.
