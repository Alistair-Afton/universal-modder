---
kind: game
title: 'Build 42 Lua runtime in practice: events, timed actions, context menus, UI, ModOptions, outfits and spawn tables'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); all paths/APIs verified in the install media/lua and two live mods'
platform: windows
engine: java
route: loader-api
tools: ["Project Zomboid Lua (Kahlua) mod API", "PZ-Umbrella EmmyLua stubs", "PZAPI.ModOptions (vanilla mod-options UI)"]
anti_cheat: 'none (single-player; hosted servers run the same Lua)'
status: in-progress
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Lua_(API)"
- "https://pzwiki.net/wiki/Lua_event"
- "https://github.com/PZ-Umbrella/Umbrella"
tags: [b42, lua, events, timed-actions, context-menu, ui, pzapi, modoptions, outfits, spawn-tables, translations, debug, networking]
---
# Build 42 Lua runtime in practice: events, timed actions, context menus, UI, ModOptions, outfits and spawn tables

> The behaviour half of PZ modding (the data half is `b42-script-system-...md`): which Lua runs where and when,
> the event system, the vanilla patterns for timed actions / context menus / UI, the built-in
> `PZAPI.ModOptions` settings API, how zombie outfits and spawn tables are defined, and the debug workflow.
> Verified against `media/lua/` in 42.21.0 plus a 23-monster workshop mod and a custom-animal mod.

