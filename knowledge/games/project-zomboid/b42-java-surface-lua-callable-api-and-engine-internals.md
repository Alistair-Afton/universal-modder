---
kind: game
title: 'The Lua-callable Java surface: a cheat sheet of the methods that matter on 15 key classes (42.21.0 javap dump)'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); method names dumped from projectzomboid.jar with javap'
platform: windows
engine: java
route: loader-api
tools: ["javap (JDK)", "PZ-Umbrella stubs for signatures", "ZomboidDecompiler for internals"]
anti_cheat: 'none'
status: in-progress
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://www.projectzomboid.com/modding/"
- "https://github.com/PZ-Umbrella/Umbrella"
tags: [b42, api, cheat-sheet, javap, isozombie, isoplayer, isoanimal, isogridsquare, isocell, isoworld, inventoryitem, itemcontainer, basevehicle, virtualzombiemanager, scriptmanager, bodydamage, stats, isoobject]
---
# The Lua-callable Java surface: a cheat sheet of the methods that matter (42.21.0)

> javap-dumped surface of the classes modders actually touch, curated into clusters so "does the game expose
> X?" is a Ctrl-F away. Counts are the total public methods on 42.21.0 - each cluster lists the useful ~10%.
> Regenerate with `javap -cp projectzomboid.jar <class>`; signatures + docs are in Umbrella/JavaDocs - this
> note is the map of what exists, not every arg list. Lua calls these as `obj:method(args)`.

## Setup
- `javap -cp projectzomboid.jar zombie.characters.IsoZombie` etc. (JDK 25 used). `javap -p` adds privates for
  reverse-engineering; Lua only reaches `public` methods.

## Route and why
Loader API, same as always - this is the index OF that API.

## The cheat sheet

### `zombie.characters.IsoZombie` (211 methods) - the monster chassis
- Identity/dress: `dressInPersistentOutfitID`, `dressInNamedOutfit`, `dressInRandomOutfit`,
  `dressInClothingItem`, `getOutfitName`, `isPersistentOutfitInit`, `getPersistentOutfitID`,
  `getHumanVisual`, `InitSpritePartsZombie`, `getObjectName`, `getOnlineID`, `isRemoteZombie`,
  `getOwnerPlayer`.
- AI/senses: `spotted(player)` / `spottedNew` / `spottedOld`, `isTargetInCone`, `pathToCharacter`,
  `pathToLocationF`, `RespondToSound`, `isRespondingToPlayerSound`, `isMovingToPlayerSound`,
  `setTurnAlertedValues`, `addAggro`, `isLeadAggro`, `clearAggroList`, `setWalkType`,
  `getZombieWalkTowardSpeed`, `setSpeedTypeFromWalkType`.
- State tricks: `setCrawler`/`isBecomeCrawler`/`setCanCrawlUnderVehicle`/`shouldGetUpFromCrawl`,
  `setFakeDead`/`isFakeDead`/`setForceFakeDead`/`setWasFakeDead`, `setKnockedDown`, `knockDown`,
  `setOnFloor`, `setCanWalk`, `setGodMod`, `setSitAgainstWall`, `setReanimatedForGrappleOnly`,
  `actionStateChanged`, `initializeStates`, `GetAnimSetName`.
- Combat/voice: `Hit`, `getAttackDidDamage`, `setAttackOutcome`, `onZombieGrappleEnded`,
  `playHurtSound`, `DoFootstepSound`, `getVoiceChoice`, `getVoiceSoundName`, `getBiteSoundName`,
  `updateVocalProperties`, `setVehicleHitLocation`.
- Lifecycle: `load`/`save`, `removeFromWorld`, `onDeath_ShouldDoSplatterAndSounds`, `getOwner`,
  `setHealth`, `isDead`.

### `zombie.characters.IsoPlayer` (IsoZombie's sibling, ~500)
- Identity/visual: `getDescriptor`, `getHumanVisual`, `resetModel`, `getCharacterTraits`,
  `setPerkLevelDebug`, `getXp`/`setXp`, `getPerkLevel`, `getZombieKills`.
