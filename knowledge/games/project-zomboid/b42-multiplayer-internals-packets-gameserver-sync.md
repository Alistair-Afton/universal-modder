---
kind: game
title: 'Multiplayer internals: the 296-class packet layer, GameClient/GameServer, ownership, co-op hosting and MP-safe mod patterns'
game: Project Zomboid
games_also: []
game_version: 'Build 42.21.0 (Steam, Windows 10); javap/jar-verified'
platform: windows
engine: java
route: loader-api
tools: ["javap (JDK)", "zombie.network.* (296 packet classes)", "ServerCommands.lua", "dedicated server (Steam app 380870)"]
anti_cheat: 'none'
status: working
agents:
- Devin (SWE-2 Medium)
humans: [Alistair-Afton]
date: '2026-10-12'
links:
- "https://pzwiki.net/wiki/Networking"
tags: [b42, multiplayer, server, packets, sync, ownership, dedicated-server, co-op, networking, luanet]
---
# Multiplayer internals: packets, GameServer, ownership and MP-safe mod patterns (42.21.0)

> Since 42.13 the server is authoritative; every "it works solo but not MP" bug is a sync-model bug.
> This note maps the real `zombie.network` layer - 296 packet classes, `GameClient`/`GameServer`,
> `CoopMaster`/`CoopSlave`, `FakeClientManager` - and gives the rules and recipes that make custom
> creatures/systems survive a dedicated server.

## Setup
- javap-verified 42.21.0. Builds on `b42-lua-runtime-...md` (command model) and the cheat-sheet note
  (transmit*/sync* verbs).

## Route and why
Loader API - the MP layer is Java-internal but Lua-reachable through the documented verbs.

## The network layer map (`zombie.network`, jar-verified)
- **`GameServer` (143 KB) / `GameClient` (97 KB)** - the two endpoints; almost all MP verbs route through
  them. `ServerOptions`/`ServerSettings`, `SteamWorkshopItems`/`GameServerWorkshopItems` (workshop sync),
  `DevGameServer`.
- **`zombie.network.packets` - 296 classes**, one per action: `AddExplosiveTrapPacket`,
  `AddItemToMapPacket`, `AddObjectToMapPacket`, `AddInventoryItemToContainerPacket`,
  `AddItemInInventoryPacket`, `AddBrokenGlassPacket`, `AddTrackPacket`, `AddXPMultiplierPacket`,
  `AddUserlogPacket`, `AddTicketPacket`, `AddWarningPointPacket`, `BanUnbanUserActionPacket`... the packet
  name IS the feature. New vanilla actions = new packets; your mod doesn't create packets - it rides the
  command layer or reuses the right packet by using the right API call.
- **Co-op hosting**: `CoopMaster`/`CoopSlave`, `CoopUserName`, `ICoopServerMessageListener` - "host from
  the client" spins an in-process server. `ConnectionManager`, `ClientChunkRequest`, `ChunkChecksum`,
  `ClientServerMap` (chunk sync + integrity), `ConnectionDetails`, `IConnection`.
- **`FakeClientManager`** - fake clients for server-side testing/bots (javap-verified to exist).
- **`BodyDamageSync` + `BodyDamageUpdatePacket`** - the damage-sync pipeline (`IsoPlayer:
  startReceivingBodyDamageUpdates` from the cheat sheet is the client entry).
- **Admin/moderation**: `BanSystem`, `DBTicket`/`DBBannedIP`/`DBBannedSteamID`, `DBResult`,
  `DiscordBot`/`DiscordSender` (server→Discord exists!), `Account`, `DesktopBrowser`.

## The sync model - the rules that decide if your mod works on a server
1. **Server owns the world.** State mutations happen on `server/` Lua (or admin-side); presentation lives
   on `client/`. A client that mutates world state changes nothing for other players.
2. **Commands are the mod-level wire**: `sendServerCommand(module, command, args)` (client→server),
   `sendClientCommand(player, module, command, args)` (server→client) + `Events.OnServerCommand`/
   `OnClientCommand`. This is the packet interface mods get - see `ServerCommands.lua` for the vanilla
   pattern.
3. **`transmit*`/`sync*` verbs are the engine-level wire** for stock object changes:
   `sq:transmitAddObjectToSquare`/`transmitRemoveItemFromSquare`/`transmitModdata`,
   `obj:sync`/`syncIsoObject(Send|Receive)`/`syncFluidContainer*`, `transmitPlayerVoiceSound`,
   `transmitWeather`. Mutate on the server WITH these, not with the silent setters.
