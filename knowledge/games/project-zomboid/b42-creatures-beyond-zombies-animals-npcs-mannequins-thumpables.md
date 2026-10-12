---
kind: game
title: 'Creatures beyond zombies and animals: friendly NPCs, mannequins, thumpable pseudo-creatures, multi-part bosses and hybrids'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10)'
platform: windows
engine: java
route: loader-api
tools: ["javap (JDK)", "SurvivorSwap API", "MannequinScript blocks", "IsoThumpable", "Bullet physics (btBullet)"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Lua_(API)"
tags: [b42, creatures, npc, follower, companion, mannequin, thumpable, boss, hybrid, ragdoll, survivor, pseudo-creature]
---
# Creatures beyond zombies and animals: the third, fourth and fifth chassis

> The porting playbook assumes a creature maps to the zombie or animal chassis. Plenty of things don't:
> friendly NPC survivors, turrets, mimics, talking heads, multi-part bosses, ghosts, drones. This note
> catalogues every other "body" the engine gives you on 42.21.0, what each is actually capable of, and the
> Lua patterns that turn them into creatures.

## Setup
- Extends `b42-porting-creatures-from-other-games.md` (chassis 1+2), `b42-lua-callable-java-cheat-sheet.md`
  (verified methods). Method names below are javap-verified unless marked otherwise.

## Route and why
Same loader-API route - every chassis here is a stock engine object driven from Lua.

## The extended chassis table
| "Body" | Engine object | What it gives free | Use for |
|---|---|---|---|
| Zombie, AI inverted | `IsoZombie` + controller | full pathing, animset, combat states, vehicle collisions | **friendly/companion NPCs**, neutral monsters, wildlife |
| Simulated survivor | `IsoPlayer`-shaped (non-network player) or zombie-in-clothes + `SurvivorSwap` | full clothing/hair/voice/profession system | named NPCs with real identity |
| Animal, domesticated | `IsoAnimal` (`attachAnimal`, leash, troughs) | persistence, follow/lead, milk/eggs/stress | pets, mounts-adjacent, pack mules |
| Mannequin | `IsoMannequin` + `mannequinScript`/`registerMannequinZone` | posed dressed human body, saved pose | static NPCs, corpses-that-talk, ghosts |
| Thumpable | `IsoThumpable` | placed object with `Thump`/`WeaponHit`/`animalHit`/`changeSprite`, health, break sound | **turrets, mimics, plants, destructible creatures** |
| Physics body | `Bullet` (`addPhysicsObject`, `addRagdoll`, constraints) | real Bullet sim: forces, impulses, 6DoF/hinge/point/rope joints | ragdoll monsters, thrown creatures, physics beasts |
| Bare sprite/model | `IsoObject` + attached sprites / `ModelManager.addStatic` | pure visuals on a square | illusions, holograms, ambient wildlife |
| Multi-part composite | N synced objects of any above | whatever each part gives | **bosses** (limbs that die separately), swarms |

## Chassis 3: the friendly/neutral humanoid (zombie with inverted AI)
An `IsoZombie` is not hardcoded hostile - hostility is its target-selection state. From Lua:
- `z:clearAggroList()` + never call `spotted(player)` -> it never acquires players; it still paths, idles,
  plays its animset and can be attacked.
- Keep it from wandering: reissue `pathToLocationF`/`pathToCharacter` to a follow target every think tick
  (the animal `pathToCharacter` equivalent exists on IsoZombie too).
- Suppress bites: the attack state only triggers via `getShouldAttack` proximity+target - with no target it
  never swings; belt-and-braces is clearing `attackOutcome`/interrupting `ZombieBite` variables in your
  controller.
- Voice: `playerVoiceSound` is IsoPlayer-only, but zombies have `getVoiceSoundName`/`updateVocalProperties`;
  a controller can play `Sound` script entries at the zombie's position for custom speech/barks.
- Dressing: `dressInNamedOutfit` + normal clothingItems = a real-looking survivor, no monster item needed.
- Interaction: `OnFillWorldObjectContextMenu` on the zombie square -> trade/talk/quest menus; hand them
  items via the menu -> `player:getInventory()` side-effects.
- Persistence: zombies save/load; keep identity via persistent outfit ID, never ModData (recycling gotcha).
- Cost: it's still a zombie internally - it eats performance like a zombie, gets targeted by zombie-hit
  code, and the *other* zombies will treat it as one of theirs. Player melee will auto-target it.

## Chassis 4: mannequins - the posable human body
- `mannequinScript` script blocks + `ScriptManager:getAllMannequinScripts` - dress and pose are script data.
- `IsoWorld:registerMannequinZone` places them via metazones (unloaded-world safe) - that's how vanilla
  seeds mannequins into stores.
- Capabilities: a dressed posed body with a real inventory/outfit, per-mannequin zone, save/load.
- Use as creatures: "static NPC" (shopkeeper that never moves - dialogue via context menu), scripted scare
  (swap `mannequinScript`/move the square's object), corpse displays. Animating one = re-pose swaps, not
  skeletal animation - accept the limitation.

## Chassis 5: IsoThumpable - the pseudo-creature object
javap-verified surface: `Thump`, `WeaponHit`, `animalHit`, `Damage`, `changeSprite`, `addSheet`,
`canClimbOver`/`canClimbThrough`, `TestVision`/`TestPathfindCollide`/`TestCollide`, `ToggleDoor`,
`IsStrengthenedByPushedItems`, `GetBreakFurnitureSound`, padlocks, curtains.
- It is a *placed, damageable, pathfind-aware, vision-blocking* object - everything a stationary creature
  needs. Zombies path around it, weapons hit it, animals hit it.
- Brains: register it in `IsoCell:addToProcessIsoObject` (per-tick update list) or drive from
  `OnObjectAdded`/tick events + ModData state.
- Presentation: `changeSprite` + `addAttachedAnimSprite` = cheap state animation (a mimic that opens its
  mouth is two sprites); pair with a `WorldSoundManager:addSound` heartbeat and nobody checks for legs.
- Proven family: traps/turrets/turrets-that-hate-zombies (`WeaponHit`+`animalHit` give you the hit entry
  points), destructible bosses that are really furniture.

## Chassis 6: physics bodies and ragdolls (`zombie.core.physics.Bullet`)
B42 ships a real Bullet wrapper: `addPhysicsObject`, `addRagdoll`, `defineRagdollBodyDynamics`/
`defineRagdollAnchors`/`defineRagdollConstraints`, `add6DofConstraint`/`addHingeConstraint`/
`addPointConstraint`/`addRopeConstraint`, `applyForce`/`applyImpulse`/`applyCentralForceToVehicle`/
`applyTorqueToVehicle`, `addVehicle`, `checkWheelCollision`, `CatchToBullet`.
- Creature uses: ragdoll corpses that keep interacting, throwable creatures, physics-driven monsters
  (a wrecking-ball boss chained by rope constraints), monster-vs-car physics.
- Caveat: this surface is Java-internal physics; reach it via the character/vehicle hit paths that already
  feed it (vehicle ramming, explosions) rather than constructing Bullet objects from Lua. Reachability of
  raw `Bullet` from Kahlua is unverified - plan around the mediated paths.

## Hybrids and multi-part creatures
- **Boss with destructible limbs**: N IsoZombies/IsoAnimals spawned at offsets, one controller owns all of
  them, `attachAnimSprite` overlays weld the visuals, shared ModData pool = shared HP. Death of a limb =
  remove that object + `dismember` presentation on the main body (`ModelInstance.dismember` exists).
- **Swarm**: K IsoAnimals of a tiny breed (the CDDA ants precedent - colony behavior, pack `goAttack`).
- **Rider+mount**: animal chassis body + attached dressed zombie or itemVisual on the same square; move in
  lockstep from one think function. No native mounting - don't promise it.
- **Ghost/hologram**: invisible-ish presentation (alpha/`setInvisible` on a character leaves the AI but
  drops visuals) + attached sprite + `WorldSoundManager` whispers. Cheaper than any creature.

## Gotchas
1. **A friendly zombie still smells like a zombie** - melee auto-aim, other zombies' crowd behavior, the
   fear system (players get panic near it). Budget a panic override or use mannequin/animal instead.
2. **Mannequins can't skeletal-animate** - pose swaps only. A "mannequin that walks" is a dressed zombie.
3. **IsoThumpable has no locomotion** - moving it = remove + respawn at the next square (fine for
   slug/burrower monsters, broken for chasers).
4. **Multi-part sync dies on unload** - reassemble the composite on chunk load (`OnObjectAdded`+ModData
   membership tags); parts despawn independently.
5. **`attachAnimal`/leash needs the animal to actually be attachable** (breed flag) - check
   `checkAnimalAttachedToRope` semantics before designing around leads.
6. **`playerVoiceSound` is IsoPlayer-only** - NPC speech = positioned `Sound` script entries, or a fake
   voice via your own audio; there is no third-party speech API.
7. **Bullet physics from raw Lua is unproven** - treat `zombie.core.physics` as engine-internal; design
   physics-creature behavior around vehicle hits/explosions (the mediated paths) until verified.
8. **All non-zombie bodies dodge zombie-AI cost but lose the free hunter brain** - you write targeting,
   pathing cadence and attack gating yourself (see CDDA gotchas).

## Verification
- Verified on 42.21.0: javap surfaces (`IsoThumpable`, `Bullet`, `BallisticsController`,
  `ModelInstance`/`ModelManager`, `WorldSoundManager`), friendly-zombie pattern derivable from the
  target-selection API, mannequin zone registration exists in `IsoWorld`.
- Not verified in-game: raw `Bullet` construction from Lua, `ModelManager.addStatic` Lua reachability,
  dressed-zombie-as-companion end-to-end (the inverse-AI trick is new ground), multi-part boss sync.

## Assets
Same pipeline as the porting note - these chassis all take the same models/anims/sounds; only the host
object changes.

## Open questions
- Can Lua call `Bullet`/`BallisticsController` directly, or only through hit paths? (decompile
  `zombie.core.physics` Lua bindings - Kahlua only reaches registered classes).
- `IsoMannequin`'s full method surface (not yet dumped - `javap` it when needed).
- Whether a non-network `IsoPlayer` can be spawned purely as a puppet (likely Java-internal; the
  zombie-in-clothes route covers the use case).
