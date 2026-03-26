# Shakedown Run

A shakedown run is the first real voyage of a new vessel — every system tested under way, not in dry dock. This skill runs a full end-to-end validation of the Loadout CLI, exercising every command and verifying correct behavior at each step.

## Ground Rules

- **Fix as you go.** If any step fails, investigate the root cause and fix it before moving on. The shakedown isn't just a test — it's a chance to catch and repair issues while you have eyes on every system.
- **Clean up after yourself.** Every temp directory, fixture, and config change must be reversed at the end.
- **Report what you find.** Keep a running tally of passes, failures, and fixes applied.

## Pre-flight

Before anything else, determine where the loadout project lives. Check whether the current working directory contains `cmd/loadout/` and a `Makefile` — if so, this is the loadout project root. If not, ask the user for the path to their local `loadout` checkout.

Once identified, set it for the rest of the run:

```bash
LOADOUT_DIR="/path/to/loadout"  # replace with detected or user-provided path
```

Build the project and run quality gates before testing CLI behavior.

```bash
cd "$LOADOUT_DIR"
make vet
make test-race
go build -o .tmp/loadout ./cmd/loadout
```

All three must pass. If `make test-race` or `make vet` fails, fix the issue before continuing.

Alias for convenience:

```bash
LOADOUT=.tmp/loadout
```

## Validation Steps

Work through each section in order. Every command shown is a real invocation — run it, check the output, and record pass or fail.

### 1. Config Compatibility + Inventory

Verify that the existing config loads cleanly and inventory renders without error.

```bash
$LOADOUT inventory
```

Expected: a table of skills with status columns for each configured target. No errors, no panics.

### 2. Inspect

Pick a skill from the inventory list and inspect it.

```bash
$LOADOUT inspect <name>
```

Expected: skill metadata (name, description, tags, targets) followed by the SKILL.md content.

### 3. Equip / Unequip

Choose a skill that is **not currently installed** and **supports both claude and codex targets**. Use `$LOADOUT inspect <name>` to verify the skill's targets before equipping — not all skills support both.

```bash
$LOADOUT equip <name> --target claude
$LOADOUT equip <name> --target codex
```

Verify with `$LOADOUT inventory` that the skill shows as installed for both targets.

Then unequip:

```bash
$LOADOUT unequip <name> --target claude
$LOADOUT unequip <name> --target codex
```

Verify the skill returns to an uninstalled state.

### 4. Sync

```bash
$LOADOUT sync
```

Expected: repo pull succeeds, any outdated managed installs are refreshed. If nothing is outdated, that's fine — the command should still complete without error.

### 5. Doctor

```bash
$LOADOUT doctor
```

Expected: all checks pass. If any check fails, investigate and fix before proceeding.

### 6. Import

Create a temporary skill fixture and import it.

```bash
IMPORT_TMP=$(mktemp -d)
mkdir -p "$IMPORT_TMP/temp-shakedown-import"

cat > "$IMPORT_TMP/temp-shakedown-import/skill.json" << 'EOF'
{
  "name": "temp-shakedown-import",
  "description": "Temporary fixture for shakedown import test.",
  "tags": ["test"],
  "targets": ["claude", "codex"]
}
EOF

cat > "$IMPORT_TMP/temp-shakedown-import/SKILL.md" << 'EOF'
# Temp Shakedown Import

This is a temporary skill created by the shakedown run. It should be deleted before the run ends.

See [notes](references/notes.md) for supplementary material.
EOF

mkdir -p "$IMPORT_TMP/temp-shakedown-import/references"
cat > "$IMPORT_TMP/temp-shakedown-import/references/notes.md" << 'EOF'
# Notes

Supplementary reference file to verify extra files survive import.
EOF

$LOADOUT import "$IMPORT_TMP/temp-shakedown-import"
```

Verify with `$LOADOUT inventory` that `temp-shakedown-import` appears.

Verify that the extra file was preserved in the repo:

```bash
ls "$(grep -o '"repo_path": *"[^"]*"' ~/.config/loadout/config.json | sed 's/"repo_path": *"//;s/"//')/skills/temp-shakedown-import/references/notes.md"
```

Expected: the file exists. If missing, the import did not preserve extra files.

