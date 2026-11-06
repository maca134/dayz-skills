---
name: dayz-player-lifecycle
description: Hook a DayZ player's session from a mod, server or client side. Use before writing code that runs when a player connects, gets a character or loadout, respawns, dies, disconnects or sits in the logout timer, or when a hook fires with a null player or identity.
---

# Player lifecycle hooks

A session passes through server mission handlers (`MissionServer`), `PlayerBase` hooks and client events. Pick the hook for the
moment you need, then read what is valid there. Enforce rules (override signatures, `super`, the null rule) are in the
`dayz-enforce-script` skill; its `patterns.md` has the short version of this page.

Files next to this one:
- `flows.md`: the server handlers step by step (new character, loaded character, respawn, disconnect and logout timer), with
  vanilla's code shape for each. Open it before overriding one of them.

## Pick the hook

| You need to run code when | Hook | Side |
|---|---|---|
| a client is about to get a character (DB or fresh, spawn position) | `MissionServer.OnClientPrepareEvent(identity, out useDB, out pos, out yaw, out preloadTimeout)` | server |
| a **new** character exists | `MissionServer.OnClientNewEvent`, after `super` returns the player | server |
| a new character needs a loadout | `MissionServer.StartingEquipSetup(player, clothesChosen)`, or after `super` in `OnClientNewEvent` | server |
| a character was **loaded from the saved world** | `MissionServer.OnClientReadyEvent(identity, player)` | server |
| either of those two, one place | `MissionServer.InvokeOnConnect(player, identity)` or `PlayerBase.OnConnect()` | server |
| the client's controlled player is loaded and the HUD is up | `PlayerBase.OnPlayerLoaded()`, guarded by `IsControlledPlayer()` | client (also runs on the server) |
| the client's session is ready | `DayZGame.OnEvent` with `MPSessionPlayerReadyEventTypeID` | client |
| the player pressed Respawn | `MissionServer.OnClientRespawnEvent(identity, oldPlayer)` | server |
| a player dies | `PlayerBase.EEKilled(killer)` | server and client |
| a client left or was kicked, the logout timer may start | `MissionServer.OnClientDisconnectedEvent(identity, player, logoutTime, authFailed)` | server |
| the logout is final and the character is about to be saved and removed | `MissionServer.PlayerDisconnected(player, identity, uid)` or `PlayerBase.OnDisconnect()` | server |
| a logout was cancelled | the `LogoutCancelEventTypeID` case of `MissionServer.OnEvent` (no dedicated method) | server |

Hook with `modded class MissionServer` or `modded class PlayerBase`, write `override`, and call `super` first unless you mean
to replace the step. Copy each signature exactly from vanilla (`5_mission/mission/missionserver.c`,
`4_world/entities/manbase/playerbase.c`).

```c
modded class MissionServer
{
    override void InvokeOnConnect(PlayerBase player, PlayerIdentity identity)
    {
        super.InvokeOnConnect(player, identity);
        if (player && identity)
            Print("joined: " + identity.GetId() + " " + identity.GetName());
    }

    override void PlayerDisconnected(PlayerBase player, PlayerIdentity identity, string uid)
    {
        Print("leaving: " + uid);                // before super: the player is still in the world
        super.PlayerDisconnected(player, identity, uid);
    }
}
```

## What is valid where

| Moment | Identity | Player |
|---|---|---|
| `OnClientPrepareEvent` | the `identity` argument | none yet |
| `OnClientNewEvent`, before `super` | the `identity` argument | none yet |
| `OnClientNewEvent`, after `super` | argument | the returned player, with its loadout |
| `OnClientReadyEvent`, `InvokeOnConnect` | argument; `player.GetIdentity()` | valid |
| `OnClientRespawnEvent` | argument | the **old** character (dead, unconscious or restrained), not the new one |
| `OnClientDisconnectedEvent` | argument | valid; the event does nothing when it carries no player |
| `PlayerDisconnected` | **may be null** after a logout timer | may be null (already removed) |
| `PlayerBase.EEKilled` | `GetIdentity()` on the server | the dying player |

- Key anything per player by the `uid` string `PlayerDisconnected` receives, or by `identity.GetId()` stored at connect time
  (a hashed id; `GetPlainId()` is the Steam id). The identity can be null when a logout timer ends.
- Null-check both arguments before calling a method on them; a method call on null is skipped silently and a field read ends
  the function.

## Which handlers fire for which case

- **First join, no saved character:** `OnClientPrepareEvent`, `OnClientNewEvent` (character created and equipped),
  `InvokeOnConnect`. `OnClientReadyEvent` does not run.
- **Join with a saved character** (the server has a database for the mission): `OnClientPrepareEvent` with `useDB = true`,
  `OnClientReadyEvent`, `InvokeOnConnect`. Boot a server and log to see whether `OnClientNewEvent` and `StartingEquipSetup`
  also run on your mission before giving saved characters anything there.
- **Disconnect:** `OnClientDisconnectedEvent` first. With a database and an alive player it starts the logout timer; any other
  case finishes at once. See `flows.md`.
- `InvokeOnConnect` serves both join paths. Put "every join" logic there, "fresh character only" logic after `super` in
  `OnClientNewEvent` or in `StartingEquipSetup`, and "returning character only" logic in `OnClientReadyEvent`.

## Steps for a new hook

1. Pick the row of "Pick the hook" and note the side. Code for one side goes behind `#ifdef SERVER` / `#ifndef SERVER` or an
   `IsServer()` check (see `patterns.md`).
2. Copy the vanilla signature, add `override`, call `super`.
3. Null-check every player and identity, using "What is valid where".
4. Log one line on entry with `Print`, boot a server, join with a client, leave and rejoin, and compare the log order with
   `flows.md`.

Done when the log shows your line once per join and once per leave, and no `SCRIPT (E)` lines. A hook on the death or
respawn path also needs one death and one respawn in that run.

## Traps

- A mission's `init.c` commonly overrides `StartingEquipSetup` without calling `super`; a modded override then never runs.
  Hook after `super` in `OnClientNewEvent` instead.
- `PlayerBase.OnPlayerLoaded` is queued from the player's constructor, so it runs on every instance of the player: the server
  and each client that sees it. Guard HUD work with `IsControlledPlayer()` and server work with `GetGame().IsServer()`.
- A character in the logout timer stays in the world, alive, until `PlayerDisconnected`; a count of online players taken from
  the world's player list includes it.
- `OnClientRespawnEvent` receives the old body. Code that needs the new character belongs on the new-character path.
