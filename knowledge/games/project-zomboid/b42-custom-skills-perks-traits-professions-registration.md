---
kind: game
title: 'Custom skills, perks, traits and professions: the resolved PerkFactory.AddPerk contract, CustomPerks.initLua, XP curves and character-creation hooks'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); javap-verified signatures'
platform: windows
engine: java
route: loader-api
tools: ["javap (JDK)", "PerkFactory.AddPerk", "CustomPerks.initLua", "media/scripts trait/profession blocks", "CharacterCreationProfession.lua"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Lua_(API)"
tags: [b42, skills, perks, traits, professions, xp, perkfactory, customperk, character-creation]
---
# Custom skills, perks, traits and professions: the resolved contract (42.21.0)

> Earlier notes flagged "is a custom skill even possible?" as an open question. It is - the API is
> javap-verified: `PerkFactory.AddPerk` exists with real signatures, and a dedicated `CustomPerks`
> class with `initLua` exists purely for mod registration. This note resolves the contract end-to-end
> and maps the trait/profession/XP side.

## Setup
- javap-verified 42.21.0. Builds on the character-hooks note (SurvivorSwap/XP surface).

## Route and why
Loader API - registration happens in Lua at `initLua` time.

## The resolved `AddPerk` signatures (javap-verified)
```java
static Perk AddPerk(Perk perk, String name,
                    int a1, int a2, int a3, int a4, int a5,
                    int a6, int a7, int a8, int a9)
static Perk AddPerk(Perk perk, String name,
                    int a1..a9, boolean passive)
static Perk AddPerk(Perk perk, String name, Perk parent,
                    int a1..a9)
static Perk AddPerk(Perk perk, String name, Perk parent,
                    int a1..a9, boolean passive)
```
- `Perk` (an enum-like registry entry) + `getPerk`/`getPerkFromName`/`getPerkName` - the registry.
- The 9 ints are the per-level XP curve thresholds (vanilla skills are level-1..10; the ints parametrize
  the required XP per level - confirm exact ordering against `zombie.characters.skills` decompile, the
  signature is verbatim).
- `parent` overload = skill trees/derived perks (e.g. aiming -> marksman-style parents).
- `passive` flag distinguishes passive perks.
- **`CustomPerks`** (jar-verified): `CustomPerks.instance`, `init()`, `initLua()`, `Reset()` +
  `CustomPerk` - the dedicated mod-registration path; call `AddPerk` from the Lua init path
  (`CustomPerks.initLua` is the engine-side hook that runs your registration).

## Registering a custom skill - the recipe
1. In `shared/` Lua: `PerkFactory.AddPerk(PerkFactory.Perks.<existing-similar-or-nil>, "ModSkill", ...)`
   via the `CustomPerks.initLua` path; the returned `Perk` is your handle.
2. Levels: `player:setPerkLevelDebug(perk, n)`/`getPerkLevel` (cheat-sheet surface); XP:
   `getXp():AddXP(perk, n)` + `getXp():getXP(perk)`; multipliers via `AddXPMultiplierPacket` (MP) and
   perk-linked XP tables.
3. UI: `ISPlayerStatsUI`/`ISPlayerStatsAddXPUI`/`ISCharacterInfo` already enumerate `PerkFactory` -
   registered perks appear in the skills panel automatically (it's data-driven).
4. Effects: read `getPerkLevel` in your systems (craft speed, accuracy modifiers, unlock gates) - perks
   don't DO anything alone; they're a stat your mod consults.
5. Localization: `initTranslations` on the factory - skill names come from the perk name + translation
   keys (`Perks_<name>`-family in `Translate/`).

## Traits and professions
- Trait/profession definitions live in script data (`trait`/`profession` script blocks - see the
  script-census note; `media/scripts` contains the vanilla set) + creation UI in
  `client/OptionScreens/CharacterCreationProfession.lua`.
- A profession = trait bundle + free perk levels + spawn items/clothing + spawnpoints (maps note) +
  optional `OnNewGame`-family hooks; a trait = point cost + modifiers (`addXPBoost`,
  `FreeRecipes`, stat offsets, `getCharacterTraits` entries).
- Character-creation hooks: `OnCreateLivingCharacter`/profession-screen events give you the new-
  character moment; `SurvivorSwap` (character-hooks note) rewrites an existing character if you need
  post-creation changes.
- Custom trait/profession effects route the same way as skills: the definition grants flags/boosts;
  your Lua reads `getCharacterTraits`/`getDescriptor`/`hasTrait` and applies the behaviour.

## XP and levelling internals
- `Xp` object (`playerObj:getXp()`): `AddXP`, `getXP`, per-perk multipliers, `getPerkBoost`/`AddXPBoosted`,
  level-up events (`levelUp` on IsoPlayer + `playGainExperienceLevelSound`).
- XP multipliers: `AddXPMultiplierPacket` exists in the packet layer (MP-safe XP mods); sandbox
  `XPMultiplier` globals already scale per-perk.
- `canAddFitnessXp` (body-sim note) is the fitness-specific gate - custom "does this count" logic
  hooks there.
- Custom progression doesn't have to be a Perk at all - ModData counters + a custom UI work, but
  `AddPerk` buys you the skills panel, sandbox multiplier integration, save/load and MP for free.

## Gotchas
1. **`AddPerk` is additive, not a script block** - no `perk {}` script type; skills register from Lua at
   initLua (hence `CustomPerks.initLua` existing).
2. **The 9-int curve ordering isn't named** - match a vanilla perk's decompiled `AddPerk` call for the
   field semantics (they're level thresholds, but get the direction right).
3. **Traits/professions are script data but their *effects* are hardcoded trait IDs** - a custom trait
   gets custom behaviour only via your Lua reading `hasTrait`; stock effects (organized, strong...) are
   enum-bound.
4. **`getPerkFromName` lookups are string-keyed** - stable IDs across saves, but typos return nil;
   store the `Perk` handle, don't re-lookup per tick.
5. **Perk level >10 / negative**: `setPerkLevelDebug` is a debug call for a reason - clamp your own
   writes; production level changes go through XP.
6. **MP**: perk/XP changes must come from the server or a synced path (`AddXPMultiplierPacket` exists);
   client-side `AddXP` desyncs.
7. **Skills panel auto-lists registered perks** - a bad registration pollutes everyone's UI; gate it
   behind a sandbox option during dev.

## Verification
- javap-verified 42.21.0: `AddPerk` signatures above (verbatim), `CustomPerks.instance`/`init`/`initLua`,
  `CustomPerk`, `getPerk`/`getPerkFromName`/`getPerkName`/`initTranslations`.
- Verified in the wild: `ISPlayerStatsUI`/`ISCharacterInfo`/`ISPlayerStatsAddXPUI` enumerate perks
  (visible in the shipped Lua), `CharacterCreationProfession.lua` is the creation screen.
- Not verified in-game: the int-field ordering of `AddPerk`, `CustomPerk` field surface (javap shows a
  ctor; full dump pending a focused session), whether `parent` perks enforce ordering.

## Open questions
- Exact semantic per `AddPerk` int arg (level thresholds vs multipliers - decompile one vanilla call).
- `CustomPerk` field surface (icon? boost fields?).
- Whether trait script blocks support custom trait *effects* beyond flagging (likely Lua-consulted
  only, matching the enum-bound stock effects).