- Voice: `playerVoiceSound(name)` (plays VoiceMale/Female*), `transmitPlayerVoiceSound`,
  `stopPlayerVoiceSound`, `getVoiceType`/`setVoicePitch`, `playGainExperienceLevelSound`,
  `playBloodSplatterSound`.
- Body/stats: `getBodyDamage`, `getStats`, `getNutrition`, `getFitness`, `getMoodleLevel`,
  `setMoodleCantSprint`, `hitConsequences`, `applyDamageFromVehicleHit`, `onHitByVehicleApplyDamage`,
  `startReceivingBodyDamageUpdates`, `isAsleep`/`getAsleepTime`, `allPlayersAsleep` (MP sleep sync),
  `isOnlyPlayerAsleep`, `getSleepingPillsTaken`.
- World interaction: `getCurrentSquare`, `isWearingNightVisionGoggles`/`setWearingNightVisionGoggles`,
  `addAttachedAnimal`/`getAttachedAnimals`/`checkAnimalAttachedToRope` (leash/lead system),
  `isRandXp`, `getOnlineID`.
- Bumps: `setBumpType`, `setBumpFall`, `setBumpFallType`, `setVariable("BumpDone")`,
  `clearVariable`, `reportEvent("wasBumped")` (the trip/knockdown sequence, see CDDA note).

### `zombie.characters.IsoGameCharacter` (base of both, ~450)
`getBodyDamage`, `getXp`, `getPerkList`, `setHealth` (does NOT clamp for animals), `isAsleep`/`setAsleep`,
`setIgnoreStaggerBack`, `getTimeThumping`, `setVariable`/`getVariable`/`clearVariable` (animation
variables!), `getAdvancedAnimator`, `reportEvent`, `getModData`, `setX/setY/setZ` + `setLastX/Y`,
`getCurrentSquare`, `getInventory`, `getPrimaryHandItem`/`setPrimaryHandItem`, `getSecondaryHandItem`,
`Dressup`/`getWornItems`, `setInvisible`, `setGodMod`, `pathToLocation`, `pathToCharacter`,
`getSpeedMod`/`setVariable("animalSpeed")`, `clearFallDamage`, `isOnFloor`, `setDirectionAngle`,
`MoveForward`/`faceThisObject`, `getZombieKills`, `setZombieKills`.

### `zombie.characters.animals.IsoAnimal` (the creature chassis, ~250)
`getBehavior` (`BaseAnimalBehavior`: `goAttack`, `forceFleeFromChr`, `isDoingBehavior`), `pathToCharacter`,
`pathToLocation`, `pathToTrough`, `fleeTo`, `respondToSound`, `getLastSoundRespondedTo`,
`changeStress`/`getStress`/`setDebugStress`/`updateStress`/`getStressTxt`, `isAlerted`/`setIsAlerted`,
`setAnimalAttackingOnClient`, `isAnimalAttacking`, `hitConsequences`, `setHitReaction`,
`getFullGenome`/`getFullGenomeList`/`copyGenome` (`AnimalGene`s - moddable genetics), `getBreed`
(`AnimalBreed`), `getHunger`/`getThirst`, `canBeMilked`/`milkAnimal`/`haveEnoughMilkToFeedFrom`/
`getMilkType`, `addEgg`/`createEgg`/`getEggsPerDay`/`getEggGeneMod`, `playDeadSound`,
`playNextFootstepSound`, `getAnimalSoundState`, `getHealthText`, `save`/`load`/`unloaded` (they persist!),
`pathFailed`/`shouldFollowWall` (free "unreachable" signal).

