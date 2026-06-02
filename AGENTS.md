# Agent guidelines

## General
- Keep README updated when you add/edit a skill

## Repository Shape

- Each skill lives at `skills/<skill-name>/SKILL.md` and starts with YAML frontmatter containing `name` and `description`.
- Consider using `references`, `scripts` or `assets` when SKILL gets bigger or where it makes sense
- Use `references` in skill folder to add some examples, guides, etc
- Use `scripts` for reusable deterministic operations that complement the skill
- Prefer using Python or `sh` for `scripts`

## Editing Skills

- Add example requests where possible, at the bottom of the skill
