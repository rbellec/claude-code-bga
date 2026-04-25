# CLAUDE.md — claude-code-bga

## What this repo is

A repository distributing the BGA development skill for Claude Code. It is structured as **both** a Claude Code plugin marketplace and a Vercel `skills` package — the same layout serves both ecosystems.

Layout:
- `.claude-plugin/plugin.json` — Claude Code plugin manifest
- `.claude-plugin/marketplace.json` — marketplace manifest (this repo registers itself as a single-plugin marketplace)
- `skills/board-game-arena/SKILL.md` — the skill file (primary artifact)
- `skills/board-game-arena/references/` — detailed guides for each BGA framework library (Deck, BgaCards, Stock, etc.), loaded on demand by the skill
- `TECHNICAL_NOTES.md` — explanations of the pitfalls encoded in SKILL.md (the *why* behind each rule)
- `README.md` / `README.fr.md` — install + usage instructions in English and French

## Skill file maintenance

`skills/board-game-arena/SKILL.md` is the source of truth. For local development, symlink it into your Claude Code skills directory so edits take effect immediately:

```bash
ln -s "$PWD/skills/board-game-arena" ~/.claude/skills/board-game-arena
```

When adding new BGA pitfalls or patterns:
1. Add the short rule to the appropriate section in `skills/board-game-arena/SKILL.md` (with fix in the error table if applicable)
2. Add the detailed explanation to `TECHNICAL_NOTES.md` (with a *why* section)

When adding or updating BGA library references:
1. Add/update the file in `skills/board-game-arena/references/` (follow the standard format: Setup, API, Example, Pitfalls)
2. Ensure the library appears in the quick-reference table in `SKILL.md` section 2.5
3. Update `skills/board-game-arena/references/README.md` index

When publishing a release: bump `version` in `.claude-plugin/plugin.json` so installed users see the update (omit it to auto-track commit SHA instead).

## Language

`README.md` is in English. `README.fr.md` is the French translation — keep both in sync when updating.

## Commit style

Follow the existing style: `type: short description` (conventional commits). Co-author commits with Claude when Claude wrote the code.