### `zombie.characters.BodyDamage.BodyDamage` & `Stats`
`getBodyPartHealth`, `AddDamage`, `AddGeneralHealth`, `ReduceGeneralHealth`, `DamageFromWeapon`,
`applyDamageFromWeapon`, `DamageFromAnimal`, `AddRandomDamageFromZombie`, `damageFromSpikedArmor`,
`IsBandaged`/`IsBleeding`/`IsBleedingStemmed`/`IsDeepWounded`, `isBodyPartBleeding`/`getNumPartsBleeding`/
`isNeckBleeding`, `IsFakeInfected`, `getApparentInfectionLevel`, `WasBurntToDeath`, `ReducePanic`,
`JustAteFood`, `JustTookPainMeds`, plus per-part `BodyPart` objects (`BodyPartType` enum - fixed list).
`Stats`: `getNumVisibleZombies`/`getNumChasingZombies`/`getNumVeryCloseZombies`/`getVisibleZombies`,
`getLastEndurance`/`getEnduranceWarning`, `getNicotineStress`, `setTripping`/`getTrippingRotAngle`/
`addTrippingRotAngle`, `get`/`set`/`add` per-stat.

### `zombie.iso.IsoGridSquare` (the tile, ~250)
`getObjects`/`getObjectWithSprite`/`AddTileObject`/`RemoveTileObject(+ErosionNoRecalc)`,
`transmitAddObjectToSquare`/`transmitRemoveItemFromSquare(+OnClients)`/`transmitModdata` (MP sync verbs),
`getRoom` (IsoRoom -> RoomDef name for loot/stories), `getFire`, `getWater`/`hasWater`/`getWaterObject`/
`startWaterSplash`/`clearWater`, `getGraffitiObject`/`haveGraffiti`/`removeGraffiti`,
`splatBlood`/`haveBlood(Wall|Floor)`/`removeBlood`/`removeCorpse`, `removeGrime`,
`GetR/G/BLightLevel` + `SetR/G/BLightLevel` (writeable lighting!), `interpolateLight`,
`DoWallLightingNW/N/W`, `switchLight`, `removeGlassAttachments`, `removeUnderground`, `has(flag)`/
`has(flag)`/`Is(flag)`-family (the `Is` name was removed in 42.21 - the rename trap), `isInARoom`/`getBuilding`, `isFree`, `hasTypes`.

### `zombie.iso.IsoCell` (loaded world, ~200)
`getGridSquare(x,y,z)` (nil outside loaded area - THE fundamental constraint), `getZombieList`,
`getObjectList`/`getObjectListForLua`, `getRoomList`/`getRoom`, `addMovingObject`,
`getProcessIsoObjects`/`addToProcessIsoObject(Remove)` (register objects for per-tick updates),
`addToProcessItems`/`getProcessWorldItems`/`addToProcessWorldItems`, `addToWindowList`/`removeFromWindowList`,
`addLamppost` (IsoLightSource - custom lights!), `GetEffectivePlayerRoomId`,
`SetCutawayRoomsForPlayer`, `isSafeToAdd`, `getRemoveList`/`getAddList`.

### `zombie.iso.IsoWorld` (the world, ~150)
`getCell`, `getMetaGrid` (IsoMetaGrid - unloaded-safe zones), `getMetaChunk(FromTile)`,
`registerZone`/`registerZoneNoOverlap`/`registerNavZones`/`registerVehiclesZone`/`registerMannequinZone`/
`registerRoomTone`/`registerSpawnOrigin`/`registerWaterZone`/`registerWaterFlow`/
  (note: `registerAnimalZone` lives on `IsoMetaGrid`, not `IsoWorld`)/
`removeZonesForLotDirectory`/`checkVehiclesZones`, `getWeather`/`setWeather`/`transmitWeather`,
`getZombiesDisabled`, `getRandomizedZoneList`/`getRandomizedZoneStoryByName`,
`getRandomizedBuildingList`, `getRandomizedVehicleStoryList`/`getRandomizedVehicleStoryByName`,
`getRandomizedWorldBase` (world-story registries!), `getLuaSpawnCellX/Y`, `getSpawnRegion`,
`setSpawnRegion`, `getWorldAgeDays`/`getTimeOfDay`-style time accessors (see `getGameTime()` too).

