# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running Tests

```bash
# Run fast skill tests (~2 min each)
cd tests/claude-code && ./run-skill-tests.sh

# Run specific test
./run-skill-tests.sh --test test-subagent-driven-development.sh

# Run with verbose output
./run-skill-tests.sh --verbose

# Run integration tests (10–30 min, requires `claude` CLI)
./run-skill-tests.sh --integration
```

Integration tests must be run **from the superpowers directory** (not from temp dirs), and require `"superpowers@superpowers-dev": true` in `~/.claude/settings.json`.

Analyze token usage from a session:
```bash
python3 tests/claude-code/analyze-token-usage.py ~/.claude/projects/<encoded-path>/<session>.jsonl
```

## Architecture

Superpowers is a Claude Code plugin (and Cursor/Codex/OpenCode compatible) that injects a skills system into coding agents.

### How Skills Load

1. **SessionStart hook** (`hooks/session-start` via `hooks/run-hook.cmd`) fires on every session and injects the full content of `skills/using-superpowers/SKILL.md` into the agent's context. This bootstraps the agent's awareness of the skills system.
2. The agent then uses the `Skill` tool to load individual skills on demand, based on their `description` field matching the current task.
3. `lib/skills-core.js` handles skill discovery, frontmatter parsing, and path resolution. Personal skills in `~/.claude/skills/` shadow superpowers skills of the same name.

### Skill File Format

Each skill lives at `skills/<name>/SKILL.md` with YAML frontmatter:

```yaml
---
name: skill-name-with-hyphens
description: Use when [triggering conditions, symptoms, context]
---
```

**Critical rules for the `description` field:**
- Must start with "Use when..." and describe *only triggering conditions*, never the workflow
- Summarizing the workflow in the description causes agents to skip reading the skill body and follow the description shortcut instead
- Max 1024 characters total for frontmatter; keep description under 500 chars

Supporting files (scripts, heavy reference docs) go in the same skill directory. Cross-reference other skills by name only (`superpowers:skill-name`), never with `@file` links (which force-load into context).

### Skill Development Workflow (TDD for Documentation)

New skills follow RED-GREEN-REFACTOR adapted to documentation:
1. **RED**: Run a pressure scenario *without* the skill and document what the agent does wrong
2. **GREEN**: Write the minimal skill that addresses those specific failures; re-test until compliant
3. **REFACTOR**: Identify new rationalizations agents use to avoid the rule; close each loophole explicitly

The `writing-skills` skill (see `skills/writing-skills/SKILL.md`) is the complete authoring guide.

### Default Workflow (Injected into Every Agent)

`brainstorming` → `writing-plans` → `subagent-driven-development` or `executing-plans`

The `using-superpowers` skill enforces that relevant skills are invoked *before* any action. This is mandatory, not optional.

### Test Structure

`tests/claude-code/` contains bash tests that run Claude Code in headless mode (`claude -p`) and verify behavior by parsing `.jsonl` session transcripts. `test-helpers.sh` provides shared assertion utilities (`assert_contains`, `assert_order`, `run_claude`, etc.).

### Cross-Platform Hook Compatibility

`hooks/run-hook.cmd` is a polyglot batch/bash script that handles Windows (Git for Windows bash discovery) and Unix (`exec bash`). Hook scripts use extensionless filenames to avoid Claude Code's Windows `.sh` auto-detection prepending `bash`.
