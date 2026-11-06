# dayz-skills

Agent skills for DayZ modding (`skills/<name>/SKILL.md`) and the README that describes them.

- One folder per skill; its `name` frontmatter matches the folder. A new or renamed skill updates the README in the same commit
  when the README lists skills.
- State a rule plainly, with no tags, test ids or notes on how it was found.
- A skill's examples compile: any Enforce Script in it is checked in a mod before the commit.

## Agent skills

### Issue tracker

GitHub Issues on `maca134/dayz-skills`, through the `gh` CLI. See `docs/agents/issue-tracker.md`.

### Triage labels

The five default roles, each label named as its role (`needs-triage`, `needs-info`, `ready-for-agent`, `ready-for-human`,
`wontfix`). See `docs/agents/triage-labels.md`.

### Domain docs

Single-context: `GLOSSARY.md` and `docs/adr/` at the repo root, created when first needed. See `docs/agents/domain.md`.
