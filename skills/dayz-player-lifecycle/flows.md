# Server flows

The order inside `MissionServer.OnEvent` (`5_mission/mission/missionserver.c`) for each event. Every step is a method you can
override; arrows are calls made in that order.

## Prepare

`ClientPrepareEventTypeID`: the mission syncs its config data to the client, then calls
`OnClientPrepareEvent(identity, useDB, pos, yaw, preloadTimeout)`. Vanilla sets `useDB = true` when the server has a database,
otherwise `useDB = false` with a fixed position. A mission's `init.c` often overrides it to choose the spawn.

## New character

`ClientNewEventTypeID` → `OnClientNewEvent(identity, pos, ctx)` → `InvokeOnConnect(player, identity)` → player list sync and
lighting RPCs. If `OnClientNewEvent` returns null the event stops there and `InvokeOnConnect` never runs.

Inside `OnClientNewEvent`: it reads the login data, picks a character type (or one from the spawn preset file), calls
`CreateCharacter(identity, pos, ctx, characterName)` (creates the player and selects it for the identity), then
`EquipCharacter(charData)`, which creates the chosen attachments and ends with `StartingEquipSetup(player, true)`.

```c
modded class MissionServer
{
    override PlayerBase OnClientNewEvent(PlayerIdentity identity, vector pos, ParamsReadContext ctx)
    {
        PlayerBase player = super.OnClientNewEvent(identity, pos, ctx);
        if (player)
            player.GetInventory().CreateInInventory("Rag");   // once per fresh character
        return player;
    }
}
```

## Loaded character

`ClientReadyEventTypeID` → `OnClientReadyEvent(identity, player)` (selects the player for the identity) →
`InvokeOnConnect(player, identity)` → player list sync and lighting RPCs. The character already carries its saved inventory;
persistence hooks on its items are in `patterns.md`.

## Reconnect, respawn, death

- `ClientReconnectEventTypeID` → `OnClientReconnectEvent(identity, player)` → `PlayerBase.OnReconnect()`.
- `ClientRespawnEventTypeID` → `OnClientRespawnEvent(identity, player)`. When the old body is unconscious or restrained,
  vanilla logs it and kills it (`SetHealth("", "", 0.0)`). The client asks for a respawn with `GetGame().RespawnPlayer()` (the
  in-game menu's Respawn button); `CanRespawnPlayer()` says whether it is allowed.
- `PlayerBase.EEKilled` runs when the player dies. Vanilla's version also records the death in the database and syncs the
  respawn mode to the player's client; call `super`.

## Disconnect and logout timer

`ClientDisconnectedEventTypeID` → `OnClientDisconnectedEvent(identity, player, logoutTime, authFailed)`:

1. Server with a database, `authFailed` false, player alive and not already logging out: the player goes into the logout lists
   with an end time of now + `logoutTime` seconds, and the client is told the time. The handler returns and the character stays
   in the world.
2. Any other case (no database, auth failed, player dead): the client is told 0 and `PlayerDisconnected` runs at once.
3. Each frame, `UpdateLogoutPlayers` checks the lists; when an end time passes it calls
   `PlayerDisconnected(player, identity, uid)`, with `identity` read from the player (it can be null).
4. `LogoutCancelEventTypeID` (the player cancelled) removes the player from both lists; nothing else runs.

`PlayerDisconnected(player, identity, uid)`:

1. Returns at once when `player` is null.
2. `InvokeOnDisconnect(player)` → `PlayerBase.OnDisconnect()`.
3. With a database: `player.Save()`, then the database releases the character.
4. `player.ReleaseNetworkControls()`, then `HandleBody(player)`: an alive player who is unconscious or restrained is killed
   (the body stays) unless the kick reason is server exit, admin kick-all, kick-all or shutdown; any other alive player is
   deleted.
5. The identity is disconnected, and the player list is re-synced one second later.

To run code for each leaving player, override `PlayerDisconnected` and act before `super` (the player is still present and not
yet saved). To react when a logout starts, override `OnClientDisconnectedEvent`.