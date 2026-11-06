# Vanilla patterns

How the vanilla DayZ 1.30 scripts do common jobs. Copy the shape and keep the vanilla signature exactly, since `override`
needs it. Each item names the vanilla file (`scripts/...` in `scripts.pbo`, also in Bohemia's DayZ-Script-Diff repository).
Boot anything that matters and read the log before relying on it.

## Which side am I on

`GetGame().IsServer()`, `IsClient()`, `IsMultiplayer()`, `IsDedicatedServer()` (`3_game/global/game.c`). For code that must
not exist on one side, vanilla uses `#ifdef SERVER` / `#ifndef SERVER` (e.g. `PlayerBase.OnRPC`,
`4_world/entities/manbase/playerbase.c`). The server is the authority: it decides state and tells clients.

## RPC

Send (`3_game/gameplay.c`, `ScriptRPC : ParamsWriteContext`):

```c
ScriptRPC rpc = new ScriptRPC();
rpc.Write(amount);                           // write each value; read them back in the same order
rpc.Send(target, MY_RPC_ID, true, identity); // target object, id, guaranteed, recipient (null = everyone)
```

For a single value, `GetGame().RPCSingleParam(target, id, new Param1<string>(s), true, identity)` does the same.

Receive on the target object (`Object.OnRPC`, `3_game/entities/object.c`):

```c
override void OnRPC(PlayerIdentity sender, int rpc_type, ParamsReadContext ctx)
{
    super.OnRPC(sender, rpc_type, ctx);
    if (rpc_type != MY_RPC_ID)
        return;
    int amount;
    if (!ctx.Read(amount))
        return;                              // Read returns false on a short or malformed message
    ...
}
```

On the server, `sender` is the client that sent the message, so validate everything it sent. Pick ids that are unlikely to
clash with vanilla's `ERPCs` and other mods.

## Net-synced variables

In the entity's constructor: `RegisterNetSyncVariableInt("m_State", 0, 3);` (also `...Bool`, `...Float`;
`3_game/entities/entityai.c`). Server: change the member, then call `SetSynchDirty();`. Client:
`override void OnVariablesSynchronized() { super.OnVariablesSynchronized(); ... }`.

## Persistence (items, entities)

`4_world/entities/itembase.c`:

```c
override void OnStoreSave(ParamsWriteContext ctx)
{
    super.OnStoreSave(ctx);
    ctx.Write(m_Charges);
}

override bool OnStoreLoad(ParamsReadContext ctx, int version)
{
    if (!super.OnStoreLoad(ctx, version))
        return false;
    if (!ctx.Read(m_Charges))
        return false;
    return true;
}
```

Adding a field to a mod already on live servers breaks loading of existing saves unless the mod versions its own data (for
example, writes a version int first and reads by it). Returning false fails the load of that object.

## Player lifecycle (server)

`5_mission/mission/missionserver.c`: `OnEvent(EventType, Param)` dispatches connects. `InvokeOnConnect(PlayerBase,
PlayerIdentity)`, `OnClientReadyEvent`, `OnClientRespawnEvent`, `OnClientDisconnectedEvent`, `PlayerDisconnected`. Hook them
with `modded class MissionServer`, `override` and `super`.

A `modded class MissionServer` override of `StartingEquipSetup` never runs: the mission's `init.c` `CustomMission` overrides
it without calling `super`.

## Timers and deferred work

`GetGame().GetCallQueue(CALL_CATEGORY_SYSTEM).CallLater(this.Fn, ms, repeat, args...)` and `Remove(this.Fn)`
(`ScriptCallQueue`, `2_gamelib/tools.c`). From `MissionServer.OnInit`, a `CallLater` only runs once the mission ticks,
about 20 s later. Keep per-frame work (`OnUpdate`, `EOnFrame`, `CommandHandler`) small: it runs for every instance, every
frame.

## Config files

`JsonFileLoader<T>.LoadFile(path, data, error)` / `SaveFile(path, data, error)` return a bool plus a message
(`3_game/tools/jsonfileloader.c`; `JsonLoadFile` / `JsonSaveFile` are the older void versions). `SaveFile` returns true
without checking the write. Use `$profile:MyMod/config.json` paths; `MakeDirectory` and `FileExist` are in
`1_core/proto/ensystem.c`. `FileSerializer.Open` only opens paths under the `$` roots, such as `$profile:`.