### 7. Delete

First, verify that the interactive confirmation prompt works by running delete without `--force`. This should fail with a non-zero exit because there is no interactive input to provide:

```bash
$LOADOUT delete temp-shakedown-import 2>&1 || true
```

Expected: the command exits with an error containing "confirmation cancelled". This confirms the safety prompt is working.

Now actually delete the skill using `--force` to skip the interactive confirmation:

```bash
$LOADOUT delete temp-shakedown-import --force
```

Verify with `$LOADOUT inventory` that it no longer appears. Clean up the temp directory:

```bash
rm -rf "$IMPORT_TMP"
```

### 8. Project-scoped Flow

Test project-local installation by creating a temporary project directory. The directory must be a git repo with `.claude/` and `.codex/` directories for project scope resolution to work.

```bash
PROJECT_TMP=$(mktemp -d)
cd "$PROJECT_TMP"
git init .
mkdir -p .claude .codex
```

Run a scoped equip, inventory, sync, and unequip cycle. Note that `--project` is a string flag requiring a path argument:

```bash
$LOADOUT equip <name> --target claude --project .
$LOADOUT inventory --project .
$LOADOUT sync --project .
$LOADOUT unequip <name> --target claude --project .
$LOADOUT inventory --project .
```

Expected: the skill appears as project-installed during the cycle and disappears after unequip. Return to the loadout directory and clean up:

```bash
cd "$LOADOUT_DIR"
rm -rf "$PROJECT_TMP"
```

### 9. Init (Fresh Setup)

Back up the current config and state, then test `init` with temporary target directories.

```bash
CONFIG_DIR="$HOME/.config/loadout"
cp "$CONFIG_DIR/config.json" "$CONFIG_DIR/config.json.bak"

INIT_TMP=$(mktemp -d)
$LOADOUT init --repo "$INIT_TMP/repo" --targets claude,codex --claude-skills "$INIT_TMP/claude-skills" --codex-skills "$INIT_TMP/codex-skills"
```

Verify that init completed by checking the config file exists. Note: `$LOADOUT doctor` will show warnings after a fresh init because the new local repo has no remote upstream and previously installed skills won't be in the new registry — this is expected. Then restore the originals:

```bash
cp "$CONFIG_DIR/config.json.bak" "$CONFIG_DIR/config.json"
rm "$CONFIG_DIR/config.json.bak"
rm -rf "$INIT_TMP"
```

### 10. Filesystem Safety

Test that Loadout refuses to destroy unmanaged directories and rejects symlinks in skill trees. All fixtures use isolated temp directories with explicit cleanup.

#### 10a. Unmanaged Install Target Protection

Create a same-name directory without a `.loadout` marker in a temp target root, then attempt to equip a skill into it.

```bash
SAFETY_TMP=$(mktemp -d)
SAFETY_TARGET="$SAFETY_TMP/claude-skills"
mkdir -p "$SAFETY_TARGET/temp-shakedown-import"
echo "user data" > "$SAFETY_TARGET/temp-shakedown-import/precious.txt"
```

Pick any skill from the repo that supports claude. Attempt to install it using the CLI with the temp target. Since `equip` installs to the configured target root (not an arbitrary path), test this at the Go level by invoking `install.Install` directly. Instead, use the simpler approach — pre-create a conflicting directory in the real claude skills path, attempt equip, and verify the error:

```bash
CLAUDE_ROOT=$(jq -r '.targets.claude.path' ~/.config/loadout/config.json)
CONFLICT_SKILL="test-skill"  # pick a real skill from inventory
mkdir -p "$CLAUDE_ROOT/$CONFLICT_SKILL"
echo "precious" > "$CLAUDE_ROOT/$CONFLICT_SKILL/user-file.txt"

$LOADOUT equip "$CONFLICT_SKILL" --target claude 2>&1 || true
```

Expected: the command fails with an error mentioning "not managed by loadout". Verify the unmanaged directory is preserved:

```bash
cat "$CLAUDE_ROOT/$CONFLICT_SKILL/user-file.txt"
```

Expected: prints "precious". Clean up:

```bash
rm -rf "$CLAUDE_ROOT/$CONFLICT_SKILL"
```

