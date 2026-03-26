# loadout-example-skills

A sample skills repo for [Loadout](https://github.com/sethdeckard/loadout), a skill manager for Claude and Codex.

This repo is for demonstrating structure and metadata patterns, not for acting as a serious starter library. The examples are intentionally playful and framed as an adventuring kit so the repo matches Loadout's inventory language.

## Directory structure

```
skills/
  debugging-compass/
    skill.json          # Skill metadata and target config
    SKILL.md            # Skill instructions (installed to target)
  bardic-review/
    skill.json
    SKILL.md
  field-rations/
    skill.json
    SKILL.md
  loadout-smith/
    skill.json
    SKILL.md
  signal-flare/
    skill.json
    SKILL.md
  warden-manual/
    skill.json
    SKILL.md
  watchfire-briefing/
    skill.json
    SKILL.md
  shakedown-run/
    skill.json
    SKILL.md
```

Every skill lives in `skills/<name>/` and must contain both `skill.json` and `SKILL.md`. `name` is the canonical identifier and must match the directory name.

## `skill.json` format

```json
{
  "name": "debugging-compass",
  "description": "A shared example skill with per-target metadata for tracing bugs through unfamiliar code.",
  "tags": ["demo", "debugging", "navigation"],
  "targets": ["claude", "codex"],
  "claude": {
    "allowed-tools": "Read, Grep, Bash",
    "compass-mode": "survey"
  },
  "codex": {
    "approval-mode": "suggest"
  }
}
```

### Required fields

| Field | Type | Rules |
|-------|------|-------|
| `name` | string | Canonical identifier. Must match `^[a-z][a-z0-9-]*$` and the directory name under `skills/`. |
| `description` | string | Short description of what the skill does. |
| `targets` | array | Non-empty list. Valid values: `"claude"`, `"codex"`. |

### Optional fields

| Field | Type | Description |
|-------|------|-------------|
| `tags` | array | Freeform tags for categorization. |
| `claude` | object | Target-specific config for Claude (see below). |
| `codex` | object | Target-specific config for Codex (see below). |

### Target-specific config

The `claude` and `codex` objects contain key-value pairs that become YAML frontmatter when the skill is installed. For example, given this `skill.json`:

```json
{
  "name": "bardic-review",
  "targets": ["claude"],
  "claude": {
    "allowed-tools": "Read, Grep",
    "disable-model-invocation": true
  }
}
```

Loadout will install `SKILL.md` with this frontmatter prepended:

```markdown
---
name: bardic-review
description: An intentional Claude-only example that demonstrates a single-target skill with Claude metadata.
allowed-tools: Read, Grep
disable-model-invocation: true
---

# Bardic Review
...
```

If a target has no config object (or an empty `{}`), only `name` and `description` appear in the frontmatter.

## What This Repo Demonstrates

This repo intentionally centers three core structural patterns:

| Pattern | Example | Why it exists |
|--------|---------|---------------|
| Shared skill, no target metadata | [field-rations](skills/field-rations/) | Demonstrates the simplest happy path |
| Shared skill, per-target metadata | [debugging-compass](skills/debugging-compass/) | Demonstrates one skill with target-specific config |
| Intentional single-target skill | [bardic-review](skills/bardic-review/) | Demonstrates that `targets` are explicit source truth and may be single-target |

Preferred authoring pattern:

- Default to both targets when the skill is portable
- Use per-target metadata inside one skill before splitting skills
- Use single-target only when compatibility or metadata is genuinely target-specific
- Importers should preserve declared `targets` rather than silently broadening compatibility

Additional examples in the repo extend those patterns without introducing a new format:

- [watchfire-briefing](skills/watchfire-briefing/) is another simple shared skill
- [warden-manual](skills/warden-manual/) shows that one target can have metadata while the other relies on default frontmatter
- [signal-flare](skills/signal-flare/) demonstrates the reverse single-target case with a Codex-only skill
- [loadout-smith](skills/loadout-smith/) demonstrates the preferred in-repo authoring workflow for creating a new Loadout skill and is useful in practice when you want an agent to scaffold one correctly

[loadout-smith](skills/loadout-smith/) and [shakedown-run](skills/shakedown-run/) are also genuinely useful skills, not just demos. `loadout-smith` helps an agent author a valid new skill in-repo, while `shakedown-run` runs a full end-to-end validation of the Loadout CLI itself, exercising every command and fixing issues found along the way.

## `SKILL.md` format

Plain markdown containing the skill instructions. This is the file that gets installed into the target's skills directory (`~/.claude/skills/` or `~/.codex/skills/`). Write it as instructions for the AI agent.

## Example skills

| Skill | Targets | Config | Description |
|-------|---------|--------|-------------|
| [field-rations](skills/field-rations/) | claude, codex | No target config | Simplest shared skill example |
| [debugging-compass](skills/debugging-compass/) | claude, codex | Both targets have metadata blocks | Shared skill with per-target metadata |
| [bardic-review](skills/bardic-review/) | claude | Claude-only metadata example | Intentional single-target skill |
| [watchfire-briefing](skills/watchfire-briefing/) | claude, codex | No target config | Another shared item in the same theme |
| [warden-manual](skills/warden-manual/) | claude, codex | Claude metadata only | Shared skill where only one target has extra config |
| [signal-flare](skills/signal-flare/) | codex | Codex-only metadata example | Reverse single-target case |
| [loadout-smith](skills/loadout-smith/) | claude, codex | No target config | Useful in-repo authoring example for new Loadout-compatible skills |
| [shakedown-run](skills/shakedown-run/) | claude, codex | Both targets have allowed-tools | End-to-end validation of the Loadout CLI |

## Usage with Loadout

```bash
# Point Loadout at this repo
loadout init --repo /path/to/loadout-example-skills

# List available skills
loadout inventory

# Install a skill for Claude
loadout equip debugging-compass --target claude

# Open the TUI
loadout
```

## Authoring vs. Import

Use the patterns in this repo, especially [loadout-smith](skills/loadout-smith/), when creating a new managed skill directly inside `skills/<name>/`.

Use import when a skill already exists outside the repo and you want Loadout to copy it into the managed `skills/` inventory, either through `loadout import` or from the TUI with `i`.
