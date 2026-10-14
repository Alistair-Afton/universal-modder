---
kind: game
title: 'Build 42 script system census: items, craftRecipes, entities, vehicles, traits, fluids - every block type'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); block census counted on media/scripts of that install'
platform: windows
engine: java
route: loader-api
tools: ["Project Zomboid script system (media/scripts *.txt)", "PZwiki ScriptsDocs"]
anti_cheat: 'none (data modding; no client patching)'
status: in-progress
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Scripts"
- "https://pzwiki.net/wiki/CraftRecipe_(scripts)"
tags: [b42, scripts, items, recipes, craftrecipe, entities, vehicles, traits, professions, fluids, models, sounds, fixing, mannequin, timedaction, data-oriented]
---
# Build 42 script system census: items, craftRecipes, entities, vehicles, traits, fluids - every block type

> Build 42 defines almost all game content in `media/scripts/**/*.txt` block files - no Lua needed for items,
> recipes, craft stations, vehicles, traits, professions, fluids, sounds or models. This note is a census of every
> top-level block type in a 42.21.0 install (counts included so you can judge how "supported" a feature is),
> the syntax patterns they share, and which vanilla file to copy per content type. Read after
> `b42-modding-anatomy-...md`.

## Setup
- Project Zomboid **42.21.0**, Steam, Windows 10. Census: `media/scripts/` of the install - `entities/` (239
  craft-station/world entities), `generated/` (items, recipes, characters, sounds, vehicles, weapons, physics),
  `ragdolls/`, `xui/` (UI skins).
- Mods add the same blocks from `media/scripts/*.txt` inside their own `common/` or version folder; a mod script
  uses `module Base` to share the vanilla namespace or its own `module MyMod` for namespaced ids
  (`MyMod.MyItem`). `imports { Base }` inside a module pulls vanilla blocks for `base:` type references.

## Route and why
Data-only modding. Before writing any Lua, check whether your content is a script block: items, recipes,
stations, vehicles, sounds, models, traits and professions are pure data in B42. Lua is only for *behaviour*.

## How the game works

### Block census (42.21.0 install, `media/scripts`)
| Block | ~Count | What it is | Vanilla files to copy |
|---|---|---|---|
| `item` | 5105 | Every inventory item | `generated/items/{weapon,food,clothing,drainable,container,literature,normal,...}.txt` |
| `model` | 3835 | Model entries (mesh + shader + animationsMesh) | `generated/vehicles/*_model.txt`, HLMonsters `model_houndeye.txt` |
| `sound` | 3038 | Named sounds w/ 3D falloff, categories, clip variants | `generated/sounds/**` |
| `craftRecipe` | 969 | New B42 crafting recipes | `generated/recipes/recipes_*.txt` |
| `xuiSkin` | 254 | Skins for the entity UI system | `entities/**/entity_*_xuiSkin.txt`, `scripts/xui/` |
| `vehicle` | 241 | Vehicles incl. templates | `generated/vehicles/` + `template_*.txt` |
| `entity` | 239 | B42 placeable workstations/objects with components | `entities/<domain>/workstations/*.txt` |
| `timedAction` | 119 | Script-declared timed actions | `generated/...`, `shared/TimedActions` Lua |
| `character_trait_definition` | 97 | Traits (cost, exclusions, XP boosts, granted recipes) | `generated/characters/character_traits.txt` |
| `physicsShape` | 90 | Physics colliders | `generated/physics/` |
| `template` | 77 | Reusable block templates | `vehicles/template_*.txt` |
| `animationsMesh` | 75 | Maps model -> animation folder/prefix | `model_*.txt` files, HLMonsters example |
| `ragdoll` | 66 | Ragdoll definitions | `scripts/ragdolls/` |
| `evolvedrecipe` | 63 | Cooking evolved recipes (add ingredients) | `generated/recipes/` |
| `fluid` | 61 | B42 fluid types | `generated/` (search `fluid `) |
| `mannequin` | 30 | Mannequin outfits | `generated/` |
| `fixing` | 29 | Item repair recipes | `generated/` (search `fixing `) |
| `character_profession_definition` | 25 | Professions | `generated/characters/character_professions.txt` |
| `physicsHitReaction` | 10 | Physics hit reactions | `generated/physics/` |
| `clock`, `energy`, `soundTimeline`, `vehicleEngineRPM`, `animation`, `xuiConfig` | ~20 | Misc subsystems | scattered |
| | | Also seen: `module`, `imports`, `mappers`/`itemMapper` sub-blocks | |

