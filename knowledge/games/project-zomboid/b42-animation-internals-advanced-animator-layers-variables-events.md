---
kind: game
title: 'Animation internals: AdvancedAnimator, layers/sub-layers, typed animation variables, AnimEvents, AnimNode grammar, grapples, population visuals'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); javap/jar-verified (113 advancedanimation + 49 animation + 98 model classes)'
platform: windows
engine: java
route: loader-api
tools: ["javap (JDK)", "getAdvancedAnimator() on any IsoGameCharacter", "AnimSets/animscript XML", "zombie.core.skinnedmodel.*"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Animation_guide"
tags: [b42, animation, animator, animset, animnode, animlayer, variables, animevents, grapple, skeleton, transitions]
---
# Animation internals: the AdvancedAnimator stack (42.21.0)

> Beneath "AnimSets + variables" sits a real runtime: `zombie.core.skinnedmodel.advancedanimation` (113
> classes) implements layered states, typed variable slots, transitions and in-clip events; `model`
> (98) and `animation` (49) do skinning/loading; `population` (26) renders crowds. This note maps the
> whole stack and the exact seams a mod can pull on - including bone-space and anim-driven callbacks
> that most modders never find.

## Setup
- javap/jar-verified 42.21.0. The animator is `char:getAdvancedAnimator()` on any IsoGameCharacter.
- Builds on the presentation note (AnimSet/animscript XML anatomy) and both creature notes (the
  variable-driven AnimSet merge pattern).

## Route and why
Loader API for composition (AnimSets/variables); internals knowledge for advanced control.

## The stack, top to bottom (jar-verified)
- `AdvancedAnimator`: `setState(name)`, `getCurrentStateName`, `containsState`, `setAnimSet`,
  `getRootLayer`/`getSubLayerAt`/`getSubLayerCount` (`AnimLayer`/`SubLayerSlot` - **layered
  animations are real**: upper-body layer over locomotion), `ProcessTransitions`,
  `FindTransitionsFromProxy`, `addAnimCallback`, `OnAnimEvent`/`invokeGlobalAnimEvent`,
  `debugGetVariables`, `updateSpeedScale`, `getCharacter`, `isRecording`, `getDebugMonitor`,
  `searchFolders`, `Reload`, `containsAnyIdleNodes`, `systemInit`.
- Graph: `AnimState`, `AnimNode`, `LiveAnimNode`, `AnimNodeAsset(Manager)`, `AnimTransition`,
  `AnimCondition`, `AnimationSet`, `SubLayerSlot`, `AnimLayer`.
- Variables: `AnimationVariableHandle(s)`/`AnimationVariableHandlePool`, `AnimationVariableSlot*` -
  typed slots: **Bool, Int, Float, String, Enum, and Callback variants**
  (`AnimationVariableSlotCallback{Bool,Enum,Float,Int,String}`) - variables can carry a Lua callback
  the graph invokes. `IAnimationVariable(Registry|Map|Source|SourceContainer)`,
  `AnimationVariableWhileAliveFlag*`, `AnimationVariableEnumParser`, `AnimationVariableReference`.
- Events: `AnimEvent`, `AnimEventSetVariable`, `AnimEventFlagWhileAlive` - clips fire named events
  mid-play (footsteps, hit frames, grapple latches); `addAnimCallback`/`OnAnimEvent` are the
  animator-side listeners.
- Grapples: `BaseGrappleable`, `IGrappleable(Wrapper)`, `GrappleOffsetBehaviour` - the bite/wrestle
  attach-offsets system that makes zombies latch onto bodies correctly (why CoF monsters snap to prey).
- Presentation: `model/` (98 classes incl. `ModelInstance` - `dismember`, attachments, `setItemVisual`
  from the cheat sheet), `ModelCamera(RenderData)`, `CharacterTextures`, `DeadBodyAtlas` (corpse
  rendering!), `shader/` (3 classes - character shaders), `visual/`, `runtime/`, `Matrix`, `Vector`,
  `HelperFunctions`.
- `population/` (26 classes): the visual crowd system - distant zombies are skinned instances rendered
  cheaply through this, which is why hordes don't melt the GPU.

## The Lua-facing seams (what you actually call)
1. `char:setVariable(name, value)`/`getVariable`/`clearVariable` - writes into the variable slots that
   AnimConditions read (the whole COF gating pattern). String slots select clips; bools gate states;
   floats drive blends.
2. `char:getAdvancedAnimator()` then `setState`/`getCurrentStateName`/`debugGetVariables` - force-state
   and introspect (debug animator window does exactly this).
3. `AnimEvent`s authored in animNode XML (`<m_EventName>`-family fields) - timed callback points inside
   clips; listeners via `addAnimCallback` (Lua callback slots where exposed) or polling `OnAnimEvent`-
   derived state in your tick code. Footsteps/bite-frames are AnimEvents, not frame counters.
4. `reportEvent("wasBumped")`-style semantic events (IsoGameCharacter) - the input side the graph
   consumes.
5. Layers: `getSubLayerCount`/`getSubLayerAt` reveal the layer structure; mods that want
   upper-body-only actions play on a sub-layer (aim while walking is a sub-layer), not a second
   animator.
6. `updateSpeedScale` - playback-rate scaling (slow-motion monsters, haste).

## Authoring grammar (the XML side, consolidated)
- `AnimSet` XML = `<m_Name>` + `<m_Animations>`/animNode entries; animNode = clip ref + conditions
  (`<m_Conditions><m_String m_Name="Var" m_Value="..."/>` bool/string/float checks) + events +
  `BlendTime`; states = folders (`animsets/<set>/<state>/`) scanned by `searchFolders`.
- `animscript/` combat.xml = the second, condition-driven language (AnimStates with PlayAnim/Out/BlendTime
  - see presentation note); it compiles to the same graph.
- `animationsMesh` script blocks bind model+animset+prefix; `anims_X/<dir>/<prefix>_*.x|glb|fbx` clips;
  humanoid `Bob`/`Kate` prefixes; `animNodeFilter` per-state prefixes.

## Gotchas
1. **Variables are typed slots, not strings** - `setVariable("x", true)` vs `"true"` land in different
   slots; conditions checking string "true" won't see a bool true (silent no-trigger, the classic bug).
2. **`setState` fights the AI states** - forcing animator states while the StateMachine pushes its own
   causes flicker; prefer driving variables and letting transitions flow (the AI-internals note).
3. **AnimEvents need listener wiring** - authoring an event in XML without a callback/variable
   consumption does nothing; check `debugGetVariables` in the animator window to see live values.
4. **`getRootLayer` vs sub-layers** - full-body states live on the root; playing on a sub-layer that
   the game expects empty (or vice versa) breaks blends; inspect first.
5. **`DeadBodyAtlas` is corpse rendering** - custom corpses need to play along (the CoF corpse-variants
   pattern, not a separate system).
6. **`Reload`/`checkModifiedFiles` exist** - animator assets hot-reload in debug builds; use it instead
   of save reloads when tuning AnimSets.
7. **`ModelCamera` isn't the game camera** - it's the model-preview camera (UI/3D model widgets).
8. **Layer count is fixed per character type** - you compose within existing layers; new layer types
   aren't Lua-registered.
9. **Speed scale is animation-local** - `updateSpeedScale` moves the *anim*, not the character's
   locomotion (pair with `setSpeedTypeFromWalkType`/walk-type vars or the feet slide).

## Verification
- javap/jar-verified 42.21.0: all class/method names above (AdvancedAnimator method list is a real dump).
- Verified in the wild: variable-driven AnimSet gating (COF), AnimEvent-driven footsteps (vanilla
  `DoFootstepSound` is wired to anim events), sub-layer aim/walk composition (vanilla), `Reload` used
  by the debug animator.
- Not verified in-game: Lua-side `addAnimCallback` on variable callback slots (the slot types exist;
  the binding path needs a focused test), `searchFolders` mod-folder scanning rules.

## Open questions
- Exact animNode XML field list per node type (the vanilla files ARE the reference; a field-by-field
  grammar doc is a future decompile task on `AnimNode`).
- Callback-variable Lua binding signature (`AnimationVariableSlotCallback*` - how Kahlua registers).
- `population/` crowd rendering: whether mods can push into it or it's population-manager-internal.
