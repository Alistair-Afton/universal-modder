---
kind: game
title: 'Mod interop: load order, require=/mod.info dependencies, shared Lua globals as APIs, conflict patterns and building mods that cooperate'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10)'
platform: windows
engine: java
route: loader-api
tools: ["mod.info require= key", "shared Lua globals", "OnGameBoot/OnGameTimeLoaded ordering", "mod hooks"]
anti_cheat: 'none'
status: in-progress
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Mod.info"
tags: [b42, interop, dependencies, load-order, api, compat, shared, globals]
---
# Mod interop: dependencies, shared globals and building mods that cooperate (42.21.0)

> Single-mod docs are done; this is the "mods as an ecosystem" layer: how load order actually works,
> how one mod exposes an API another consumes, how to soft-depend, and the collision patterns that
> break real load orders.

## Setup
- `mod.info` fields from the anatomy note (`require=`, `id=`, `name=`, `versionMin/Max`, `poster`,
  `modversion`). Lua loads shared->client->server per the runtime note.

## Route and why
Loader API - the interop surface is mod.info + Lua globals.

## 1. Dependencies and load order
- `mod.info` `require = modA,modB` - hard dependencies; the game loads them first and blocks enabling
  without them. `id=` is your namespace for others to require.
- Load order within a set: dependency-first, then listed order - **shared/ files of all mods load
  before client/server**, and per-file load order across mods follows the mod list order; same-path
  files override (the media/lua override-by-path rule from the anatomy note).
- `OnGameBoot`/`OnGameTimeLoaded`/`OnGameStart` ordering: events fire for every subscriber in
  subscription order - a mod that must run first should subscribe in shared/ at require time, not
  inside a later event.

## 2. Exposing an API to other mods
- The Lua global table IS the public API: `MyMod = MyMod or {}; function MyMod.doThing(...) end` in
  `shared/` - any mod that loads after you can call `MyMod.doThing`. That's the whole mechanism.
- Version it: `MyMod.API_VERSION = 3` so dependents can gate features; document your globals like a
  public API (they are - you can't take them back).
- Consumption pattern (soft dep): `if MyMod and MyMod.API_VERSION and MyMod.API_VERSION >= 2 then
  MyMod.registerThing(...) end` - missing mod = skip, not crash.
- Hard dep: `require=` in mod.info instead - load error tells the user what's missing.
- Reverse callbacks > polling: `MyMod.registerHook(fn)` / `MyMod.onEvent:add(fn)`-style registries let
  dependents inject without wrapping your functions (wrapping `require`'d module functions is the
  fragile alternative).

## 3. The real conflict surfaces
- **Same-path file override**: two mods shipping `media/lua/.../SameFile.lua` - last-loaded wins,
  first mod breaks silently. Namespace your Lua paths (`media/lua/shared/ModName/...`) - vanilla
  convention and the #1 avoidable conflict.
- **Script block name collisions**: items/recipes/entities share global namespaces (`module Base {`
  vs `module MyMod {`)- use your own module name in script blocks.
- **Script block `mode:keep`/`mode:override`** (census note): keep merges, override replaces - two
  mods overriding the same vanilla block collide; prefer keep/additive patterns.
- **Shared script objects**: `ScriptManager` registries are global - two mods registering "Sword" in
  different modules coexist; same module+name = last wins.
- **Events**: subscribing to a hot event (`OnZombieUpdate`, `OnTick`) twice per mod or stacking
  expensive handlers across mods compounds into frame cost - the perf patterns in the creature notes
  exist partly for this.
- **ModData keys**: namespaces! `modData["MyMod_foo"]` or your own table under a mod-namespaced key -
  flat keys collide across mods.
- **Outfit IDs, sound names, clothingItem GUIDs**: all global - prefix with your mod tag.
- **Sandbox option names**: `sandbox-options.txt` vars land in `SandboxVars` - prefix
  (`SandboxVars.MyMod.Option`).

## 4. Compatibility patterns that work
- Feature-detect, don't mod-detect: `if type(SomeGlobal) == "table"` beats
  `if getActivatedMods():contains("modid")` (mod ids vary by folder/workshop id).
- Late-init hooks for cross-mod registration: do your "register with other mods" pass in
  `OnGameTimeLoaded`/`OnGameStart` when all shared/ files exist, not at file load.
- Keep a compat shim file per dependency: `MyMod/compat/TargetMod.lua` containing every touch of the
  other mod's globals - isolates the fragile surface to one file when the other mod changes.
- Ship a public `API.md` or a `MyMod/api.lua` file - the community pattern (CoF exposes registration
  hooks; other mods add monsters through it).

## 5. Versioning and workshop
- `versionMin`/`versionMax` in mod.info gates game versions; `modversion` is your semver surface.
- Workshop update flow: `workshop.txt` content_id - updates redistribute through Steam; breaking API
  changes need a major version + migration note or dependent mods break on update day.

## Gotchas
1. **`require=` order isn't guaranteed across equal-depth mods** - if load order matters beyond deps,
   control it via the chain or late-init, not list order.
2. **Globals are shared, not sandboxed** - another mod can monkey-patch your functions; defensively
   re-read `MyMod.fn` at call time if it matters.
3. **Mod id vs workshop id vs folder name** are three different strings - `mod.info` `id=` is the
   require key; the folder/workshop id isn't.
4. **Disabling a mod mid-save leaves its ModData** - save contamination is real; document a clean
   removal path (your ModData, spawned objects, entity registrations).
5. **`require` cycles aren't possible** - the loader rejects them; design APIs one-directional.
6. **Same Lua file-path overrides are silent** - no warning; the losing mod just doesn't run.
7. **Events subscribed inside other event handlers subscribe late** - deterministic ordering wants
   top-level `Events.X:add` in shared/.
8. **`getActivatedMods()`** exists for hard checks but is load-order dependent - prefer feature
   detection.

## Verification
- Verified in the wild: `require=` semantics (mod.info docs + workshop mods use it), file-override rule
  (the anatomy note's media/lua rules), `MyMod`-global API pattern (COF exposes hooks other mods use -
  its own files document it), ModData/sandbox namespacing (every shipped mod follows it).
- Not verified this session: exact cross-mod load-order tie-breaking beyond dependency chains,
  `getActivatedMods` behavior on dedi.

## Open questions
- Whether `mod.info` supports optional/soft `require` semantics beyond hard deps.
- Exact load-order algorithm among independent mods (alphabetical? list order? - worth one focused
  test if a project depends on it).
