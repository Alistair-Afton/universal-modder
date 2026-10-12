---
kind: game
title: 'The B42 entity system: GameEntity components, fluids, crafting logic (furnaces/drying/mashing), resources/logistics, Lua components - the real ECS'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); jar/javap-verified'
platform: windows
engine: java
route: loader-api
tools: ["javap (JDK)", "media/scripts/entities/ (vanilla component data)", "zombie.entity.* package"]
anti_cheat: 'none'
status: in-progress
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Entity_scripts"
tags: [b42, entity, ecs, components, fluids, crafting-bench, furnace, drying, mashing, resources, logistics, workstation, gameentity]
---
# The B42 entity system: the real component runtime (42.21.0)

> Correction to earlier guesses: B42's "ECS" is NOT the tiny `zombie.characters.ecs` scaffold (6 classes,
> lifecycle glue). The real component system is **`zombie.entity`** - a full entity/component/simulation
> runtime behind the `entity` script blocks. Every new B42 world object that *does* something - crafting
> benches, furnaces, drying racks, mashing stations, fluid containers, appliances, animal fixtures -
> is a `GameEntity` assembled from components. This is how you build complex interactive systems as data.

## Setup
- javap/jar-verified on 42.21.0. Vanilla entity data: `media/scripts/entities/` with subfolders
  `admin agricultural animals appliances barricades blacksmith fences_low furniture misc outdoors
  stairs walls` - the folders themselves show what vanilla builds with it.

## Route and why
Loader API - `entity` blocks in `media/scripts/entities/...` are first-class script data
(`ScriptManager:getGameEntityTemplate`/`getAllGameEntityTemplates` from the cheat sheet).

## The runtime map (`zombie.entity`, jar-verified)
Core: `GameEntity`, `GameEntityFactory`, `GameEntityManager`, `GameEntityNetwork` (MP sync built in),
`GameEntityType`, `Engine`/`EngineEntityManager`/`EngineSystem`, `SystemManager`/`EntitySimulation`,
`EntityBucket`/`EntityBucketManager`/`CustomBuckets` (spatial/state buckets - entities are batched!),
`MetaEntity`/`MetaEntitySystem` (unloaded-chunk twins, like metazones), `Component`/`ComponentFactory`/
`ComponentScriptFactory`/`ComponentType`/`ComponentContainer`/`ComponentOperationHandler`,
`ComponentOperationHandler`, `Family`, `EntityDebugger`, `InventoryItemSystem`, `UsingPlayerUpdateSystem`,
`CFlags`.

## The component catalogue (`zombie.entity.components.*`, jar-verified)
| Package | Key classes | What it makes |
|---|---|---|
| `attributes` | `Attribute`, `AttributeType`, `AttributeContainer`, `AttributeValueType` | typed key/value state on entities |
| `build` | `BuildLogic` | construction/buildable behaviour |
| `combat` | `Durability`, `DurabilityScript` | damageable entities (walls, barricades) |
| `contextmenuconfig` | `ContextMenuConfig` | data-driven right-click menus |
| `crafting` | `CraftBench`, `CraftLogic`/`BaseCraftingLogic`/`CraftLogicSystem`, `FurnaceLogic(System)`, `DryingLogic(System)`, `MashingLogic(System)`, `CraftRecipeComponent`, `CraftRecipeMonitor`, `InputFlag`/`OutputFlag`, `FluidMatchMode`, `ItemApplyMode`, `StartMode`, `CraftMode`, `WallCoveringConfig` | **workstations**: benches, furnaces, drying racks, mashing - recipes with item+fluid I/O, timed processes |
| `fluids` | `Fluid`, `FluidContainer`, `FluidType`, `FluidProperties`, `FluidConsume`, `FluidFilter`, `FluidInstance`, `FluidSample`, `SealedFluidProperties`, `PoisonEffect`/`PoisonInfo`, `FluidContainerUpdateSystem` | tanks/barrels/pots: capacity, mixing, poison, rain-catching |
| `lua` | `LuaComponent` | **script-attached component - your custom entity behaviour hook** |
| `parts` | `Parts` | installable sub-parts on entities |
| `resources` | `Resource`, `ResourceBlueprint`, `ResourceChannel`, `ResourceEnergy`, `ResourceFluid`, `ResourceFlag`, `LogisticsSystem` | **energy/fluid/resource networks** between entities |
| `script` | (script bindings) | script-defined components |
| `signals` | signal plumbing | component-to-component events |
| `sounds` | entity sound config | looped/positional sounds |
| `spriteconfig` | sprite binding/variants | which sprites render the entity |
| `ui` | entity UI config (`ISFluidContainerMenu` is the vanilla face) | data-driven interaction UIs |

