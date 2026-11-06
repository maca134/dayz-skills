---
name: dayz-persistence
description: Saving and loading state across DayZ server restarts. Use before writing OnStoreSave/OnStoreLoad, before adding or removing a saved field in a mod already running on servers, when a mod is removed or an old save logs "Scripted variables corrupted", when items vanish or never respawn (Central Economy lifetime, restock, hoarders), or when a mod keeps its own data in $profile: files.
---

# Persistence in DayZ

Three separate stores, each with its own rules: the **entity storage** (items, vehicles, bases: your `OnStoreSave` data
rides inside it), the **Central Economy** (decides whether an object lives on, and respawns types), and **files in the
profile folder** (your own JSON or binary data). The basic `OnStoreSave`/`OnStoreLoad` shape is in the
`dayz-enforce-script` skill's `patterns.md`; this skill covers what happens around it.

## Write the version first

Every saved layout of your own gets a version number of your own, written before the data and read before anything else.
Never rely on the engine's `version` argument for your fields: it is the game's save version, not yours.

```c
class MyItem extends ItemBase
{
    protected int m_Charges;
    protected float m_Heat;                         // added in layout 2

    override void OnStoreSave(ParamsWriteContext ctx)
    {
        super.OnStoreSave(ctx);
        ctx.Write(2);                               // my layout version
        ctx.Write(m_Charges);
        ctx.Write(m_Heat);
    }

    override bool OnStoreLoad(ParamsReadContext ctx, int version)
    {
        if (!super.OnStoreLoad(ctx, version))
            return false;
        int mine;
        if (!ctx.Read(mine))
            return false;
        if (!ctx.Read(m_Charges))
            return false;
        if (mine >= 2 && !ctx.Read(m_Heat))         // read a field only from the layout that wrote it
            return false;
        return true;
    }
}
```

Rules that follow:
- Call `super` in both hooks. Leaving out `super.OnStoreSave` loses the item's own quantity (it came back at its default).
- Read exactly what was written, in the same order, and return true only then. Any other read pattern logs a corruption
  warning.
- A `Read` past the end of the data still returns true, with a value nobody wrote. A returned true does not prove the
  field existed: only your version number does.
- Write a version even if the class has no fields yet. An empty `OnStoreSave` still gets a load call, and a later field
  then has no old layout to tell apart.
- Health, quantity and similar built-in values are restored after `OnStoreLoad` returns. Read them in `AfterStoreLoad`;
  inside `OnStoreLoad` they still show defaults.
- `EEInit` runs before `OnStoreLoad`, so script fields are still at their defaults there. Do work that needs loaded
  values in `AfterStoreLoad`.
- A container's contents are saved first and loaded first: an item inside cargo has loaded before its container's
  `OnStoreLoad` runs.

## When a mod changes or goes away

An object is never deleted for failing to load; the server keeps it and logs the problem.

| Change | Result |
|---|---|
| Field added, no version | Old saves yield junk in the new field, plus a corruption warning. Fix: version first. |
| Field removed and the class no longer overrides `OnStoreLoad` | Item kept, the field at its default, warning logged. Keep reading (and discarding) old layouts by version instead. |
| Class stopped reading its saved data (or the load returns false) | Item kept with health and quantity; your fields stay at defaults (or what was read before the failure); warning logged. |
| Class removed from config | The object is gone; later objects in the file still load; log shows "Serious stream damage detected during load". |

To migrate: keep reading old layouts by version forever, or load an old layout once and write the new one at the next
save (the next save upgrades it).

## When it saves

- The server saves while it is **not idle**: a rolling pass roughly every 52 seconds, and one full save on entering idle
  mode, plus a clean shutdown. A **killed process keeps nothing since the last save.**
- With vanilla `globals.xml` (`IdleModeStartup 1`) an empty server starts idle, so with no player ever connecting the only
  save is at a clean shutdown. To test persistence in a short run, end the server through
  `GetGame().RequestExit(0)` (or the normal shutdown), not by killing it, and use a fixed `-storage=<folder>` between boots.
- Spawn flag `ECE_NOPERSISTENCY_WORLD` makes an object skip saving: it is not there after a restart.
- An item with no `types.xml` entry is saved and loaded like any other. Whether it also survives cleanup depends on its
  Central Economy lifetime (the `dayz-server-config` skill).

## Files in $profile:

For a mod's own data (config, player stats, logs) that should not ride in the entity storage.

- Paths must start with `$profile:` (e.g. `$profile:MyMod/data.json`). A path without the prefix opens for read but
  not for write or append, and `$profile:../x` is rejected.
- `MakeDirectory("$profile:MyMod")` first (returns true when created; false for a `..` path). `FileExist`, `DeleteFile`
  and `CopyFile` take the same prefix; `CopyFile` to a `$logs:` destination fails, so copy within `$profile:`.
- `FileSerializer.Open` modes WRITE, READ and APPEND work under `$profile:`; a handle opened for WRITE or APPEND is not
  readable and the reverse. Close before reopening in the other mode.
- JSON: `JsonFileLoader<T>.LoadFile` / `SaveFile` (shape in `patterns.md` of the `dayz-enforce-script` skill). Give the
  file a `version` field and load by it, as with `OnStoreLoad`. Handle a missing file by writing defaults.
- Write the file on a server event (a change, a timer, shutdown), never every frame.

## Done when

- Old saves from every layout you ever shipped load with no `Scripted variables corrupted` line in the RPT and no
  `Virtual Machine Exception` in the script log.
- You restarted the server twice on one `-storage` folder and the second boot read what the first wrote.
- Any item you added has a `types.xml` entry, or you know it is cleaned up after the default lifetime.