#### 10b. Unmanaged Remove Protection

Same pattern — create an unmanaged directory and attempt unequip:

```bash
mkdir -p "$CLAUDE_ROOT/$CONFLICT_SKILL"
echo "keep me" > "$CLAUDE_ROOT/$CONFLICT_SKILL/notes.txt"

$LOADOUT unequip "$CONFLICT_SKILL" --target claude 2>&1 || true
```

Expected: command fails with "not managed by loadout" error. Verify preservation:

```bash
cat "$CLAUDE_ROOT/$CONFLICT_SKILL/notes.txt"
```

Expected: prints "keep me". Clean up:

```bash
rm -rf "$CLAUDE_ROOT/$CONFLICT_SKILL"
```

#### 10c. Import Symlink Rejection

Create a temp skill fixture with a valid SKILL.md plus a symlinked file, then attempt import.

```bash
SYMLINK_TMP=$(mktemp -d)
mkdir -p "$SYMLINK_TMP/symlink-test-skill"

cat > "$SYMLINK_TMP/symlink-test-skill/skill.json" << 'EOF'
{
  "name": "symlink-test-skill",
  "description": "Fixture with symlink for safety test.",
  "targets": ["claude"]
}
EOF

cat > "$SYMLINK_TMP/symlink-test-skill/SKILL.md" << 'EOF'
# Symlink Test
This skill should never be imported.
EOF

echo "external secret" > "$SYMLINK_TMP/external-secret.txt"
ln -s "$SYMLINK_TMP/external-secret.txt" "$SYMLINK_TMP/symlink-test-skill/link.txt"

$LOADOUT import "$SYMLINK_TMP/symlink-test-skill" 2>&1 || true
```

Expected: import fails with an error mentioning "symlink". Verify no repo copy was created:

```bash
REPO_PATH=$(jq -r '.repo_path' ~/.config/loadout/config.json)
ls "$REPO_PATH/skills/symlink-test-skill" 2>&1 || true
```

Expected: directory does not exist. Clean up:

```bash
rm -rf "$SYMLINK_TMP"
```

#### 10d. Install Symlink Rejection

Create a repo skill fixture containing a symlink, then attempt to install from it.

```bash
SYMLINK_REPO_TMP=$(mktemp -d)
mkdir -p "$SYMLINK_REPO_TMP/skills/symlink-install-test"

cat > "$SYMLINK_REPO_TMP/skills/symlink-install-test/skill.json" << 'EOF'
{
  "name": "symlink-install-test",
  "description": "Repo fixture with symlink.",
  "targets": ["claude"]
}
EOF

cat > "$SYMLINK_REPO_TMP/skills/symlink-install-test/SKILL.md" << 'EOF'
# Symlink Install Test
This skill has a symlinked file.
EOF

echo "secret" > "$SYMLINK_REPO_TMP/secret.txt"
ln -s "$SYMLINK_REPO_TMP/secret.txt" "$SYMLINK_REPO_TMP/skills/symlink-install-test/leaked.txt"
```

This cannot be tested via the CLI directly (the CLI uses the configured repo), so verify the guard works by running the Go test that covers this path:

```bash
cd "$LOADOUT_DIR"
go test -run TestCopyDir_RejectsSymlinks ./internal/fsx/
```

Expected: test passes. Clean up:

```bash
rm -rf "$SYMLINK_REPO_TMP"
rm -rf "$SAFETY_TMP"
```

## Cleanup

Remove all build artifacts and temp directories created during the run.

```bash
cd "$LOADOUT_DIR"
rm -rf .tmp/
```

Confirm that no temp directories or backup files remain:

```bash
ls /tmp/ | grep shakedown || true
ls "$HOME/.config/loadout/" | grep bak || true
```

If either command produces output, remove the leftover files.

## Report

Summarize the results:

1. **Steps passed**: list each section that passed cleanly
2. **Fixes applied**: describe any issues found and how they were resolved (include file paths and a brief description of each change)
3. **Steps failed**: list any sections that could not be resolved (should be empty if the run was successful)

A clean shakedown means every system works under way. If fixes were needed, they're now part of the codebase — the vessel is stronger for having made the voyage.
