# Claude Code + Board Game Arena: AI-Assisted Game Development

## For BGA game developers: quick start

**Want to use Claude Code to build your BGA game?** This repo provides a skill file and workflow that give Claude Code the knowledge it needs to work effectively within BGA Studio's constraints.

> **This is a work in progress.** The skill file and workflow are being actively refined through multiple game experiments. They work, but expect rough edges. Your feedback is valuable — please [open an issue](../../issues) if something breaks, is missing, or could be clearer.

### Getting started (5 minutes)

1. **Install Claude Code** — [CLI](https://claude.ai/claude-code), desktop app, or IDE extension
2. **Install [Claude in Chrome](https://chrome.google.com/webstore/detail/claude-in-chrome)** — needed for the automated test loop on BGA Studio
3. **Copy the skill file** into your Claude Code skills directory:
   ```bash
   mkdir -p ~/.claude/skills/bga-alpha
   cp SKILL.md ~/.claude/skills/bga-alpha/SKILL.md
   ```
4. **Have your BGA Studio account ready** — game registered, SSH key configured ([BGA docs](https://en.doc.boardgamearena.com/Studio))
5. **Start Claude Code in your project directory** and tell it to implement your game:
   ```
   I want to implement [GAME NAME] on BGA Studio.
   Here are the rules: [paste rules or link to rulebook]
   My BGA username is [USERNAME] and the game name is [GAMENAME].
   ```

Claude Code will use the skill file to handle the BGA-specific setup (Makefile, config files, state machine, deploy workflow) and the browser extension to test directly on BGA Studio.

### What the skill file covers

- Project scaffolding (Makefile, directory structure, config files)
- BGA new framework patterns (PHP 8.4, state classes, notifications, globals)
- JavaScript client structure (ES6 modules, state handlers)
- Automated deploy-test loop via Chrome extension
- Known BGA pitfalls and how to avoid them (SQL comments, table names, DB query gotchas)

### What you still need to do yourself

- Create the BGA Studio account and register your game
- Set up the SSH key for SFTP access (can take up to 1h to propagate)
- Manually quit stuck game tables when BGA's UI requires it
- Reconnect the Chrome extension if it drops
- Review the generated code — it's an alpha, not production-ready

See `workflow.md` for the full step-by-step development sequence, and `prompts/` for example prompts adapted from the first experiment.

---

## About this project

This repository documents a method for developing board games on [Board Game Arena](https://studio.boardgamearena.com) using [Claude Code](https://claude.ai/claude-code) as the primary development tool — from published rules to a playable hotseat game, without manual code writing.

The first experiment uses **Quantum Tic-Tac-Toe** (Allan Goff, AJP 2006). Further experiments with other games are in progress. The point of this repo is not any specific game — it's the **method**: a reproducible workflow for using an AI coding assistant within a constrained, real-world environment.

### What this demonstrates

1. **Constraint-driven development** — BGA is a fixed environment: specific PHP framework, SFTP-only deployment, no shell access, Svelte-based UI. The AI must work within these constraints, not around them.
2. **Rules as spec** — The published game rules serve as the formal specification. No product manager, no design doc — just a rulebook.
3. **Browser as test harness** — With no SSH access to BGA servers, the browser (via Claude in Chrome) is the only way to observe runtime behavior, read errors, and verify game state.
4. **Skill files as accumulated knowledge** — The `SKILL.md` captures BGA-specific patterns, pitfalls, and framework behaviors discovered during development. It persists across sessions and prevents the same mistakes from being made twice.

### What this does NOT claim

- That AI can autonomously develop production software
- That the resulting game code is optimal or complete
- That this approach scales to complex games without significant human oversight

## Repository structure

```
README.md               ← you are here
SKILL.md                ← Claude Code skill file for BGA development
workflow.md             ← step-by-step development workflow
CONTRIBUTING.md         ← how to contribute or adapt this method
prompts/
  README.md             ← how to use the prompt templates
  01-setup.md           ← project scaffolding and BGA setup
  02-core-mechanics.md  ← game logic implementation
  03-ui.md              ← client-side rendering and interaction
screenshots/            ← annotated screenshots of key moments
game/                   ← the actual game source code (quantictactoe)
experiments/            ← results from additional game experiments
```

## Results

This workflow has been developed and tested on two games so far:

### Quantum Tic-Tac-Toe — [source](https://github.com/rbellec/BGA_quantum_tic_tac_toe)

*Completed in a single Claude Code session (~3 hours wall time):*

- Full BGA game implementation: 4 PHP state classes, 1 JS client, SQL schema, config files
- Working game mechanics: quantum move placement, entanglement graph, cycle detection (DFS), collapse cascade, victory detection
- 3 bugs found and fixed during the test loop (SQL comment stripping, table name conflict, `getCollectionFromDb` key collision)
- First complete game played end-to-end on BGA Studio (7 moves, 2 collapses, GoOn0 wins)

*Metrics:*

- Lines of code produced: ~850 (PHP + JS + SQL + CSS)
- Deploy-test cycles: ~15
- Human interventions: ~5 (SFTP key setup, Express Stop on stuck tables, browser extension reconnect)

### Go On Rasalva — [source](https://github.com/rbellec/Go-On-Rasalva-on-BGA)

*(Results to be documented)*

## Limitations

- **Early stage** — the skill file has been validated on two games so far; more experiments are needed
- **Simplified rules only** — the first experiment implemented a simplified variant; full rules require more iterations
- **No tests** — all validation done via the browser test loop; no unit tests
- **Minimal UI** — functional but not polished
- **BGA-specific knowledge required** — the skill file encodes hard-won framework knowledge; without it, the AI hits the same pitfalls repeatedly
- **Browser extension fragility** — the Claude in Chrome connection can drop mid-session
- **No AI opponent** — 2-player hotseat only

## Next steps

- [ ] Validate the workflow on 2-3 more games of increasing complexity
- [ ] Extract a fully generic skill file (remove quantum-tic-tac-toe specifics)
- [ ] Fill in the prompt templates with real, tested examples
- [ ] Document token/cost usage per phase
- [ ] Add unit testing patterns for BGA game logic
- [ ] BGA built-in tutorial integration

## License

MIT License. See [LICENSE](LICENSE).

The game *Quantum Tic-Tac-Toe* was invented by Allan Goff. Game rules are credited and attributed; this repository contains only the BGA implementation code and the development methodology.