### Shared syntax patterns
- **Block = `name Id { key = value, ... }`** inside `module X { }`. Values with `;` are lists.
- **Item references**: `[Base.Axe;Base.Saw]` explicit list; `tags[base:saw;base:crudesaw]` any item whose
  `Tags` include those; `mapper:name` output resolved through an `itemMapper name { Out = In, default = ... }`.
- **Recipe flags on inputs**: `mode:keep` (tool isn't consumed) vs `mode:destroy`/default consumed;
  `flags[Prop1;Prop2]` = which hand/hotbar slot the ingredient takes in the craft UI;
  `flags[MayDegradeLight]` small condition loss; `flags[InheritCondition]` output inherits input condition.
- **`template! = Name`** = inherit a `template` block and override keys (vehicle variants use this heavily).
- **Icons**: `Icon = Foo` resolves `media/textures/Item_Foo.png`; `IconsForTexture` lists variants.
- Every display string is a translation key (`DisplayName = ItemName_Base.Axe`, `Tooltip`, `UIName`) resolved
  via `media/lua/shared/Translate/<LANG>/*.json` (see the Lua/runtime note).

### Anatomy per block (what modders actually write)
**item** - type via `ItemType = base:weapon|food|clothing|drainable|normal|...`; combat stats for weapons
(`MaxDamage`, `MinRange/MaxRange`, `SwingAnim`, `Categories = base:blunt`, `DamageCategory = Slash`,
`CriticalChance`, `KnockdownMod`, `ConditionMax`/`ConditionLowerChanceOneIn`); sounds as `HitSound`, `SwingSound`,
`BreakSound`, `DropSound`; behaviour tags (`Tags`, `DisplayCategory`, `Weight`, `ClothingItem`,
`BodyLocation`, `AttachmentType` for hotbar slots, `EatType`, `PourType`, `ReplaceOnUse`, `UseDelta`).

**craftRecipe** (B42 format - replaces B41 `recipe`):
```
craftRecipe SawLogs {
  timedAction = SawLogs,         -- links to a timedAction script/Lua class
  time = 230,                    -- action length
  Tags = InHandCraft;CanBeDoneFromFloor,
  category = Carpentry,          -- craft UI tab
  xpAward = Woodwork:5,
  inputs { item 1 [Base.Log] flags[Prop2],
           item 1 tags[base:saw] mode:keep flags[MayDegradeLight;Prop1], }
  outputs { item 3 Base.Plank, }
}
```
`itemMapper` blocks map input->output variants (SawLongStick maps LongStick->WoodenStick2, Sapling->TreeBranch2,
with `default`). Recipes on a station list in the entity's `CraftBench` component instead of being global.

**entity** - B42's component-based placeables:
```
entity Charcoal_Burner_MetalDrum {
  component UiConfig { xuiSkin = default, entityStyle = ES_Charcoal_Burner, uiEnabled = true, }
  component CraftBench { Recipes = WoodCharcoal, }
  component SpriteConfig { health = 100, skillBaseHealth = 20,
    face SINGLE { layer { row = crafted_02_51, } } }
  component CraftRecipe { timedAction = BuildLowNoTool, time = 50, category = Blacksmithing,
    Tooltip = Tooltip_craft_charcoalBurnerDesc,
    inputs { item 1 [Base.MetalDrum], item 4 [Base.Stone2], } }
}
```
Components seen: `UiConfig`, `CraftBench`, `SpriteConfig` (health + which sprite rows per facing),
`CraftRecipe` (what building it costs), plus fluid-container, feeding-trough and workstation components in
`entities/` subfolders (agricultural, animals, appliances, barricades, blacksmith, fences_low, furniture,
misc, outdoors, pottery, stairs, walls). An entity pairs its `entity` script with an `entity_*_xuiSkin.txt`
for the interaction UI and an `entityStyle` in the xui skin files.

**vehicle** - `model { file, scale, offset }`, `skin { texture }` (multiple = paint skins), `extents`,
`mass`, `centerOfMassOffset`, `physicsChassisShape`, `shadowExtents/Offset`, `attachment trailer|trailerfront|
seat|door...` with offset/rotate/zoffset, `crawlThroughWheel FrontLeft|...` (front, offset, radius, width),
`template! = CarNormal` to clone a base template. Parts/engine/sounds come from `template_*.txt` +
`vehicleEngineRPM` blocks; skins are `media/textures/Vehicles/` PNGs.

**character_trait_definition / character_profession_definition** - `Cost` (negative = gives points),
`MutuallyExclusiveTraits = base:x;base:y`, `XPBoosts = Skill=level`, `GrantedRecipes = A;B`,
`IsProfessionTrait`, `DisabledInMultiplayer`, `UIName`/`UIDescription` (`UI_trait_*` translation keys).
Professions add `FreeClothing`, spawn points, starting items.

**fluid** - B42's liquid system (beverages, mixtures, containers); `evolvedrecipe` - soup/stir-fry style
cooking additions; `fixing` - repair recipes (`require`, `fixer`, `globalItem`); `mannequin` - outfit display
entities; `timedAction`/`animation`/`soundTimeline` - script-side declarations consumed by the
`media/lua/shared|client/TimedActions` Lua classes.

### items.xml
`media/items/items.xml` is a flat registry the game generates/reads - not the authoring format. Author in
script blocks; ignore it.

## Build steps
1. Decide the block type; find the nearest vanilla file in `media/scripts/generated/` (or `entities/` for
   stations) and copy its shape into `media/scripts/mymod_<thing>.txt`.
2. Use your own `module` for namespaced ids, or `module Base` to merge into the vanilla namespace (needed when
   vanilla systems must find it by unqualified name, e.g. `Base.MyItem` still works via module prefix).
3. Add translation keys (`ItemName_`, `Recipes_`, `Tooltip_`, `UI_trait_`, `Sandbox_`) in
   `media/lua/shared/Translate/EN/*.json`.
4. Icons/textures under `media/textures/` with the `Item_` prefix convention.
5. Launch `-debug`; script parse errors land in `console.txt` with file+line. Hot-reload scripts from the debug
   menu where supported; otherwise restart.

## Verification
- `console.txt` script errors are precise (module, block, key). A failed block usually kills only that file's
  remaining blocks.
- Craft recipes: check the crafting UI tab named by `category`; entities: `ES_*` styles must exist or the
  entity UI is blank; vehicles: `-debug` vehicle editor/spawner.
- Not verified: hand-building a `xuiSkin` from scratch (copied vanilla skins only); `clock`/`energy` blocks'
  mod-facing use.

## Gotchas
1. **Recipe doesn't show in UI.** Cause: missing `category` translation key or `Tags` gating
   (`InHandCraft`/`CanBeDoneFromFloor` vs bench-only). Fix: give the category a `Recipes_*`/UI translation and
   check tags match where you want it craftable.
2. **Tool got consumed.** Cause: no `mode:keep`. Fix: `item 1 tags[base:saw] mode:keep ...`.
3. **Output is wrong variant.** Cause: mappers need the input listed in `mappers[...]` AND an `itemMapper`
   block with `default`; unmapped inputs fall through to default.
4. **Icon invisible.** `Icon = X` needs `media/textures/Item_X.png`; world model needs
   `WorldStaticModel = ` pointing at a `model` block or ground sprite.
5. **`base:` tag references fail from a custom module** without `imports { Base }`.
6. **Vehicle spawns as a cube.** `model.file` must reference a `model` block (`Vehicles_X`), which points at
   `mesh = vehicles/...` in `models_X`; scale mismatch (vanilla uses `scale = 0.01` on meshes, `scale = 1.82`
   on the vehicle model block) distorts collision.
7. **Entity builds but has no UI.** `component UiConfig` needs a valid `entityStyle` defined in the xui skin
   files; copy a vanilla `entity_*_xuiSkin.txt` pair, not just the `entity` block.
8. **Trait does nothing.** `CharacterTrait = base:x` is the trait id the Lua/perk system checks; `UIName` is
   only display. Mod effects still need Lua (`player:getTraits():contains("x")` or `HasTrait`).
9. **Overriding a vanilla item**: redeclaring `item Base.Foo` in `module Base` overrides loaded fields - last
   load order wins; safer to patch values at `OnGameBoot` via `getScriptManager():getItem(...)`.
10. **Fluids need both ends**: a `fluid` block AND container/food items declaring fluid capacity + the
    `fluids` item field; copying just one half shows an empty container that can't fill.

## Assets
- Items/vehicles/entities are pure script + textures/models; vehicle skins are palette PNGs under
  `textures/Vehicles/`; entity sprites come from tiledefs (`crafted_02_51`-style sprite names from .pack/.tiles).

## Open questions
- Full component list for `entity` beyond the ~8 types in vanilla `entities/` (undocumented ones likely exist;
  `javap` on `zombie.scripting.entity` would enumerate them).
- Whether `timedAction` script blocks can fully replace writing an `ISBaseTimedAction` Lua class for custom
  actions.
- `mannequin` block fields beyond outfit+pose.
