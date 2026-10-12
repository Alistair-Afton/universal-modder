---
kind: game
title: 'Weather and climate internals: ClimateManager air-mass simulation, networked ThunderStorm, seasons, fog/night colors, per-square temperatures'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); javap/jar-verified (58 classes in zombie.iso.weather)'
platform: windows
engine: java
route: loader-api
tools: ["javap (JDK)", "getClimateManager() entry point", "SandboxVars climate fields", "media/lua/shared|client weather tables"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Weather"
tags: [b42, weather, climate, airmass, thunder, storm, seasons, temperature, fog, moon, simulation]
---
# Weather and climate internals: ClimateManager, air-mass sim, ThunderStorm, seasons (42.21.0)

> `zombie.iso.weather` is a 58-class weather *simulation*, not a lookup: ClimateManager runs an
> air-mass/front model with history + forecasting, temperature is per-square/per-character, thunder is
> a networked event system, and fog/night/moon colors are real climate values. Weather mods ride a
> deep API, not a "setWeather(rain)" bool.

## Setup
- javap/jar-verified 42.21.0. Entry point: `getClimateManager()` (returns the `ClimateManager`).

## Route and why
Loader API - the climate sim is Lua-readable/writable through `ClimateManager` + `IsoWorld` weather verbs.

## The system map (javap/jar-verified)
- `ClimateManager` (~all queries + sim): `getAirMass`/`getAirMassDaily`/`getAirMassTemperature`,
  `CalculateWeatherFrontStrength`, `getBaseTemperature`, `getAirTemperatureForCharacter`,
  `getAirTemperatureForSquare` (**temperature is spatial - different squares differ**),
  `getCloudIntensity`, `getClimateColor`/`getClimateBool`/`getClimateFloat` (typed climate channels),
  `getColFog`/`getColFogLegacy`/`getColFogNew`, `getColNight`/`getColNightMoon`, `getAmbient`,
  `getClimateForecaster`, `getClimateHistory`, `getClimateValuesCopy`, `CopyClimateValues`,
  `execute_Simulation`, `forceDayInfoUpdate`, `ToKph`/`ToMph`, `clamp*`/`clerp`.
- `ClimateValues`: the channel/value container (what `getClimateFloat` reads - temperature, humidity,
  wind, cloud, precipitation, fog strengths).
- `ClimateForecaster` + `ClimateHistory`: forecast + recorded curves (radio weather reports read this -
  the vanilla weather channel is data-driven off the same sim).
- `ClimateMoon`, `ClimateColorInfo`, `WeatherPeriod`, `WeatherShader`, `WorldFlares`, `SimplexNoise`,
  `Temperature` (unit conversions + history).
- `ThunderStorm` (javap-verified): `startThunderCloud`, `stopCloud`/`stopAllClouds`, `getClouds`,
  `triggerThunderEvent`/`enqueueThunderEvent`, `readNetThunderEvent`/`writeNetThunderEvent`
  (**thunder is MP-synced**), `applyLightningForPlayer`, `isModifyingNight`, `HasActiveThunderClouds`,
  `save`/`load`, `getMapDiagonal`, `noise`.
- `IsoWorld` bridges (cheat-sheet): `getWeather`/`setWeather`/`transmitWeather`.

## The mod-facing seams
1. **Read the sim**: `getClimateManager():getClimateFloat/ Bool(name)` for any channel; per-square
   temperature for survival logic (`getAirTemperatureForSquare`) instead of inventing your own.
2. **Drive the sim**: `setWeather`/`CopyClimateValues`/`forceDayInfoUpdate` + `execute_Simulation` -
   a scripted cold snap/hurricane is a `ClimateValues` write + sim step, not hacked flags.
3. **Thunder/lightning**: `triggerThunderEvent`-family + cloud management; lightning hits
   `applyLightningForPlayer` (the flash+deafen path - hook it for custom storms).
4. **Seasonal visuals**: `getClimateColor`/`getColFog*`/`getColNight*`/`getClimateMoon` - the
   fog/night/moon tint channels drive the renderer; a "different planet" mod reskins the *climate*
   values, not every sprite.
5. **Sandbox**: the climate sandbox options (rain/temperature curves) live in `SandboxVars` - a
   per-save weather profile is data.
6. **Body coupling**: Thermoregulator (body-sim note) reads `getExternalAirTemperature`/wind - weather
   mods automatically affect body temperature; no extra wiring needed.

## Gotchas
1. **`getAirTemperatureForSquare` != ambient** - interiors/shade/sun differ per square; use the
   square call for anything physical.
2. **`execute_Simulation` ticks the air-mass model** - write `ClimateValues` then simulate; setting
   floats alone gets damped back by the sim.
3. **Thunder is networked** - server writes events (`writeNetThunderEvent`), clients read; call the
   API on the server or it won't sync.
4. **`transmitWeather` is the MP broadcast verb** - server-side weather writes need it or clients
   see stale skies.
5. **Climate channels are typed** - `getClimateFloat` vs `getClimateBool` on the wrong channel type
   returns defaults; the channels come from `ClimateValues`/`climate.txt`-style data.
6. **`getClimateForecaster` predicts ahead** - don't compute tomorrow yourself; the sim already did
   (the radio channel reads the same thing).
7. **`SimplexNoise` drives fronts** - the "weather pattern" is noise + air masses; forcing flat values
   fights the natural drift - prefer biasing (multiplier channels) over pinning.
8. **`isModifyingNight`** - storms darken the night through a separate channel; night-vision/torch mods
   should read actual darkness, not "is it storming".

## Verification
- javap/jar-verified 42.21.0: `zombie.iso.weather` 58-class census; `ClimateManager`, `ThunderStorm`,
  `GameSound*` method lists above are real dumps.
- Verified in the wild: weather radio reads the forecaster; sandbox climate options exist;
  `transmitWeather` is the sync verb (cheat-sheet); thermoregulator couples to air temperature.
- Not verified in-game: writing `ClimateValues` end-to-end for a custom weather event, `WeatherShader`
  parameters, `WorldFlares` mod reachability.

## Open questions
- The full `ClimateValues` channel name list (each `getClimateFloat` key - enumerable via the debug
  weather window).
- `WeatherPeriod`/`WeatherShader`/`WorldFlares` mod-facing semantics.
- Whether `ClimateForecaster` is seeded per-save (deterministic weather for scripted scenarios).
