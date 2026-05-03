# Claude Code + Board Game Arena: BGA Development Skill

A [Claude Code](https://claude.ai/claude-code) skill for developing board games on [Board Game Arena](https://studio.boardgamearena.com) Studio — from published rules to a playable game.

Distributed as both a **Claude Code plugin** and a **Vercel `skills` package** — pick whichever fits your setup.

## Install

### Option A — Claude Code plugin (recommended)

In Claude Code:

```
/plugin marketplace add rbellec/claude-code-bga
/plugin install board-game-arena@claude-code-bga
```

### Option B — `npx skills`

From your project (or anywhere with `-g` for global):

```bash
npx skills add rbellec/claude-code-bga -a claude-code
```

### Option C — Manual copy

```bash
git clone https://github.com/rbellec/claude-code-bga.git
mkdir -p ~/.claude/skills
ln -s "$PWD/claude-code-bga/skills/board-game-arena" ~/.claude/skills/board-game-arena
```

## Getting started

1. **Install [Claude in Chrome](https://chrome.google.com/webstore/detail/claude-in-chrome)** — needed for the automated test loop on BGA Studio.
2. **Have your BGA Studio account ready** — game registered, SSH key configured ([BGA docs](https://en.doc.boardgamearena.com/Studio)).
3. **Start Claude Code in your project directory** and tell it what you want to build:
   ```
   I want to implement [GAME NAME] on BGA Studio.
   Here are the rules: [paste rules or link to rulebook]
   My BGA username is [USERNAME] and the game name is [GAMENAME].
   ```

Claude Code will use the skill to handle the BGA-specific setup (Makefile, config files, state machine, deploy workflow) and the browser extension to test directly on BGA Studio.

### Optional: PHP code quality loop (advanced)

If you're comfortable with PHP tooling, the skill includes an optional quality loop that runs PHP-CS-Fixer, Rector, PHPStan (level max), and PHPMD on every commit. Read-only by default, ~10 s per run, installed once into a shared `~/.bga-tools/` toolbox. See [`references/php-code-quality.md`](skills/board-game-arena/references/php-code-quality.md) for the full setup. **Skip this if you're not comfortable installing PHP dev tools** — the skill works fine without it.

### Optional: skip permission prompts for git/make/scp

The skill commits automatically at each milestone and deploys via `scp`/`make`. To avoid confirmation prompts, add to your `.claude/settings.local.json` (not committed):

```json
{
  "permissions": {
    "allow": ["Bash(git *)", "Bash(make *)", "Bash(scp *)"]
  }
}
```

## What the skill covers

- Project scaffolding (Makefile, directory structure, config files)
- BGA new framework patterns (PHP 8.4, state classes, notifications, globals)
- JavaScript client structure (ES6 modules, state handlers)
- **BGA library references** — detailed guides for Deck, BgaCards, Stock, Counter, Scrollmap, and 10 other BGA framework libraries, loaded on demand to keep context usage low (see [`skills/board-game-arena/references/`](skills/board-game-arena/references/))
- **Optional PHP code quality loop** — PHP-CS-Fixer, Rector, PHPStan (level max), PHPMD wired as `make audit`. See [`references/php-code-quality.md`](skills/board-game-arena/references/php-code-quality.md).
- Automated deploy-test loop via Chrome extension
- Known BGA pitfalls and how to avoid them (SQL comments, table names, DB query gotchas)

See [`TECHNICAL_NOTES.md`](TECHNICAL_NOTES.md) for the *why* behind each rule.

## What you still need to do yourself

- Create the BGA Studio account and register your game
- Set up the SSH key for SFTP access (can take up to 1h to propagate)
- Manually quit stuck game tables when BGA's UI requires it
- Reconnect the Chrome extension if it drops
- Review the generated code — it's an alpha, not production-ready

## Games built with this skill

- [Quantum Tic-Tac-Toe](https://github.com/rbellec/BGA_quantum_tic_tac_toe) — quantum superposition mechanics, entanglement graph, cycle detection
- [Go On Rasalva](https://github.com/rbellec/Go-On-Rasalva-on-BGA) — tile placement, resource management
- Visite Royale / Royal Visit — card game, tug-of-war mechanics *(repo will be made public once the publisher license is granted)*
- [Duelly](https://github.com/rbellec/bga_duelly) — adaptation in progress
- [Renartefact](https://github.com/rbellec/bga_renartefact) — adaptation in progress

## Articles

- **Article 1** *(in progress)* — methodology and lessons learned from adapting a board game with Claude Code.
- **Article 2** *(planned)* — the optional PHP quality loop in detail: setup, tuning for BGA-specific code, and a real first-run audit on a vanilla project.

## License

MIT License. See [LICENSE](LICENSE).
