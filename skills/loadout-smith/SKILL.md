# Loadout Smith

You are a careful craftsperson forging new skills for a Loadout-managed repo.

## Goal

Create or update a skill directly under `skills/<name>/` so it follows Loadout's expected layout and metadata rules.

## Authoring Procedure

1. Choose a kebab-case skill name that matches `^[a-z][a-z0-9-]*$`.
2. Create or update `skills/<name>/skill.json` and `skills/<name>/SKILL.md`.
3. Keep the directory name and `skill.json.name` identical.
4. Write a short `description` that explains what the skill does for the agent.
5. Set `targets` explicitly. Default to both `claude` and `codex` unless the behavior or metadata is genuinely target-specific.
6. Add `tags` only when they help categorize the skill.
7. Add `claude` or `codex` metadata blocks only when that target needs platform-specific installed frontmatter keys or values.
8. Preserve any supporting files inside the skill directory, such as `references/`, when the instructions depend on them.

## Frontmatter References

- Claude frontmatter reference: https://code.claude.com/docs/en/skills#frontmatter-reference
- Codex and Agent Skills frontmatter reference: https://agentskills.io/specification#frontmatter

Use those references when deciding which frontmatter belongs in the shared skill body versus which keys should live under the `claude` or `codex` metadata blocks in `skill.json`.

## Output Requirements

- `skill.json` must be valid JSON.
- `skill.json` must include `name`, `description`, and a non-empty `targets` array.
- `SKILL.md` must read like instructions for the AI agent that will use the skill.
- The skill should fit the existing repo conventions before you stop.

## Rules

- Author the skill in-repo under `skills/<name>/`; do not create an external import fixture unless the user explicitly asks for one.
- Prefer one shared skill with per-target metadata over splitting into separate target-specific skills.
- Keep the skill focused on one job with concise instructions.
- Before finishing, verify that the final directory structure and metadata match the repo's documented format.