### `zombie.VirtualZombieManager` (population director)
`createRealZombieNow(x,y,z)`/`createRealZombie`/`createRealZombieAlways` (variants),
`createCorpseZombie`, `createEatingZombies`, `createHordeFromTo`, `addZombiesToMap`,
`tryAddIndoorZombies`/`addIndoorZombiesToChunk`, `addDeadZombiesToMap`, `RemoveZombie`,
`removeZombieFromWorld`, `addToReusable`/`isReused`/`reusableZombiesSize` (the recycling pool from the
CoF bugs), `canSpawnAt`/`shouldSpawnZombiesOnLevel`, `checkZombieKeyForBuilding`/`spawnBuildingKeyOnZombie`
/`checkAndSpawnZombieForBuildingKey` (key-drops on zombies!), `roomSpotted`, `AddBloodToMap`,
`getKeySpawnChanceD100`, `instance`.

### `zombie.inventory` pair
`ItemContainer`: `AddItem`/`AddItemBlind`/`DoAddItem`/`addItems`/`SpawnItem`, `contains`/`containsWithModule`,
`getItemFromTypeRecurse`, `getCapacity`/`setCapacity`/`getEffectiveCapacity`/`getFreeCapacity`,
`isExplored`/`setExplored` (loot "has been opened" flag - set it false to re-lootable a container),
`getNumberOfItem`, `getContainingItem`.
`InventoryItem`: `getModData` (per-item KahluaTable - store anything), `getByteData`/
`loadCorpseFromByteData`/`createAndStoreDefaultDeadBody` (dead-body-as-item!), `getType`/`getCategory`/
`getTex`, `getWorldItem`/`setWorldItem`/`getEquipParent`/`setEquipParent`,
`getEquipSound`/`getUnequipSound`/`getDropSound`/`getBringToBearSound`/`getAimReleaseSound`,
`getBookSubjects`/`getMagazineSubjects`/`getCoverType`, `UseAndSync`,
`getSoundLimiterGroupID`/`registerWithSoundLimiter`, `setJobType`/`getJobDelta`, `SetContainerPosition`,
`getA/R/G/B` (tint!), `getCleanString`.

### `zombie.vehicles.BaseVehicle` (587 methods - the deep end)
`createVehicleKey`/`putKeyToZombie`/`putKeyToContainer`/`putKeyToWorld`/`addKeyToGloveBox`/
`addBuildingKeyToGloveBox`, `getEngineCondition`/`getEngineState`/`setEngineSpeed`/`addEngineSpeed`,
`getSkinCount`/`getSkinIndex`/`setSkinIndex`/`updateSkin`/`getSkin`/`LoadVehicleTexture(s)` (repaints!),
`getPart`/`playPartAnim`/`playPartSound`, `toggleLockedDoor`/`can(Lock|Unlock|Open)Door`,
`haveOneDoorUnlocked`, `isEngineSounding`/`isBrakePedalPressed`/`isGasPedalPressed`/`isDoorAlarmSounding`/
`isHornSounding`, `getPassengerDoor(2)`/`getSeat`/`getPassengerSwitchSeat`/`getPassengerSwitchSeatCount`,
`setGeneralPartCondition`, `getSqlId`, `getFakeSpeedModifier`. Part-level mechanics are the
`VehiclePart` class + `client/Vehicles` Lua (install/uninstall actions).

### `zombie.scripting.ScriptManager` (122 - the data registry)
`getItem`/`getAllItems`, `getRecipe`/`getAllRecipes`, `getUniqueRecipe`/`getAllUniqueRecipes`,
`getEvolvedRecipe`/`getAllEvolvedRecipes(List)`, `getFixing`/`getAllFixing`, `FindItem`,
`getVehicle`/`getVehicleTemplate`/`getAllVehicleTemplates`, `getModelScript`/`getAllModelScripts`/
`addModelScript` (register models at runtime!), `getAnimationsMesh`/`getAllAnimationsMeshes`,
`getGameSound`/`getAllGameSounds`, `getSoundTimeline`/`getAllSoundTimelines`,
`getPhysicsShape`/`getAllPhysicsShapes`, `getGameEntityTemplate`/`getAllGameEntityTemplates`
(the entity blocks), `getMannequinScript`/`getAllMannequinScripts`, `getSpriteModel`/`addSpriteModel`,
`getClockScript`/`getAllClockScripts`, `getScriptsForType`, `instance`.

