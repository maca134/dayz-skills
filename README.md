# dayz-skills

Agent skills for DayZ modding: instructions an AI coding agent (Claude Code, Codex, or any agent that reads `SKILL.md`
skills) loads when it works on a DayZ mod, such as writing Enforce Script, packing and testing a mod, or reading a server log.

- **One folder per skill**, under `skills/`, each with a `SKILL.md`.
- **Built for** DayZ 1.30.
- No skills yet: the first ones are on the way.

## Layout

```
skills/
  <skill-name>/
    SKILL.md          the skill: frontmatter, then the instructions
    reference.md      optional: longer material SKILL.md points to
    scripts/          optional: helper scripts the skill runs
```

`SKILL.md` starts with YAML frontmatter:

```markdown
---
name: <skill-name>
description: What the skill does and when to use it, in one or two sentences. The agent reads only this to decide whether to load the skill.
---

# <Skill title>

The instructions.
```

- `name` matches the folder name: lowercase words joined by hyphens.
- `description` says when to load it ("Use before writing any DayZ .c script"), not just what it covers.
- Keep `SKILL.md` short; move long tables and examples into files beside it and say when to read them.

## Install

Copy (or symlink) a skill's folder into your agent's skills folder:

- Claude Code: `~/.claude/skills/<skill-name>/` for every project, or `.claude/skills/<skill-name>/` in one project.
- Other agents: the folder their docs name for skills; the `SKILL.md` format is the same.
