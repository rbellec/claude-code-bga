# CLAUDE.md — claude-code-bga

## What this repo is

A repository for sharing the BGA development skill file for Claude Code. It provides everything needed for a developer to use Claude Code to implement a board game on Board Game Arena Studio.

Key artifacts:
- `SKILL.md` — the Claude Code skill file (primary artifact), symlinked at `~/.claude/skills/bga-alpha/`
- `references/` — detailed reference guides for each BGA framework library (Deck, BgaCards, Stock, etc.), loaded on demand by the skill
- `TECHNICAL_NOTES.md` — detailed explanations of the pitfalls encoded in SKILL.md (the *why* behind each rule)
- `README.md` / `README.fr.md` — usage instructions in English and French

## Skill file maintenance

`SKILL.md` is the source of truth for the skill. It is symlinked into `~/.claude/skills/bga-alpha/SKILL.md` — edits here take effect immediately in Claude Code sessions.

When adding new BGA pitfalls or patterns:
1. Add the short rule to the appropriate section in `SKILL.md` (with fix in the error table if applicable)
2. Add the detailed explanation to `TECHNICAL_NOTES.md` (with a *why* section)

When adding or updating BGA library references:
1. Add/update the reference file in `references/` (follow the standard format: Setup, API, Example, Pitfalls)
2. Ensure the library appears in the quick-reference table in `SKILL.md` section 2.5
3. Update `references/README.md` index

## Language

`README.md` is in English. `README.fr.md` is the French translation — keep both in sync when updating.

## Commit style

Follow the existing style: `type: short description` (conventional commits). Co-author commits with Claude when Claude wrote the code.