## `FluidContainer` API (javap-verified) - the liquid system's actual calls
`addFluid`, `adjustAmount`, `adjustSpecificFluidAmount`, `canAddFluid`, `getAmount`, `getCapacity`,
`getFilledRatio`, `getFreeCapacity`, `getPrimaryFluid`/`getPrimaryFluidAmount`, `getRainCatcher`,
`getPoisonEffect`/`getPoisonRatio`, `getColor`, `getContainerName`, `getCustomDrinkSound`,
`getBlacklist`, `getProperties`, `createFluidSample`, `contains`, `copyFluidsFrom`,
`CanTransfer`/`GetTransferReason`/`Transfer`, `canPlayerEmpty`, `Empty`, `CreateContainer`,
`DisposeContainer`, `DoTooltip`. Fluids are typed (`Fluid`/`FluidType`), colored, poisonous, and
mixable - a brewing/alchemy/chemistry mod is component data + these calls.

## Building a custom entity
1. `entity` block in `media/scripts/entities/<folder>`: pick components (`component <Name>` entries) and
   fill their fields - copy the closest vanilla file (`blacksmith` for a furnace, `appliances` for a
   powered box, `agricultural`/`animals` for husbandry fixtures).
2. Crafting behaviour = a `crafting` component pointing at `craftRecipe`s + a logic type
   (`FurnaceLogic` burns fuel, `DryingLogic` time-decays, `MashingLogic` processes) - your recipe data
   does the rest (inputs/outputs/fluids from the script-census note).
3. Interactivity: `contextmenuconfig` + `ui` components give the menus; `LuaComponent`/`script`
   components take your Lua when stock components aren't enough.
4. World-facing: `spriteconfig` binds tiles/sprites, `sounds` loops ambience, `resources` wires
   power/fluids between entities (`LogisticsSystem` moves things between channels).
5. Save/MP: `GameEntityManager`/`GameEntityNetwork`/`MetaEntitySystem` handle persistence,
   unloaded chunks and sync - entities are the *supported* way to make complex objects MP-safe
   (compare hand-rolled `SGlobalObjectSystem`, which you still need for non-entity custom systems).

## When to use which
- **`entity` + components**: interactive world objects with stock behaviours (craft, store fluid, damage,
  power). Prefer this - it's data, it saves and syncs for free.
- **`SGlobalObjectSystem`**: behaviours no component covers (rain barrels are the canonical example).
- **IsoThumpable/object + Lua**: one-off destructible/pseudo-creature things (beyond-chassis note).
- **Items/recipes only**: no world presence needed.

## Gotchas
1. **`zombie.characters.ecs` ≠ `zombie.entity`** - the 6-class `ecs` package is lifecycle glue; the
   component runtime is `zombie.entity`. Docs/confusion elsewhere conflates them.
2. **Components are script-declared but Java-implemented** - you compose stock components and add
   `LuaComponent` for custom logic; you can't write new component *types* in Lua.
3. **`MetaEntity` twins matter on unload** - entities have a meta version offloaded with the chunk;
   long-range logic must run on meta, not the live entity.
4. **Fluid poison is a real field** (`PoisonEffect`/`PoisonInfo`) - poisoned-water gameplay is data, not
   a hack.
5. **`LogisticsSystem`/`ResourceChannel` are networks** - power/fluid distribution between entities is a
   supported pattern; standalone mods that hand-wire it duplicate the engine's own system.
6. **Entity script grammar is block-based** like items/vehicles, but component fields are per-component;
   the vanilla `entities/` folder IS the schema reference (there's no doc - read `blacksmith`).
7. **`contextmenuconfig` is data** - adding a verb to a stock entity's menu means editing its component
   data or going through `OnFillWorldObjectContextMenu` for non-entity objects.
8. **`EntityBucket` batching means per-entity tick costs are amortized** - entities scale better than
   equivalent `IsoObject`+Lua loops; prefer them for anything numerous (crops, barrels, fixtures).

## Verification
- javap/jar-verified 42.21.0: `zombie.entity` class list, component package list, `FluidContainer`
  method names, vanilla `entities/` folder tree.
- Verified in the wild: `ISFluidContainerMenu` exists in `media/lua/client/ISUI` (the fluid UI face),
  `DebugCSVExportFluidContainers` in `zombie.debug`, fluids used by vanilla recipes/beverages.
- Not verified in-game: writing a novel entity block end-to-end (the schema is vanilla-folder-reverse-
  engineered; no shipped mod entity was tested this session), `LuaComponent` exact script-binding
  grammar, `LogisticsSystem` mod-facing surface.

## Open questions
- Full `component <Name>` grammar per component type (decompile `ComponentScriptFactory` or diff vanilla
  files - the folder is the spec).
- `ResourceChannel`/`LogisticsSystem` exact semantics (energy grid? item transport?).
- Whether `GameEntityType` can be extended to entity categories vanilla doesn't use, or only
  re-composed from stock component types.