### `zombie.iso.IsoObject` (every world thing)
`getSprite`/`setSprite`/`getTextureName`, `getContainer`/`getContainers`/`getContainerClickedOn`,
`getModData`/`setModData`/`hasModData`, `getObjectName`, `Damage`/`getDamage`/`setDamage`,
`getAttachedAnimSprite(Count)`/`addAttachedAnimSprite(ByName|Instance)`/`clearAttachedAnimSprite`/
`isAttachedAnimSprite`/`isAttachedOrOverlaySprite` (overlay sprites on objects - grime/fire/damage
overlays!), `getChildSprites`/`setChildSprites`, `sync`/`syncIsoObject(+Send/Receive)`/
`syncFluidContainer(Receive|Send)` (MP object sync), `getForwardMovementIsoDirection`.

## Verification
- Names come from `javap -cp projectzomboid.jar <class>` on 42.21.0 cross-checked against the
  community javadocs; a handful of dump names turned out non-public or Lua-side (see inline notes).
  When in doubt, re-run javap on your build - names drift between 42.x releases.
- Spot-checked in-game in the referenced ports (createRealZombieNow, dressInPersistentOutfitID,
  addToReusable/isReused, registerAnimalZone, pathToCharacter/fleeTo/goAttack, hitConsequences,
  setVariable/reportEvent, getModData, isExplored) via COF/HLM/CDDA.
- Not verified: every overload's exact arg lists (javap -public strips parameter names - use Umbrella
  stubs), the unlisted ~4000 methods, `zombie.popman` reachability, ECS component API.

## Regenerating / going deeper
```bash
javap -cp projectzomboid.jar zombie.<pkg>.<Class>          # public surface
javap -cp projectzomboid.jar -p zombie.<pkg>.<Class>      # + privates (RE only)
javap -cp projectzomboid.jar -c zombie.<pkg>.<Class>      # bytecode
```
Umbrella stubs give the same surface with Lua-typed signatures and arg names (Rosetta). For behaviour,
decompile with ZomboidDecompiler - describe findings in your own words, never ship decompiled code.

## Gotchas
1. **Method vs field** - repeat of the big one: Lua can't read `obj.field`; find the `getField()` accessor.
2. **Signatures lie about overloads** - several methods appear twice (e.g. `AddItem(boolean)` vs
   `AddItem(item)`); Kahlua picks by arg count/types - a wrong type picks the other overload silently.
3. **Statics exist** - `IsoPlayer.allPlayersAsleep`, `VirtualZombieManager.instance`,
   `ScriptManager.instance`, `BaseVehicle.LoadAllVehicleTextures` - call as `Class.method()` not
   `obj:method()`.
4. **The `transmit*` family is the MP-safe mutation path** - `sq:transmitModdata`,
   `sq:transmitRemoveItemFromSquare`, `obj:sync*` - use them instead of raw setters on servers or your
   change never reaches clients.
5. **`getGridSquare` nil = unloaded** - combine with `getMetaGrid` zones for world-level work.
6. **`setExplored(false)` on a container re-lootables it** - and similar "state flag" methods hide on most
   classes; grep the dump before assuming something's impossible.
7. **Package-class name traps** - `zombie.characters.BodyDamage.BodyDamage` is a top-level class inside
   a package also named `BodyDamage`, not an inner class; `playerObj:getBodyDamage()` returns it.
   True inner classes (`IsoZombie$Aggro`, `BaseVehicle$engineStateTypes`) likewise come back through
   accessor methods.
8. **Method names drift between 42.x builds** - this sheet is 42.21.0-exact; re-run javap on upgrade.

## Open questions
- Which of the remaining ~4000 unlisted methods are the hidden gems (`javap` the whole jar and rank by
  modding relevance - a possible generated appendix).
- `zombie.popman` (PZPopMan) Lua reachability - the population director may be Java-internal.
- ECS (`registerECSComponents` on IsoZombie) - the entity-component surface is new in B42 and barely
  documented anywhere.
