# Claude Code + Board Game Arena: BGA Development Skill

A [Claude Code](https://claude.ai/claude-code) skill file for developing board games on [Board Game Arena](https://studio.boardgamearena.com) Studio — from published rules to a playable hotseat game.

## Getting started (5 minutes)

1. **Install Claude Code** — [CLI](https://claude.ai/claude-code), desktop app, or IDE extension
2. **Install [Claude in Chrome](https://chrome.google.com/webstore/detail/claude-in-chrome)** — needed for the automated test loop on BGA Studio
3. **Copy the skill file and references** into your Claude Code skills directory:
   ```bash
   mkdir -p ~/.claude/skills/board-game-arena
   cp SKILL.md ~/.claude/skills/board-game-arena/SKILL.md
   cp -r references ~/.claude/skills/board-game-arena/references
   ```
4. **Have your BGA Studio account ready** — game registered, SSH key configured ([BGA docs](https://en.doc.boardgamearena.com/Studio))
5. **Start Claude Code in your project directory** and tell it to implement your game:
   ```
   I want to implement [GAME NAME] on BGA Studio.
   Here are the rules: [paste rules or link to rulebook]
   My BGA username is [USERNAME] and the game name is [GAMENAME].
   ```

Claude Code will use the skill file to handle the BGA-specific setup (Makefile, config files, state machine, deploy workflow) and the browser extension to test directly on BGA Studio.

6. **(Optional) Allow git and deploy commands without prompting** — the skill commits automatically at each milestone. To avoid confirmation prompts, add to your `.claude/settings.local.json` (not committed to git):
   ```json
   {
     "permissions": {
       "allow": [
         "Bash(git *)",
         "Bash(make *)",
         "Bash(scp *)"
       ]
     }
   }
   ```

## What the skill file covers

- Project scaffolding (Makefile, directory structure, config files)
- BGA new framework patterns (PHP 8.4, state classes, notifications, globals)
- JavaScript client structure (ES6 modules, state handlers)
- **BGA library references** — detailed guides for Deck, BgaCards, Stock, Counter, Scrollmap, and 10 other BGA framework libraries, loaded on demand to keep context usage low (see `references/`)
- Automated deploy-test loop via Chrome extension
- Known BGA pitfalls and how to avoid them (SQL comments, table names, DB query gotchas)

See `TECHNICAL_NOTES.md` for detailed explanations of the pitfalls and their root causes.

## What you still need to do yourself

- Create the BGA Studio account and register your game
- Set up the SSH key for SFTP access (can take up to 1h to propagate)
- Manually quit stuck game tables when BGA's UI requires it
- Reconnect the Chrome extension if it drops
- Review the generated code — it's an alpha, not production-ready

## Games built with this skill

- [Visite Royale / Royal Visit](https://github.com/rbellec/bga_visite_royale) — card game, tug-of-war mechanics
- [Quantum Tic-Tac-Toe](https://github.com/rbellec/BGA_quantum_tic_tac_toe) — quantum superposition mechanics, entanglement graph, cycle detection
- [Go On Rasalva](https://github.com/rbellec/Go-On-Rasalva-on-BGA) — tile placement, resource management

## Article

A detailed write-up of the methodology and lessons learned is coming soon.

## License

MIT License. See [LICENSE](LICENSE).
