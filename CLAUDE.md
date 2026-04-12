# CLAUDE.md — claude-code-bga

## What this repo is

This is a **methodology repo**, not a game implementation repo. It documents and provides tools for developing board games on Board Game Arena (BGA) Studio using Claude Code as the primary development tool.

Key artifacts:
- `SKILL.md` — the Claude Code skill file. This is the primary artifact. It is loaded into Claude Code sessions via a symlink at `~/.claude/skills/bga-alpha/`.
- `TECHNICAL_NOTES.md` — detailed explanations of the pitfalls encoded in SKILL.md (the *why* behind each rule)
- `workflow.md` — step-by-step development sequence
- `prompts/` — example prompts used at each development phase
- `game/` — actual game source code from the first experiment (Quantum Tic-Tac-Toe)
- `experiments/` — results from additional game experiments

## Skill file maintenance

`SKILL.md` is the source of truth for the skill. It is symlinked into `~/.claude/skills/bga-alpha/SKILL.md` — edits here take effect immediately in Claude Code sessions.

When adding new BGA pitfalls or patterns:
1. Add the short rule to the appropriate section in `SKILL.md` (with fix in the error table if applicable)
2. Add the detailed explanation to `TECHNICAL_NOTES.md` (with a *why* section)
3. Reference `TECHNICAL_NOTES.md` from the relevant SKILL.md rule

## Language

`README.md` is in English. `README.fr.md` is the French translation — keep both in sync when updating the README.

## Commit style

Follow the existing style: `type: short description` (conventional commits). Co-author commits with Claude when Claude wrote the code.
