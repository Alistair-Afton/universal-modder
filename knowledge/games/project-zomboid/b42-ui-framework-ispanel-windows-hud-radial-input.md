---
kind: game
title: 'The UI framework: ISUI widget library, custom windows and HUD, radial menus, 3D-model widgets, XUI skinning, input and controller'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); media/lua/client/ISUI census-verified (138 files)'
platform: windows
engine: java
route: loader-api
tools: ["ISUI Lua framework (138 files in the shipped install)", "ISXuiBuilder/ISXuiSkin", "keyBinding table", "OnMouse/OnKey events"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Lua_(API)"
tags: [b42, ui, isui, hud, panels, windows, radial-menu, xui, skin, input, controller, widgets, 3dmodel]
---
# The UI framework: ISUI, custom windows, HUD, radial menus, XUI, input (42.21.0)

> The presentation layer for everything else in this docset: `media/lua/client/ISUI` ships **138 files** -
> a real retained-mode widget library (not immediate-mode hacking), plus radial menus, a 3D-model-in-UI
> widget, a skin/layout system (XUI), and full input APIs. If your mod needs a face, it builds on these.

## Setup
- Everything under `media/lua/client/ISUI/` is readable vanilla Lua - the framework itself. Copy the
  smallest similar file, don't write widgets from scratch.

## Route and why
Loader API - ISUI is pure Lua, meant to be subclassed.

## 1. The widget census (what exists so you never build it)
**Core**: `ISUIElement` (base: drawText/drawTexture/drawRect/hover/focus/input), `ISPanel`,
`ISPanelJoypad`, `ISWindow`, `ISCollapsableWindow(+Joypad)`, `ISResizeWidget`, `ISRect`, `ISLine`,
`ISMouseDrag`.
**Dialogs**: `ISModalDialog`, `ISModalRichText`, `ISCollapsableModalRichText`, `ISAlarmClockDialog`,
`ISBombTimerDialog`, `ISDigitalCode`, `ISConfirmMonitorSettingsDialog`, `ISDuplicateKeybindDialog`,
`ISSetKeybindDialog`, `ISDemoPopup`, `ISServerDisconnectUI`, `ISPostDeathUI`.
**Controls**: `ISButton`, `ISResizeableButton`, `ISLabel`, `ISTextBox`/`ISTextEntryBox`,
`ISTickBox`, `ISRadioButtons`/`ISRadioOption`, `ISComboBox`, `ISSpinBox`, `ISScrollBar`,
`ISScrollingListBox`, `ISProgressBar`/`ISRadialProgressBar`/`ISGradientBar`/`ISLcdBar`,
`ISColorPicker`/`ISColorPickerHSB`, `ISTabPanel`, `ISOnScreenKeyboard`, `ISVolumeControl`/
`ISMegaVolumeControl`/`ISGameSoundVolumeControl`, `ISButtonPrompt`, `ISMouseDrag`.
**Rich/visual**: `ISRichTextPanel`/`ISRichTextBox`/`RichTextLayout` (formatted text, the lore/media UI
engine), `ISImage`, `ISUISprite`, `ISUITextureGetter`, **`ISUI3DModel`** - renders a live 3D model inside
a panel (character previews, item showcases).
**Radials**: `ISRadialMenu` + `ISEmoteRadialMenu`, `ISFirearmRadialMenu`, `ISLightSourceRadialMenu`,
`ISJoystickButtonRadialMenu`, `ISBackButtonWheel`, `ISDPadWheels`, `ISAtomUIJoypad`, `Gamepad/` folder.
**Skin/layout**: `ISXuiBuilder`, `ISXuiSkin`, `ISXuiTableLayout`, `ISLayoutManager`, `Style/`,
`Layout/` - the XUI system (script-side styling for `xui` script blocks from the census note).
**Feature UIs worth copying**: `ISInventoryPage`/`ISInventoryPane(+ContextMenu)`/`ISEquippedItem` (the
HUD+loot UI), `LootWindow`/`InventoryWindow` folders, `ISWorldObjectContextMenu`,
`ISMoveableContextMenu`/`ISMoveableInfoWindow`, `ISLiteratureUI`, `ISWriteJournal`, `ISRadioAndTvMenu`,
`ISTradingUI`+`ISTradingUIHistorial` (**a trading window exists!**), `ISDesignationZonePanel`,
`ISFluidContainerMenu` (entity-system UI), `ISGarmentUI`/`ISMakeUpUI`, `ISVehicleAnimalUI`,
`ISWarManagerUI`, `ISBBQInfoWindow`/`ISGeneratorInfoWindow`, `ZombiePopulationWindow`,
`ISExtendedPlacementUI`, `ISFishingUI`-family, `ISFitnessUI`, `ISForageIcon`.
**Framework plumbing**: `ISUIHandler`, `ISUIElementJoypad`, `ISMPPanel`-family, `PlayerData/`,
`PlayerStats/`, `Maps/` (world-map editor), `UserPanel/`, `AdminPanel/`, `InventoryWindow`, `FPS.lua`,
`SpeedControlsHandler`, `Interpolation*Debug`.

## 2. Building a custom window (the canonical recipe)
1. `MyWin = ISCollapsableWindow:derive("MyWin")` + `MyWin:new(x, y, w, h)`; `initialise()` calls
   `ISCollapsableWindow.initialise(self)`, `createChildren()` adds widgets with `self:addChild`.
2. Instantiate + `addToUIManager()` (the `getPlayerScreenUI`-style entry); `setVisible(false)` to hide;
   `close()` removes. Vanilla pattern: a `require "ISUI/..."` class + a tiny toggle on keypress.
3. Layout by hand (x/y/w/h) or `ISLayoutManager`/`ISXuiTableLayout`; DPI-scale via
   `getCore():getScreenWidth/Height` anchors.
4. Input: `OnMouseDown`/`OnMouseUp`/`OnMouseMove`/`OnMouseWheel` on the element, `isKeyDown`/
   `keyBinding` table (`{ name, key }` registered via `getCore()`/`GameKeyboard` patterns - see the
   character-hooks note), `Events.OnKeyPressed/OnKeyStartPressed` for global keys, `onRightMouseDown`
   in world for context.
5. Text/style: `drawText`/`drawTextCentre` + `UIFont` sizes; rich text through `RichTextLayout` markup
   (`<RGB:...>`, `<IMAGECENTRE>` tags used by literature UI); textures via `getTexture(name)` or
   `ISUITextureGetter` helpers.
6. 3D previews: `ISUI3DModel` (dress a model, animate it in-panel - that's how character creation works).
7. Controller: every widget has a `Joypad` sibling/`joypad` fields (`ISPanelJoypad`,
   `ISCollapsableWindowJoypad`, `Gamepad/`) - copy the vanilla controller pattern rather than skipping it.
8. Skins: `xui` script blocks + `ISXuiBuilder`/`ISXuiSkin` let you restyle entire widget families -
   "total conversion reskin" is a data file.

## 3. HUD and world-space UI
- HUD elements: `ISEquippedItem`-style anchored panels; add via `ISUIHandler`/`UIManager` on
  `OnGameStart`; draw each frame in `render`/`prerender` (retained widgets - you draw in `render()`).
- World-space: `getRenderer():draw*`/`ISCoordConversion` (character-hooks note) for floating labels/zone
  overlays; `ISWorldObjectContextMenu` for square right-clicks; `ISDesignationZonePanel` for drag-rect
  zone selection.
- Tooltips: `ISToolTip`/`ISToolTipInv` + `DoTooltip` (items/entities) - hovering works on world objects
  too via `getMouseOverSquare`-family patterns.

## 4. ModOptions (the supported settings UI)
`PZAPI.ModOptions` (covered in the runtime note): dropdowns/checkboxes/sliders/keybinds with
onApply hooks - use it instead of shipping a custom settings window; it's the community-standard face.

## Gotchas
1. **ISUI is client-only** - all of it lives in `client/`; on dedicated servers it's dead code (gate it).
2. **Widgets are retained-mode** - create once, toggle `setVisible`; re-creating per frame leaks and
   flickers.
3. **`ISUIElement` base has no auto-layout** - children use absolute coords inside the parent; use
   `ISXuiTableLayout`/`LayoutManager` for resizable UIs.
4. **Coordinates are pixels** - anchor to screen edges via `getCore():getScreen*` on resolution change.
5. **Joypad is a separate code path** - `Joypad`-suffixed classes exist because controller focus/
   navigation is explicit; test with a pad if you claim support.
6. **Z-order follows add order/UIManager** - modals use `ISModalDialog`/`alwaysOnTop`; dragging needs
   `ISMouseDrag` semantics, not manual position math.
7. **`ISUI3DModel` needs a valid model script** - it's the same pipeline as world models; garbage model
   = silent empty panel.
8. **Don't fight `ISContextMenu` positioning** - use `ISWorldObjectContextMenu`/`OnFillWorldObjectContextMenu`
   hooks for world interactions; raw `ISContextMenu` is for UI-space menus.
9. **XUI (`xui` script blocks + ISXui*)** is the supported restyle path - editing vanilla widget files
   is fragile and version-locked.
10. **`ISUIHandler`/`UIManager` calls** are the entry points - `addToUIManager` is how every vanilla
    panel appears; missing it = invisible window.

## Verification
- Census-verified on 42.21.0: all 138 ISUI files/classes above come from the shipped install listing.
- Verified in the wild: ModOptions is community-standard; `ISTradingUI`/`ISUI3DModel`/radials are live
  vanilla systems; `Maps/` editor documented in the interactive-world note.
- Not verified in-game: building a novel window end-to-end this session (the recipe mirrors vanilla
  class files), XUI skin authoring (the system exists; authoring untested).

## Open questions
- `ISXuiTableLayout`/`Style` grammar (xui script block schema beyond what the census note names).
- Whether `ISUI3DModel` supports AnimSets in-panel or only static/idle poses.
- `ISDesignationZonePanel` reusability for mod zone-selection UX.
