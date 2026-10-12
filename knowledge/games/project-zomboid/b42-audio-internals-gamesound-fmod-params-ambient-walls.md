---
kind: game
title: 'Audio internals: the real FMOD engine, GameSound clips/parameters/sustain points, ambient walls, sound limiters, voice, dynamic music scoring'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); javap/jar-verified (zombie.audio, 25 classes)'
platform: windows
engine: java
route: loader-api
tools: ["javap (JDK)", "sound script blocks", "FMOD parameters", "media/sound"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Sound_scripts"
tags: [b42, audio, sound, fmod, gamesound, clips, parameters, ambient-walls, limiter, voice, music, occlusion]
---
# Audio internals: FMOD, GameSound, parameters, ambient walls, limiters (42.21.0)

> PZ's audio isn't a wav-player - `zombie.audio` (25 classes) is a real FMOD layer: parameter-driven
> clips, sustain points, ambient "walls" of sound, per-group limiters, dynamic music intensity/threat
> scoring, and a voice system. This note maps the whole thing so "custom sounds" scales to "custom
> audio systems".

## Setup
- javap/jar-verified 42.21.0. `sound` script blocks (census note) declare the data; these classes run it.

## Route and why
Loader API - declare `sound`/`soundTimeline`/`voice` data; drive playback through the emitter/API layer.

## The audio map (javap/jar-verified)
- `GameSound`: `getName`/`getMasterName`/`getCategory`, `getRandomClip`, `numClipsUsingParameter`,
  `isLooped`, `getMaxDistanceOfClips`, `getUserVolume`/`setUserVolume`, `getEvent`-family - a named
  sound bank entry (one "sound" = N clips + parameters).
- `GameSoundClip`: `getFile`, `getEvent`/`getEventDescription` (FMOD event), `getVolume`/`getPitch`/
  `getEffectiveVolume(InMenu)`, `getMaxDistance`/`getMinDistance`/`hasMaxDistance`/`hasMinDistance`,
  **`hasParameter`/`hasSustainPoints`** (clips can be parameter-driven AND have loop sustain regions!),
  `isStopImmediate`, `checkReloaded` (hot-reload!).
- FMOD plumbing: `FMODParameter`/`FMODParameterId`/`FMODParameterList`/`FMODParameterUtils`,
  **`FMODGlobalParameter`/`FMODLocalParameter`** (engine-wide vs per-instance params - distance,
  occlusion, weather, danger), `FMODAmbientWalls`/`FMODAmbientWallLevelData` - the environmental wall
  system (forest/sea/city sound curtains around the player), `FMODGlobalParameter`-family.
- Emitters: `BaseSoundEmitter`/`DummySoundEmitter`, `BaseSoundBank`/`DummySoundBank` (headless dedi
  stubs), `ObjectAmbientEmitters` (emitters attached to world objects - machines, water, fires),
  `TreeSoundManager` (tree rustle layer).
- Limiters: **`SoundInstanceLimiter`/`SoundLimiterParams`** + `getSoundLimiterGroupID`/
  `registerWithSoundLimiter` on items (cheat-sheet) - the cap-on-concurrent-clips system (why the CDDA
  ants needed a manual limiter for custom SFX).
- Music: `MusicIntensityConfig`/`MusicIntensityEvent(s)`, `MusicThreatConfig`/`MusicThreatStatus(es)` -
  the drama-scoring engine (MusicChoices from the presentation note feeds these).
- Combat-specific: `LoopedRangedWeaponSounds` (the gun loop tail).

## The layers, and what a mod adds where
1. **Data**: `sound` block (`media/scripts`) -> a `GameSound` with clips, category, 3D falloff,
   `looped`, `is3D`, parameters. `soundTimeline` blocks layer timed clips for a sequence (engine
   cranking, footsteps over an action). `voice` script blocks -> the `playerVoiceSound` speech pool.
2. **Playback**: `getSoundManager():PlaySound(name, ...)`/`PlayWorldSound`-family (vanilla Lua calls)
   or position-attached emitters; `ObjectAmbientEmitters` puts a looped sound on an object.
3. **Parameter-driven**: clips with FMOD parameters react to runtime values (`FMODLocalParameter` on
   the emitter, `FMODGlobalParameter` engine-wide) - a motor that pitches with RPM is ONE clip + a
   parameter, not N clips.
4. **Occlusion/ambient**: ambient walls + room-tone registration (`registerRoomTone`, cheat-sheet)
   handle interior/exterior mix automatically; don't bake reverb - let the walls do it.
5. **Voice**: `playerVoiceSound`/`transmitPlayerVoiceSound`/`getVoiceSoundName`/`getVoiceChoice` -
   the speech system (player hurt/talk barks); NPC speech = positioned `PlayWorldSound` + your own
   clips (no third-party speech API - see beyond-chassis note).
6. **Music**: `MusicChoices` entries + `MusicIntensityEvent`s/`MusicThreatStatus` reads - dynamic
   scoring keys off the same drama stats (`getNumChasingZombies` family).
7. **Limiting**: `registerWithSoundLimiter` + a `SoundInstanceLimiter` group on spammy sources
   (creature loops, machines) - the engine's own cap mechanism.

## Gotchas
1. **A "sound" is a bank, not a file** - `getRandomClip` means N variants per GameSound; author
   variants or your creature repeats audibly.
2. **Sustain points exist** (`hasSustainPoints`) - engine loops mark in/out regions in ONE file;
   don't slice loops into clips, use the sustain metadata.
3. **Hot-reload exists** (`checkReloaded`) - sound banks reload in debug without restart; use it.
4. **`isStopImmediate` vs sustain-out** - abrupt stops pop; let clips finish their sustain region.
5. **Global vs local parameters** - wrong scope = every emitter sharing one value (all motors pitch
   together); local params are per-emitter.
6. **Volume is layered** - clip `getVolume` x `getUserVolume` x category x distance; "too quiet" bugs
   are usually a category or master-volume layer, not the file.
7. **WorldSound vs audio** - `WorldSoundManager` (weapons note) is the *zombie-hearing* layer;
   `zombie.audio` is what humans hear - they're separate systems; a loud sound needs both if it
   should attract.
8. **Emitter lifetime** - `ObjectAmbientEmitters`/looped emitters need explicit stop/remove or they
   orphan on object unload (sounds that keep playing from nothing).
9. **`DummySoundBank/Emitter` on dedi** - audio is a null impl on servers; server code must not gate
   gameplay on "did the sound play".

## Verification
- javap/jar-verified 42.21.0: the `zombie.audio` census and `GameSound`/`GameSoundClip` method names
  above are real dumps.
- Verified in the wild: `sound`/`soundTimeline`/`voice` script blocks load (vanilla), `getRandomClip`
  variant pools (vanilla voice/footstep variation), sound limiters (the CDDA ants hit the cap and
  hand-rolled one - now we know the engine's own exists), `PlayWorldSound`-family calls.
- Not verified in-game: authoring parameter-driven clips end-to-end (the parameter field names in the
  `sound` block schema are the spec to copy), ambient-wall registration for custom zones.

## Open questions
- The exact `sound` block fields that map to `hasSustainPoints`/`hasParameter` (the vanilla file is
  the schema - `media/scripts/sounds*`).
- `FMODAmbientWallLevelData` semantics (per-level walls? height layers?).
- Whether custom `FMODGlobalParameter`s can be registered from script or only consumed.
