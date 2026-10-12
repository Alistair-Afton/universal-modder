---
kind: game
title: 'The modder toolchain: launch flags, the 32-window in-game debug suite, DebugOptions/LineDrawer, Lua error anatomy, fast iteration and scripted in-game testing'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); jar-verified debug package census'
platform: windows
engine: java
route: loader-api
tools: ["-debug launch flag", "zombie.debug (32 debug windows, 48 options)", "console.txt", "um win automation", "EmmyLua (Umbrella)"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Debug_mode"
- "https://github.com/PZ-Umbrella/Umbrella"
tags: [b42, tooling, debug, iteration, console, lualog, line-drawer, debug-windows, testing, automation, profiling]
---
# The modder toolchain: debug suite, error reading, fast iteration, scripted testing (42.21.0)

> Shipping the mod is 20% code and 80% iterate-verify. This note is the complete toolbelt: the engine's
> own debug suite (32 debug windows, 48 options, an overlay `LineDrawer`, CSV exporters), how to read a
> Lua crash in `console.txt`, how to iterate without restarting, and how to automate in-game
> verification end-to-end.

## Setup
- Steam launch options or a shortcut: `ProjectZomboid64.exe -debug` (plus flags below). Logs land in
  `C:\Users\<you>\Zomboid\console.txt` (truncated each run - copy before relaunching).

## Route and why
Stock tooling - no external debugger needed for 95% of work.

## 1. Launch flags that matter
`-debug` (the master switch - unlocks everything below), `-debuglog=<types>` (filter DebugLog channels:
`Sound`, `Animation`, `Network`, `Script`, `Lua`... - `zombie.debug.DebugType` enum names the channels),
`-cachedir=<path>` (point `Zomboid/` at a scratch profile so test saves/options don't touch your real
one), `-nosteam` (skip Steam init where supported), `-x` `-y` `-width` `-height` (window geometry for
screenshot automation), `-safemode`-style recovery flags exist for corrupt setups. `options.ini` in the
profile controls windowed/fullscreen - scripted runs want `fullscreen=false`.

## 2. `zombie.debug` - the built-in dev suite (jar-verified)
- **32 `debugWindows` classes**: the F11/`Debug` menu suite - animator/AnimSet inspector, world objects,
  chunk viewer, population manager view, pathing visualizers, mod-data inspector, network stats,
  climate, erosion, stash map... The animator window is the single most valuable monster-dev tool:
  live AnimSet state + variables per character.
- **48 `options` classes**: `DebugOptions`/`DebugOptionsXml`/`BooleanDebugOption` - toggles the suite
  reads (cheats like invisible-zombies, fast-forward, teleport are options, not separate hacks).
  `DebugContext`, `DebugType`, `LogSeverity`.
- **`DebugLog`/`DebugLogStream`/`DebugLogCfgFile`/`IDebugLogFormatter`/`StackTraceContainer`** - the
  logging engine itself, channel-filtered by `-debuglog`.
- **`LineDrawer`** (2 classes): debug line/box drawing in world space - draw aggro radii, path targets,
  zones during dev.
- **`DebugCSVExport`** + `DebugCSVExportFirearms`/`DebugCSVExportFluidContainers`/
  `DebugCSVExportMoveableTiles`: vanilla dumps live data to CSV - the pattern for "export my mod's state"
  tooling.
- **`BaseDebugWindow`**: the window class all 32 extend - community mods ship their own debug windows
  built on the same ISUI pattern.

## 3. `console.txt` and reading a Lua crash
- Format: `function : <name> -- file: <path>` frames + the Kahlua error line. Errors name the *mod file*
  (workshop mods appear as their staged path) - your first filter is your mod name.
- Common signatures: `attempted index of nil` (nil field - the `.field` vs `:method()` trap),
  `number expected, got table` (Java wants a primitive), `attempted comparison` (nil in comparison),
  `se.krka.kahlua...` frames = Java-side boundary errors.
- `print()`/`DebugLog.log` writes go there too; prefix lines with your mod tag so grepping a 40k-line
  file is instant.
- A Lua error kills that Lua call, not the game - a dying `OnZombieUpdate` means silent creature
  failures; always check the log after a test, not just the screen.

## 4. Iteration speed
- Scripts (`media/scripts`), textures, sounds, maps reload on world load - no game restart needed.
- **Lua requires a reload of the save** (reload game, not process - exit to menu, reload save, ~15s vs
  ~90s cold start).
- Keep a dedicated test save per feature: flat sandbox preset (no zombies unless testing AI, high loot,
  god mode for safety), right-click debug spawns for iteration.
- `ModOptions` (PZAPI) instead of editing Lua for tunables - sliders beat restart loops.

## 5. Scripted in-game verification
The `um win` + game-automation loop, PZ-flavoured:
- Launch: `ProjectZomboid64.exe -debug -cachedir=<scratch> -width 1280 -height 720` windowed.
- Drive: keyboard/mouse injection to menu -> load the test save -> run your scenario (Lua test harness
  file can self-drive: an `OnTick` that steps through spawn/check/assert and writes results to
  `console.txt`).
- Observe: `um win screenshot` at checkpoints; parse `console.txt` for your `[TEST] PASS/FAIL` lines -
  that IS your CI.
- Kill by exact PID only (`um win kill <pid>`); crash-reporter windows may appear after kill - close by
  title.
- The `game-automation` skill covers the mechanics; the PZ-specific bits are: fixed window geometry via
  launch flags, menu automation is Enter-heavy, and `console.txt` is the ground truth.

## 6. Profiling and perf hygiene
- `-debug` frame graphs + `OnTick`/`OnZombieUpdate` cost is real - stagger thinks (the creature notes'
  15-tick windows), snapshot positions on an 8-tile grid, weak-keyed caches, cap concurrent sounds.
- `pcall` instrumentation: wrap your think functions, log elapsed on exceed - cheap profiler for Lua.
- `getTimestamp`-family timers exist; `console.txt` timestamps help trace ordering bugs.

## 7. External tooling that fits this workflow
- **Umbrella EmmyLua stubs** in VS Code/IDEA: typed signatures + inline docs for every call above; the
  closest thing to a debugger surface pre-runtime.
- **Decompilers for behaviour questions**: ZomboidDecompiler (Vineflower+Rosetta) - read `zombie.ai`
  states or `popman` when the javap surface isn't enough; describe findings in your note, never commit
  decompiled code.
- **javap** for "does it exist" (the cheat-sheet workflow); the jar's own `zombie.debug` classes for
  "what does the suite already show me".

## Gotchas
1. **`-debug` changes gameplay** - zombie behaviour options alter AI timing; verify final behaviour on a
   non-debug run before calling it done.
2. **`console.txt` truncates each launch** - copy it before relaunching; `Zomboid/Logs/` keeps some
   rolled logs but not everything.
3. **`debuglog` filters hide your prints** - `print` goes to the `General` channel; over-filtering hides it.
4. **Reload only reloads data files on world load** - changed `mod.info`/workshop metadata needs full exit.
5. **Two test saves with the same name desync** - use `cachedir` scratch profiles for automation runs.
6. **`LineDrawer`/`DebugCSVExport` are debug-build affordances** - don't gate shipping features on them;
   they're dev tooling, not runtime APIs.
7. **Lua errors are per-call fatal** - an erroring tick handler may silently stop firing; check the log
   after every test, not just when something looks wrong.
8. **Umbrella stubs lag the build** - 42.21.0 methods exist that stubs don't know yet; javap is fresher.

## Verification
- Verified on 42.21.0: `zombie.debug` census (32 windows/48 options/LineDrawer/CSV exports),
  `console.txt` format, `-debug`/`-cachedir` flags, menu/save reload loop, `um win` screenshot+kill.
- Not verified: `-debuglog` channel names beyond the common set, remote Lua debugger attach (community
  reports MobDebug-style attach; not tested here), `-nosteam` reliability on this build.

## Open questions
- Exact `DebugType` channel list for `-debuglog=` filtering on 42.21 (enum exists; names need listing).
- Whether `BaseDebugWindow` is practically extensible from mod Lua for custom debug windows.
- Remote debugger attach points (Kahlua debug hooks) - worth one focused session.
