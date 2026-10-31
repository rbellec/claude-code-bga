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

## Checking for framework API drift

The authoritative API is not the wiki but two generated files BGA ships at the root of **every** Studio project and updates server-side (April projects carry October files): `_ide_helper.php` (PHP) and `bga-framework.d.ts` (JS). The wiki lags behind them (e.g. it omitted `addActionButton` options the d.ts had typed for months).

To check whether the skill is stale:

1. Download the current pair from any project of the account via SFTP (read-only; deploy config lives in the project Makefiles, never read the private key):
   ```
   sftp -i ~/.ssh/<key> -P 2022 -o IdentitiesOnly=yes USER@1.studio.boardgamearena.com
   > get GAMENAME/_ide_helper.php  > get GAMENAME/bga-framework.d.ts
   ```
2. `diff` against the baseline the skill was last synced to (last sync: stubs dated **2026-10-26**; keep a copy of the synced pair outside the repo, e.g. in the gitignored `.archives/`, or use the copies in a local project's `bga_initial_code_template/`).
3. For each added/changed/`@deprecated` member, decide: SKILL.md (only if a dev can violate it without loading a reference), the relevant `references/*.md`, or nothing.
4. Only then look at the wiki `Special:RecentChanges` for prose changes (checklists, guidelines, process pages) the stubs cannot show.

Real, running projects remain the tiebreaker for anything behavioral the stubs do not state (visibility quirks, runtime errors).

## Language

`README.md` is in English. `README.fr.md` is the French translation — keep both in sync when updating.

## Commit style

Follow the existing style: `type: short description` (conventional commits). Co-author commits with Claude when Claude wrote the code.
