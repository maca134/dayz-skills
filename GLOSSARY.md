# dayz-skills

Agent skills for DayZ modding: what an AI coding agent loads when it writes, builds or debugs a DayZ mod.

## Language

**Skill**:
One folder under `skills/` holding a `SKILL.md` an agent loads by its description.
_Avoid_: plugin, prompt

**Rule**:
A named way Enforce Script behaves that breaks code which looks valid (the line rule, the ref rule). Referred to by name, never by number.
_Avoid_: trap, gotcha, bug

**Loop**:
How an agent checks Enforce code it wrote: pack the mod, boot the game with it, read the script log.
_Avoid_: checker, harness
