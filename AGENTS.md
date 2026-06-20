# Agent guidelines

## General
- Keep README updated when you add/edit a skill

## Repository Shape

- Each skill lives at `skills/<skill-name>/SKILL.md` and starts with YAML frontmatter containing `name` and `description`.
- Use `references` in skill folder to add some examples, guides, etc
- Use `scripts` for reusable deterministic operations that complement the skill

## Editing Skills

- Add example requests where possible, at the bottom of the skill
- Consider using `references`, `scripts` or `assets` when SKILL gets bigger or where it makes sense
- Prefer using Python or `sh` for `scripts`