4. **Ownership**: `getOwnerPlayer`/`isRemoteZombie`/`getOnlineID`/`getOwner` decide who drives an
   object - remote zombies/players are read-only views; drive them through the owner or the server.
   (`IsoZombie` remote methods verified in the cheat sheet.)
5. **ModData is sync-capable but not auto**: item/object `getModData` persists; pushing changes to
   clients needs `transmitModdata` (squares) or your own command broadcast.
6. **Chunk scoping**: clients only know loaded chunks (`ClientChunkRequest`/`ChunkChecksum` verify them);
   global state belongs in metazones/`SGlobalObjectSystem`, not square caches.

## Creature mods on a server - the checklist
- Spawn on `server/` (the `createRealZombieNow`/`addAnimal` calls), never client - duplicates otherwise.
- Brains on `server/` where they mutate (damage, aggro, spawns); client-side brains for presentation only
  (sounds, particles, anim variables the animset reads locally).
- Identity must survive sync: persistent outfit ID (zombies) / save-load (animals); a client-only identity
  table drifts across chunk unload (the COF recycling saga was exactly this).
- Sounds that attract zombies: `WorldSoundManager:addSound` on the server.
- Remote presentation: the COF `COF_RemotePresentation` pattern - server broadcasts "this zombie is
  monster X", clients set the `Is*` variables + visuals. Don't try to push Lua state per-tick; push
  identity once and let each client present locally.

## Dedicated server realities
- Separate Steam app (380870); mods load from `servertest.ini` `Mods=`/`WorkshopItems=` lists, synced to
  clients via `GameServerWorkshopItems`. Client-only Lua never runs there - `client/` files are dead code
  on a dedi; `shared/`+`server/` are what matter.
- The existing KB note `b41-dedicated-server-...` covers B41; B42 differences: server-authoritative
  everything (41 let clients mutate more), animal/entity systems server-side, workshop item sync
  (`SteamWorkshopItems` class exists).
- `FakeClientManager` = bots/fake clients on the server for load testing - exists; Lua reachability
  unverified.
- `DiscordBot`/`DiscordSender` are in the jar - server→Discord announcements are a vanilla feature mods
  can piggyback (config surface unverified).

## Testing MP without two PCs
- Host co-op from the game client (`CoopMaster`) + a second local client on the same machine (second
  Steam instance or the dedicated server binary); `console.txt` on each shows both sides.
- `-debug` + the debug network menus; admin commands via `zombie/commands` (`ServerCommands.lua`).
- `ChunkChecksum` catches client-server map divergence - a mod that writes different state on each side
  triggers checksum failures/kicks; that's your canary for sync bugs.

## Gotchas
1. **`server/` files don't exist on clients and `client/` files don't exist on dedi** - put shared tables
   in `shared/` or you'll get "module not found" only on servers.
2. **`sendServerCommand` args must be table-serializable** - no functions/userdata; keep args flat
   primitives or nested plain tables.
3. **Remote objects are views** - `isRemoteZombie()` true means "another client owns this"; mutating it
   locally ghosts the state.
4. **Packets aren't moddable** - you can't subclass `IZomboidPacket` from Lua; new wire messages = the
   command layer.
5. **`transmitModdata` is square-scoped** - object-level ModData needs `obj:sync` or manual commands.
6. **Anti-cheat isn't a thing here** - but `ChunkChecksum` divergence + `AddWarningPointPacket` exist;
   servers CAN flag/kick clients whose world disagrees - sloppy client-side mutation looks like cheating.
7. **`client/` Lua on dedi is dead code, not an error** - silent no-ops are the worst MP bug source;
   log once from `shared/` with `isServer()`/`isClient()` so you know where each file ran.
8. **Dedi has no renderer** - any code touching `getRenderer`/UI must be `client/`-gated or it NPEs the
   server.

## Verification
- javap/jar-verified 42.21.0: the classes above exist; packet count = 296 `.class` files under
  `zombie/network/packets`.
- Verified in the wild: server-authoritative model (42.13+), COF's presentation-sync pattern, command
  layer used by vanilla `ServerCommands.lua`.
- Not verified in-game: `FakeClientManager` Lua reachability, `DiscordBot` config surface, dedi workshop
  sync edge cases (no dedi instance was launched in this research).

## Open questions
- Full `LuaNetManager`/command-layer internals (names exist in vanilla Lua; the Java side is `GameClient`
  internals).
- Whether mods can register custom `IZomboidPacket`s via any supported route (almost certainly not from
  Lua - confirm via decompile if ever needed).
- B42 dedi startup flags for headless testing (the B41 note covers B41; B42 dedi differences not yet
  enumerated).
