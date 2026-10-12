---
kind: game
title: 'Weapons, real ballistics, damage pipeline, effects and vehicle porting: the combat-output layer of Build 42'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); javap-verified'
platform: windows
engine: java
route: loader-api
tools: ["javap (JDK)", "BallisticsController/Bullet (btBullet)", "WorldSoundManager", "ModelManager/ModelInstance", "media/effects param files"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Item_script_parameters"
tags: [b42, weapons, ballistics, damage, effects, tracers, vehicles, feedback, worldsound, attachments, dismember, explosions]
---
# Weapons, real ballistics, damage pipeline, effects and vehicle porting (42.21.0)

> Everything that comes OUT of a fight: the weapon item model, B42's real Bullet-physics ballistics,
> the damage entry points, noise that attracts zombies, visual/audio feedback (tracers, blood, dismember,
> attachment sockets, sounds), and the complete recipe for porting a vehicle.

## Setup
- Builds on `b42-script-system-...md` (script block census) and `b42-lua-callable-java-cheat-sheet.md`
  (verified methods). Surfaces below are `javap`-verified on 42.21.0.

## Route and why
Loader API: weapon/vehicle/entity data in `media/scripts`, feel/feedback in Lua, physics reached through
the engine's own hit paths.

## 1. The weapon is an item script - the fields that matter
`item`/`weapon`-family script fields (see `media/scripts/weapons*.txt` vanilla files to copy):
damage (`MinDamage`/`MaxDamage`), range/`MaxRange`, speed (`BaseSpeed`/`SwingTime`), push (`PushBackMod`/
`KnockBackOnNoDeath`), condition (`ConditionMax`/`ConditionLowerChanceOneIn`), crit, angle, swing/aim anims
(`SwingAnim`, `RunAnim`, `IdleAnim`, `AttachmentType`), sounds (`SwingSound`, `HitSound`,
`ImpactSound`, `DoorHitSound`, `BreakSound`, `BringToBearSound`, `EquipSound`...), categories/tags
(`Tags = Chop;Cut`, `WeaponCategoriesByCharacterType`), ammo/magazine/fire modes for ranged, and the
attachment/visual fields. New damage *types* don't exist - compose from existing hit code + Lua.

## 2. Real ballistics - `zombie.core.physics`
B42 doesn't raycast-and-pray; it has a Bullet-backed ballistic sim:
- `BallisticsController` (javap-verified): `getMuzzlePosition`/`getMuzzleDirection`,
  `calculateMuzzlePosition`, `getTargets`/`getBallisticsTargets`, `getSpreadData`/
  `getBallisticsSpreadData`, `getTargetedBodyPart`, `getCameraTargets`, `hasBallisticsTarget`,
  `getIsoAimingPosition` - live, queryable aim/trace state per shooter.
- `Bullet` (the btBullet wrapper): `addPhysicsObject`, `addRagdoll`, `addVehicle`,
  `applyForce`/`applyImpulse`/`applyCentralForceToVehicle`/`applyTorqueToVehicle`,
  `add6DofConstraint`/`addHingeConstraint`/`addPointConstraint`/`addRopeConstraint`,
  `defineRagdollBodyDynamics`/`defineRagdollAnchors`/`defineRagdollConstraints`,
  `checkWheelCollision`, `CatchToBullet`.
- Mod relevance: custom projectiles are usually scripted as fast hit resolution + visual (see tracers
  below) since raw `Bullet` construction from Lua is unverified; but the *queryable* side
  (`BallisticsController` getters) is how you build custom aim assists, target prediction, and
  "what would this shot hit" tools.

## 3. The damage pipeline - every entry point
javap-verified on `BodyDamage.BodyDamage` + `IsoGameCharacter`/`IsoAnimal`:
- `DamageFromWeapon`, `applyDamageFromWeapon` (weapon-context damage w/ hit metadata),
  `DamageFromAnimal` (claw/bite path), `AddRandomDamageFromZombie`, `damageFromSpikedArmor`,
  `AddDamage`, `AddGeneralHealth`/`ReduceGeneralHealth`, `hitConsequences` (the universal
  "a thing hit me" funnel - drives stagger/death presentation), `setHitReaction`,
  `applyDamageFromVehicleHit`/`onHitByVehicleApplyDamage` (car vs body).
- Explosion/fire: `IsoFireManager.explode(cell, sq, radius_power)` (vanilla's own challenge code uses
  `explode(cell, sq, 100000)`), fire objects on squares, `WasBurntToDeath`.
- Pattern for custom attacks: pick the nearest real path (`DamageFromAnimal` for claws,
  `DamageFromWeapon` for a fake weapon hit, `AddDamage` for anything else) so armor, pain, blood and
  MP sync come for free; roll-your-own health subtraction desyncs and skips all feedback.

## 4. Noise is a first-class API - `WorldSoundManager`
`addSound`, `addSoundRepeating`, `getSoundZomb`/`getSoundAttract`, `getSoundAnimal`/
`getSoundAttractAnimal`, `getBiggestSoundZomb`, `getStressFromSounds`, `getHearingMultiplier`.
- Every real noise source goes through this - your weapons/creatures/events MUST too if you want them
  to attract zombies (or scare animals): `WorldSoundManager.instance:addSound(src, x, y, z, radius,
  volume)` (arg shape per JavaDocs/Umbrella stubs).
- Creatures hear via `respondToSound`/`getLastSoundRespondedTo` (animals) / `RespondToSound` +
  `isRespondingToPlayerSound` (zombies) - a lure/sonar/distraction mod is ~20 lines with these.

## 5. Visual & audio feedback layer
- **Tracers/projectiles**: `media/effects/*.txt` param files drive the tracer/particle effects (see
  `b42-presentation-edge-cases-...md`); custom muzzle flash/tracer = new effect param file + call site.
- **Attachments/sockets**: `ModelInstance.getAttachment(ById)`, `getAttachmentMatrix(ById)`,
  `getAttachmentWorldPosition`, `setItemVisual` - attach item visuals to bone sockets (glowing sword in
  hand, hat on monster). `ModelManager.addStatic`/`addVehicle` place non-character models in the world.
- **Dismember**: `ModelInstance.dismember` - real dismemberment call; drive it from your damage hooks for
  limb-loss monsters.
- **Blood/gore**: `sq:splatBlood`, `haveBlood(Wall|Floor)`, `removeBlood`/`removeCorpse`,
  `VZM.AddBloodToMap`, `onDeath_ShouldDoSplatterAndSounds` - the gore is all callable.
- **Overlay sprites**: `obj:addAttachedAnimSprite(ByName)` - flames, cracks, slime trails on any object.
- **Sounds**: `Sound`/`soundTimeline` script blocks + `getSoundLimiterGroupID`/`registerWithSoundLimiter`
  on items; `playPartSound`/`playPartAnim` (vehicles); `playHurtSound`, `DoFootstepSound`,
  `playerVoiceSound`, `playDeadSound`, `playNextFootstepSound` (all verified earlier).
- **Render surfaces**: `ModelManager.RenderParticles`/`RenderSkyBox`/`RenderWater`/`RenderPuddles` exist -
  skybox and puddles are engine features mods can feed.

## 6. Porting a whole vehicle - the recipe
- `VehicleScript` block (`media/scripts/vehicles/` vanilla files as templates): model, `physicsShape`,
  mass/center, wheels/tires, seats (`Passenger`/`SwitchSeat` entries), doors/parts (part tree +
  install/uninstall mechanics live in `client/Vehicles` Lua), storage containers, skin index,
  sounds (engine loop, horn, siren, door, crash).
- Runtime API (`BaseVehicle`, 587 methods): `createVehicleKey`/`putKeyTo*` family, `getSkinCount`/
  `setSkinIndex`/`updateSkin` (repaints!), `getEngineCondition`/`setEngineSpeed`, `playPartAnim`/
  `playPartSound`, `toggleLockedDoor`, `setGeneralPartCondition`, `isEngineSounding`/`isHornSounding`/
  `isDoorAlarmSounding`.
- Spawning: `addVehicle` script call / vehicle zones (`registerVehiclesZone`, `checkVehiclesZones`) +
  `randomizedVehicleStory` hooks for crash-scene spawns (`getRandomizedVehicleStoryByName`).
- Port a car from another game: mesh via assimp -> `model` block + physicsShape; re-skin with
  `LoadVehicleTexture`-family; part names must match what `client/Vehicles` mechanics expect (copy a
  vanilla part tree and rename visuals, not semantics).

## 7. MP rules for combat
Use the `transmit*`/`sync*` verbs (cheat-sheet #4): `sq:transmitAddObjectToSquare`, `obj:sync*`,
`transmitPlayerVoiceSound`, `transmitModdata`, `transmitWeather`. Raw setters mutate the server world
silently - clients never see them. Server-authoritative damage: apply on server, broadcast presentation
with `sendClientCommand` (the COF_Net pattern).

## Gotchas
1. **No new damage types** - `DamageFromWeapon`/`DamageFromAnimal`/`AddDamage` cover everything; a "sonic
   attack" is `AddDamage` + custom effects + `addSound`.
2. **`hitConsequences` is the right funnel** - it drives knockdown/death/MP; bypassing it = silent deaths.
3. **Aim state is queryable** (`BallisticsController` getters) - build aim assists on it instead of
   reimplementing cones.
4. **`addSound` radius is tiles and volume scales attraction separately** - test attraction radius in-game;
   the zombie hearing multiplier differs per sandbox.
5. **`explode` power is not radius** - vanilla uses 100000 for a room; small values do nothing visible.
   Test on a scratch save.
6. **Vehicle parts are semantic** - rename the mesh freely, but `Engine`/`GasTank`/`Door*` IDs drive
   mechanics code; inventing part names breaks install/uninstall.
7. **`dismember` needs a rigged model instance** - it won't do anything on sprites/thumpables; for those,
   fake gore with blood splats + corpse objects.
8. **Effects param files are per-frame evaluated** - spamming tracers via Lua every tick tanks FPS; attach
   them to the one-shot event, not the think loop.

## Verification
- javap-verified 42.21.0: `BallisticsController`, `Bullet`, `WorldSoundManager`, `ModelInstance`,
  `ModelManager`, `IsoThumpable`, `BaseVehicle`, `BodyDamage` surfaces cited above.
- Verified in the wild: `IsoFireManager.explode` (vanilla challenge), `WorldSoundManager` attraction
  (vanilla noise system), skin/part vehicle APIs (vanilla mechanics UI), tracer/effects param files
  (vanilla `media/effects/`).
- Not verified: raw `Bullet`/`BallisticsController` construction from Lua, `addStatic` model placement
  call path, `dismember` invocation from mod code.

## Open questions
- Exact Kahlua exposure of `zombie.core.physics` - which classes got Lua bindings vs which are
  Java-internal? (next javap/decompile target).
- `effects/` param grammar in full (the files are self-describing but the evaluator's feature set is
  undocumented).
- Whether `ModelManager.RenderParticles` is mod-callable or engine-internal.
