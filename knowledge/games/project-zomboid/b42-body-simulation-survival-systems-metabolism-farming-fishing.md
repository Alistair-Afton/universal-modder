---
kind: game
title: 'The body simulation and survival gameplay systems: Nutrition, Metabolics, Thermoregulator, Fitness, farming, fishing, foraging, traps, power and water'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); javap-verified class surfaces'
platform: windows
engine: java
route: loader-api
tools: ["javap (JDK)", "media/lua/client/Farming|Fishing", "SGlobalObjectSystem (rain barrels, generators)"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Lua_(API)"
tags: [b42, survival, metabolism, nutrition, thermoregulator, fitness, farming, fishing, foraging, traps, generator, rain-barrel, needs, moodles, illness]
---
# The body simulation and survival gameplay systems (42.21.0)

> "Whole systems and gameplay mechanics" territory: PZ models a body with real metabolic state
> (calories/macros/weight, MET→watts, core temperature, exercise regularity/stiffness) and a set of
> server-authoritative survival loops (farming, fishing, foraging, traps, power, water). Every one is
> reachable from Lua - this note is the map + extension recipes.

## Setup
- javap-verified surfaces on 42.21.0; Lua systems live under `media/lua/client|server|shared` (paths
  below are from the real install). For character stat fields see the cheat-sheet note (`Stats`).

## Route and why
Loader API - extend via Lua systems, script data and the verified getters/setters.

## 1. The body sim - `zombie.characters.BodyDamage.*`

**Nutrition** (`playerObj:getNutrition()`): `getCalories`/`setCalories`, `getCarbohydrates`/`setCarbohydrates`,
`getLipids`/`setLipids`, `getProteins`/`setProteins`, `getWeight`, `setIncWeight`/`setDecWeight`/
`setIncWeightLot` (weight trend flags), `applyTraitFromWeight`/`applyWeightFromTraits`,
`characterHaveWeightTrouble`, `canAddFitnessXp`. Food items feed these via `hungerChange`/`carbohydrates`/
`lipids`/`proteins`/`calories` item fields - mod food is script data; mod *metabolisms* are these setters.

**Metabolics**: an enum of activity intensities with real physics conversions - `MetToW`, `MetToBtuHr`,
`MetToWm2`, `getMet`/`getW`/`getBtuHr`/`getWm2`. The engine converts your current activity into watts of
body heat; your systems can read the same numbers instead of inventing a heat model.

**Thermoregulator**: `getCoreTemperature`/`getCoreCelcius`/`getCoreRateOfChange`, `getBodyHeatDelta`/
`getCoreHeatDelta`, `getBodyHeatMultiplier`/`getCoreHeatContractMultiplier`/`getCoreHeatExpandMultiplier`,
`getExternalAirTemperature`, `getBodyFluids`, `getEnergy`/`getEnergyMultiplier`, `getCombatModifier`,
`getCatchAColdDelta`, `getDefaultMultiplier`, `getDbg_*` debug reads. Clothing, wetness, wind and
exercise all feed `getCoreHeatDelta` - mod clothing warmth already works; mod *temperature sources*
(suits, mutations, zones) apply heat via the multiplier setters or by driving the ambient side
(`getExternalAirTemperature` is read-mostly; set the square/room side instead).

**Fitness**: `exerciseRepeat`, `setCurrentExercise`/`getCurrentExe`, `incRegularity`/`getRegularity`/
`getRegularityMap`/`setRegularityMap`/`initRegularityMapProfession`, `incFutureStiffness`/
`onGoingStiffness`/`getCurrentExeStiffnessInc`/`removeStiffnessValue`, `reduceEndurance`, `incStats`,
`canAddFitnessXp` (on Nutrition). The exercise minigame is data + these calls - custom exercises,
workout machines (global object + `setCurrentExercise`) and stiffness-based mechanics are all open.

## 2. Needs, moodles, illness
- Needs live in `Stats` (`get`/`set`/`add` per stat): hunger, thirst, fatigue, endurance, stress,
  boredom, unhappiness, drunkenness, panic, sanity-adjacent. New *custom* needs = a float on ModData +
  a fake moodle (UI) + the same decay math vanilla uses - `MoodleType` is a fixed enum so you cannot
  add real moodle icons (workaround documented in the character-hooks note).
- Illness/infection: `BodyDamage` fake/real infection (`IsFakeInfected`, `getApparentInfectionLevel`),
  food sickness via `JustAteFood`, poison via food item fields, colds via `getCatchAColdDelta`,
  `ReducePanic`, `JustTookPainMeds`. Custom diseases = stat drift + symptoms (pain/sickness writes) +
  timed cure item - all Lua.
- Sleep/fatigue: `isAsleep`/`setAsleep`, `allPlayersAsleep` (MP vote), `getAsleepTime`,
  `getSleepingPillsTaken`, fatigue stat on `Stats`.

## 3. Survival loops - each is a Lua system you can clone
| System | Where it lives | Extension point |
|---|---|---|
| Farming | `client/Farming/CFarmingSystem.lua` (an `SGlobalObjectSystem` child), `CFarming_Interact.lua`, `ISFarmingMenu`, TimedActions | new crops = plant props + farming entries; new farms = copy the system pattern |
| Fishing | `client/Fishing/` (`FishingManager`, `FishingHandler`, `FishingStates`, `FishingDebugWindow`) | states machine is the extension seam; water zones (`registerWaterZone`/`registerWaterFlow`) define fishable water |
| Foraging | `forageClient.lua` + `ISForageIcon.lua`, `forageData` tables | add loot categories/zones in the forage data; the icon system is client-side |
| Rain barrels | `client/RainBarrel/CRainBarrelGlobalObject.lua` + `CRainBarrelSystem` | the canonical "persistent interactive object" template (see interactive-world note) |
| Generators/power | `ISGeneratorInfoWindow`/`ISGeneratorInfoAction`, `zombie.iso.objects` generator class | power is a zone/square flag - `setHaveElectricity`-style calls + sandbox `ElecShutModifier` |
| Traps | trap items + `AddExplosiveTrapPacket` (MP), `zombie.iso.objects` trap classes | new traps = item + object + animal-catch entries |
| Fire | `IsoFireManager` (`explode`, fire objects on squares) | fire spreads via the manager; custom incendiary = `explode` + fire objects |

Pattern that covers all of them: **an `SGlobalObjectSystem` subclass** (server state, save/load, MP sync)
+ a `CGlobalObject` client twin + context-menu interaction + a TimedAction for the long verb. That exact
quartet is how rain barrels, farming and generators all work - clone `CRainBarrelSystem` (it's the
smallest complete example) for any new persistent world system.

## 4. Time, weather and world clocks that drive these systems
`getClimateManager`-style climate APIs + `IsoWorld:getWeather`/`setWeather`/`transmitWeather`, season
tables, `getGameTime` minutes - the survival loops all key off these; your systems should too (don't
invent a second clock).

## Gotchas
1. **`Stats.set` clamps to 0..1** - needs are normalized; store custom ones un-clamped on ModData.
2. **Weight isn't a stat** - it's `Nutrition.getWeight` driven by calories + macros + trait flags; setting
   `Stats` won't move it.
3. **Thermoregulator multipliers stack** - write multipliers, not absolute temperatures, or you'll fight
   the clothing system.
4. **Fishing/farming are state machines, not events** - extend by adding states/crops to their tables,
   not by hooking `OnTick` globally.
5. **Traps explode via packets in MP** - `AddExplosiveTrapPacket` exists; if your trap mod bypasses the
   packet layer it silently fails on servers.
6. **`registerWaterZone`/`registerWaterFlow` are metazone calls** - fishable/swimmable water is world data,
   works on unloaded chunks.
7. **`canAddFitnessXp` gates fitness XP** - a mod that wants "no gym gains" or double XP hooks here, not
   the XP system.
8. **Body fluids/energy reads exist but write paths are thin** - sweating/dehydration beyond vanilla is a
   ModData layer on top, not a setter.

## Verification
- javap-verified 42.21.0: `Nutrition`, `Metabolics`, `Thermoregulator`, `Fitness` method names above.
- Verified in vanilla: Farming/Fishing/forage/rain-barrel Lua systems exist at the paths cited; generator
  UI + IsoFireManager used by vanilla.
- Not verified in-game: writing Thermoregulator multipliers from Lua, custom `SGlobalObjectSystem`
  subclassing beyond the documented pattern (community mods do it), `getBodyFluids` write semantics.

## `forageData` - resolved (was an open question)
`shared/Foraging/` is a complete data-driven system: `forageSystem.defaultDefinitions` in
`forageDefinitions.lua` documents every field inline. Three definition tables:
- **Item defs** (`forageSystem.addItemDef`-family): `type`, `minCount`/`maxCount`, `skill` + `perks{}`
  (averaged gate), `xp`, `recipes`/`traits`/`itemTags` visibility requirements, `categories{}`,
  weather modifiers (`rainChance`/`hasRainedChance`/`snowChance`/`dayChance`/`nightChance` percents),
  `zones = { Forest = n, DeepForest = n, ... }` (**n = roll count, NOT a percent**), `months`/
  `validMonths`/`bonusMonths`/`malusMonths`, `spawnFuncs{}` (custom pickup callbacks), `forceOutside`/
  `isOnWater`/`forceOnWater`/`canBeOnTreeSquare`, `doIsoMarkerSprite` (custom marker sprites!),
  poison fields (`poisonChance`/`poisonPowerMin/Max`/`poisonDetectionLevel` - foraged poison is a
  data feature), `itemSizeModifier` vision-radius tuning.
- **Zone defs**: `name` (zone type - DeepForest/FarmLand/TownZone/Nav/ForagingNav...), `densityMin/Max`
  icons per zone, `refillPercent` per game day, `abundanceSetting` (sandbox key `NatureAbundance`
  multiplies density), `containsBiomes` (compose zone loot from other zones' tables).
- **Category defs**: `chance` (rolls), `identifyCategoryPerk`/`identifyCategoryLevel`, `categoryHidden`,
  `validFloors`/`validFunc` (floor-type rules), `focusChanceMin/Max` (the search-focus reroll rates),
  plus the same weather/month modifiers.
Zone state lives in `ModData` `"forageData"` (`forageClient.addZone`/`updateZone`/`updateIcon`,
`triggerEvent("onUpdateIcon", ...)`) - zone icons are ModData-persisted, server-synced objects.
Custom forageable content = a new item def + optionally a category + `ISForageIcon` presentation -
all shared/ data, no Java needed.

## Open questions
- Whether `Thermoregulator_tryouts` (the experimental sibling class in the jar) is live or dead code.
- Exercise `FitnessExercise` script-data registration surface.
