---
name: dayz-server-config
description: Configure a DayZ dedicated server or a test setup. Use when choosing server or client command-line parameters, locating the profile folder and its logs, pointing at a mission or storage folder, or editing Central Economy files (types.xml lifetime, nominal, min, restock, cfgspawnabletypes.xml, globals.xml) and predicting what the loot economy will do.
---

# DayZ server and test-setup configuration

Covers `DayZServer_x64.exe` and a client started offline. For writing the mod scripts themselves use the
`dayz-enforce-script` skill.

## Start a test server

```
DayZServer_x64.exe -config=serverDZ.cfg -profiles=profile -mission=mpmissions\dayzOffline.chernarusplus -port=2302 -mod=@MyMod -dologs -adminlog
```

- `-profiles=<dir>`: everything the server writes goes here (below).
- `-mission=<dir>`: the mission folder. Its name after the dot, `dayzOffline.chernarusplus`, picks the landscape
  (`chernarusplus`), and the mission's `storage_1` holds persistence.
- `-config=<file>`: the server config.
- `-mod=<dirs>` loads mods on server and client (`;`-separated); `-serverMod=<dirs>` loads server-only mods.
- `-port=<n>`: the game port. Default is UDP 2302 plus one random port; `-port=2399` opens 2399 and 2401 and no random
  port. The config's `steamQueryPort` is a separate listening UDP port.

Done when the profile folder holds a fresh `script_*.log` and the process stays up.

## Parameter syntax

- Names ignore case: `-MYFLAG=x` and `-myFlag=x` are the same parameter.
- `-name` alone and `-name=` both count as present with an empty value. The value is everything after the first `=`:
  `-key=a=b` gives `a=b`. One argument with spaces (`-key=two words`) keeps them.
- `-name` and `+name` are the same; `/name` is ignored.
- Scripts see every parameter on the command line, known to the engine or not, through `GetCLIParam` and `IsCLIParam`.
  Pass the name without the dash (`IsCLIParam("myFlag")`). Use this for your own mod switches.
- `-par=<file>` reads parameters from a text file, one per line (`-adminlog`, `-pid=<file>`). A `.par` file in class syntax
  did nothing.

An unknown name or a no-op parameter raises no warning, so verify an effect (a log line, a file, a port) rather than
trusting the boot.

## Profile folder and logs

With `-profiles=<dir>` the server writes there:
- the engine log (`.RPT`), `script_*.log` and the error log. Read the newest `script_*.log` first for mod problems.
- `Users/<name>/` (default `Users/Survivor/`; `-name=<n>` renames it) and `DataCache/`.
- `crash_<time>.log` when a script fault or crash happens.
- `<exe>_<time>.ADM` when `-adminlog` is on (first line `AdminLog started on ...`).
- `-pid=<file>` writes a decimal process id plus CRLF to that file, handy for scripts that stop the server.

Useful switches:
- `-dologs` grows the engine log from ~1,000 to ~5,800 lines on a fresh boot, mostly engine warnings. Leave it off unless you
  need them.
- `-doScriptLogs=1` defines `ENABLE_LOGGING` for scripts and lists each module's defines in the script log.
- `-inventoryDebug=2` floods the engine log with `INVENTORY (W): Inventory::OnItemLocationChangedEvent` lines.
- `-VMErrorMode=<mode>` sets what a script null dereference does. Default logs `SCRIPT (E): Virtual Machine Exception`,
  writes a crash log and carries on with the next statement; `silent` logs nothing and carries on; `fatal` stops the script
  and exits the server with code 5.
- `-limitFPS=<n>` caps server updates (`-limitFPS=10` gives about 9 per second instead of thousands); keeps a test server light.
- `-storage=<dir>` moves persistence files from the mission folder to `<dir>\storage_1\`; the log line
  `[StorageDirs] :: Selected storage directory` confirms it.
- `-cpuCount=<n>` adds the log line `Process affinity forced to ...`.
- `-donothing` exits with code 5 within a second and writes nothing.
- `-noFreezeCheck` stops the `FROZEN_*.mdmp` dump normally written after about 21 s of a blocked main thread;
  `-freezeCheck=<n>` makes that block end the server after about 10 s instead.

Script defines on the retail server: `SERVER` and `FEATURE_CURSOR`. `DEVELOPER` is not defined, so vanilla code behind
`#ifdef DEVELOPER` (many debug switches) never runs there.

## Client launch parameters

A client started with `DayZ_x64.exe` reads these in vanilla scripts (`3_game/dayzgame.c` unless noted):
- `-connect=<ip> -port=<n> -password=<pw>`: join that server straight from startup.
- `-mission=<dir>`: start that mission offline, with no server, when none of `-join`, `-connect`, `-autotest` is present.
  An offline mission is the quickest check for UI, input and camera code.
- `-join=...`: start through the join flow (checked before the others).
- `-loadingTest`: show progress text on the loading screen.
- `-doLogs`: turn on script `Debug` logging. Per-category switches: `-doActionLog -doSymptomLog -doInvMoveLog
  -doInvReservLog -doInvHFSMLog -doSyncLog -doQuickbarLog -doBaseBuildingLog -doWeaponLog -doWeatherLog
  -doRadioGameplayLog` (`3_game/tools/debug.c`).
- `-disablePersonalLight`: never switch on the personal light (`4_world/entities/manbase/playerbaseclient.c`).
- `-missionLoaderPath=<file>`: mission list file for the mission loader window (`5_mission/gui/scriptedwindows/missionloader.c`).

## Central Economy in one screen

The CE spawns and removes loot from `types.xml` and the other files in the mission folder. The rules to start from:
- With no player connected the CE idles and no lifetime changes (vanilla `IdleModeStartup 1`). To test cleanup or respawn on an
  empty server set `IdleModeStartup` and `IdleModeCountdown` to 0 in `globals.xml`.
- Lifetime counts down in passes about every 52 s, ~50 s at a time. Removal happens up to a minute after the lifetime ends,
  never before.
- A new item starts with a random lifetime between 1x and 2x its `types.xml` value.
- Editing `types.xml` doesn't change items already in the world, and deleting an entry doesn't remove the type's stored values.
- To watch cleanup happen, turn on `log_ce_lootcleanup` in `cfgeconomycore.xml`: removals log `<cleanup> Depleted:` lines.