## Setup
- PZ **42.21.0** Steam/Windows. All vanilla Lua is readable in
  `<install>\media\lua\{shared,client,server}\` - it is the best API documentation that exists. `media/luaexamples/`
  has minimal examples (ISTimer, timed action, UI).

## Route and why
Loader API - this note is about the game's own Lua mod API itself (Kahlua + exposed Java). The alternatives
(Java patching, external tools) are unnecessary: everything behaviour-wise in Workshop mods goes through this
runtime.

## How the game works

### Load lifecycle
1. `shared/` runs first on every context (main menu boot AND world load - `OnGameBoot` fires for the menu-time
   pass; guard world-dependent code behind `OnLoad`/`getCell()` being non-nil).
2. Then `client/` (SP+MP client) or `server/` (SP internal server + dedicated). SP runs both.
3. `require 'X'` resolves `media/lua/shared/X.lua` (then other roots); modules in `shared/` are callable from
   both sides. Reload order between mods: load order; same-path files override.
4. Re-entry: returning to main menu and reloading a save re-runs `shared` - don't assume one boot per process.
   Register events once (guard with a flag) or the handler multiplies.

### Events
`Events.<Name>.Add(fn)` / `.Remove(fn)`. Families you'll actually use (full list in Umbrella stubs):
- Boot/tick: `OnGameBoot`, `OnTick`, `OnTickEvenPaused`, `EveryOneMinute`, `EveryDays`, `EveryHours`.
- World: `OnLoad`, `OnSave`, `OnInitWorld`, `OnFillWorldObjectContextMenu`, `OnRightMouseDown`,
  `OnObjectAdded`, `OnGridSquareLoaded`, `OnWorldSound`, `OnDeadBodySpawn`, `OnSeeNewRoom`.
- Characters: `OnZombieUpdate`, `OnZombieCreate`, `OnZombieDead`, `OnCharacterDeath`, `OnPlayerGetDamage`,
  `OnWeaponHitCharacter`, `OnWeaponSwing`, `OnPlayerUpdate`, `OnAnimalUpdate`-family (B42).
- Items/inventory: `OnFillInventoryObjectContextMenu`, `OnCreateItem`, `OnContainerUpdate`.
- Net (MP): `OnClientCommand` (server side), `OnServerCommand` (client side), `OnReceiveGlobalModData`,
  `onLoadModDataFromServer`, `OnServerStarted`, `OnConnected`.
- UI: `OnCreateUI`, `OnResolutionChange`, `OnCustomUIKey`/`OnKeyPressed`.
The event objects are Java objects - call their methods, never index fields.

### Timed actions (the "do something over N seconds" pattern)
- Base class `ISBaseTimedAction` (`media/lua/shared/TimedActions/` in B42; `client/TimedActions` for
  client-only ones like reading). Subclass and implement `isValid()` (rechecked while queued), `update()`
  (per-frame progress), `perform()` (fires when done - in B42 run authoritative effects server-side),
  `start()`/`stop()`. `self.item`, `self.character` come from `new(...)`.
- Queue it: `ISTimedActionQueue.add(MyAction:new(character, item, time))`. `time` in frames (~60/s) or use
  `getModifiedTime` helpers for speed modifiers.
- Script-side `timedAction` blocks (119 vanilla) declare the data; the Lua class does the work. Copy a near
  vanilla action (`ISSawLogsAction` for `SawLogs`) rather than starting blank.

### Context menus (right-click)
```lua
Events.OnFillWorldObjectContextMenu.Add(function(player, context, worldobjects, test)
    context:addOption(getText("ContextMenu_MyMod_DoThing"), square, MyMod.doThing, arg)
end)
```
`OnFillInventoryObjectContextMenu` for inventory right-clicks, `OnFillContainer` for loot. `test==true` means
"just checking" - don't do work. Vanilla `ContextMenuCode.lua` is the reference.

### UI
- Framework: `ISUI/` (ISPanel, ISButton, ISTickBox, ISComboBox, ISRichTextPanel...). Instantiate, `initialise()`,
  `addToUIManager()`. `DebugUIs/` is a goldmine of working examples; `PZAPI/ui/` too.
- **PZAPI.ModOptions** (`media/lua/client/PZAPI/ModOptions.lua`): vanilla in-game mod-settings UI. Create
  `PZAPI.ModOptions.Options:new(modID, name)`, `addTitle/addDescription/addSeparator/addTextEntry/
  addTickBox/addComboBox/...` with ids + tooltips, values read back via `getOption(id).value`. Saves per-mod
  options the player can change without editing sandbox - use this for client preferences; use
  `sandbox-options.txt` for world-affecting settings (which are part of the save).
- Translation: `getText("Key")` reads `Translate/<LANG>/*.json`; keys are namespaced strings
  (`IG_UI_`, `ContextMenu_`, `Tooltip_`, `ItemName_Base.X`).

### Zombie outfits and spawn tables
- **Outfits** live in `media/clothing/clothing.xml` (`outfitManager`): entries per gender,
  `<m_Name>OutfitName</m_Name>`, `<m_Guid>`, and `<m_items>` each holding an `<itemGUID>` (a clothingItem's
  GUID from `fileGuidTable.xml`), optional `<probability>0.8</probability>` and `<subItems>` alternates.
  Adding an outfit = adding an `m_MaleOutfits`/`m_FemaleOutfits` block; every item GUID must exist in your
  `fileGuidTable.xml` or the outfit fails silently.
- **Zone spawn tables**: `media/lua/shared/NPCs/ZombiesZoneDefinition.lua` - a plain Lua table mapping a
  WorldEd zone name to `chanceToSpawn`, `toSpawn`, and outfit entries with `name`, `toSpawn`, `gender`,
  `mandatory`, `room`, `chance` (weighted share of non-mandatory spawns). Mods extend it by adding keys to
  `ZombiesZoneDefinition` from a shared Lua file - that's how "special zombies in specific places" works
  (wedding party in a church is the vanilla example).
- Manual spawning: `addZombiesInOutfit(...)` (needs zombie population on) or
  `VirtualZombieManager.instance:createRealZombieNow` + `dressInPersistentOutfit` (server-side; see the
  dressed-zombie note for the full failure matrix).

### Networking (B42, server-authoritative)
- Since ~42.13 the server owns damage/item state. Write shared code that forks on `isClient()`/`isServer()`.
- Client->server: `sendClientCommand(module, command, argsTable)`; server listens on
  `Events.OnClientCommand.Add(function(module, command, playerObj, args) ... end)`.
- Server->client: `sendServerCommand(playerOrNil, module, command, args)`; clients listen on
  `OnServerCommand`. Args = plain data only (no Java objects - pass online IDs / coordinates).
- `ModData.transmit(key)` / `ModData.request(key)` sync global mod-data tables;
  `Events.OnReceiveGlobalModData` receives them (table may be `false` on a request reply).
- Vanilla reference implementations: `media/lua/client/ServerCommands.lua` + `server/` command tables, and
  the CoF mod's `COF_Net.lua` split.

### Debug workflow
- Launch option `-debug`: enables the debug menu (Lua debugger with breakpoints, script/anim reload, creature
  debug views, iso regions, item spawner, "cheat" toggles like god/invisible/no-clip).
- `Zomboid/console.txt` - all script parse errors (module+file+line) and Lua stack traces; accumulates
  sessions, so read from the last `loading <modid>`. Older sessions in `Zomboid/Logs/`.
- `print()` goes to console.txt; use a namespaced prefix (`MyMod[topic]`) and a sandbox/debug flag to gate
  verbose logging - it's how both studied mods diagnosed spawn bugs.
- In-game: right-click debug menus, the AnimSet/animator debug, and `DebugUIs` windows; `luaexamples/` shows
  ISTimer patterns for polling instead of OnTick.
- API drift check between builds: dump methods via `javap -p` or the reflection lister from the CDDA note and
  diff against your calls; 42.20->42.21 renamed at least `IsoGridSquare.Is`->`has`.

## Build steps
1. `media/lua/shared/MyMod/MyMod.lua` for definitions + `Events.OnGameBoot` registration guarded by a flag;
   `client/` for UI/context menus/controllers; `server/` for spawning/state changes.
2. Behaviour on zombies: `Events.OnZombieUpdate` (per-zombie per-frame - keep it O(1); see the CoF dispatch
   pattern for per-monster controllers).
3. Settings: `media/sandbox-options.txt` (world) + `Translate/EN/Sandbox.json`, and/or `PZAPI.ModOptions`
   (client preferences).
4. MP: put authoritative logic behind `isServer()`, mirror presentation via `sendServerCommand`.
5. Iterate with `-debug` + console.txt; commit working states.

## Verification
- console.txt + the debug menu are the oracles; for logic-heavy mods the lupa offline-harness approach in the
  CDDA note (run real server Lua against stubs for thousands of ticks) catches logic bugs a 10-minute in-game
  test won't.
- Not verified: every listed event's exact signature on 42.21 - trust the Umbrella stubs for args.

## Gotchas
1. **Handler runs twice / N times.** Cause: shared re-run on save reload + unguarded `Events.X.Add`. Fix:
   `if MyMod._registered then return end MyMod._registered = true`.
2. **`getCell()` nil at boot.** Cause: shared runs at menu boot. Fix: touch the world only inside `OnLoad`/
   later events.
3. **Context menu does work when `test`.** The engine calls your callback with `test=true` to ask "would you
   add options?" - never mutate state then.
4. **`sendClientCommand` args silently drop Java objects.** Only plain data crosses; send `getOnlineID()` or
   coordinates and re-resolve on the other side.
5. **ModData not syncing in MP.** By design; use `ModData.transmit/request` or commands. Global mod data on
   clients doesn't persist across reconnects - keep authoritative data server-side.
6. **Custom outfit never applied.** Cause: item GUID missing from `fileGuidTable.xml` (or duplicate GUIDs).
   Fix: unique GUID per clothingItem XML + outfit entry.
7. **TimedAction `perform` runs on the wrong side.** In B42 do authoritative effects on server; client-only
   actions (UI) are fine in `client/TimedActions`.
8. **`Events.OnZombieUpdate` too heavy.** It fires per zombie per frame; precompute, use throttled think
   windows (both studied mods stagger: 15-tick windows / per-second sweeps).
9. **`getText` returns the key.** Missing or misnamed translation JSON key; `Translate/EN/` file must be valid
   JSON (`"Base.X": "Name"`) and the folder named exactly `Translate`.
10. **Sandbox enum labels broken.** Use `numValues` + `valueTranslation` and translate
    `Sandbox_<ns>_<opt>_option1..n`; `%` in any label breaks the screen (String.format).

## Assets
n/a (behaviour-layer note; asset pipelines in the anatomy + CDDA + dressed-zombie notes).

## Open questions
- Whether `PZAPI.ModOptions` covers all widget types vanilla uses internally (some panels look hand-built).
- Exact event signature drift between 42.x builds - Umbrella regenerates per build; diff stubs when upgrading.
- Server-side-only contexts on dedicated servers (does `client/PZAPI` get loaded at all - probably not; keep
  ModOptions usage behind `isClient()`/client folder).
