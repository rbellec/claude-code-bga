# Claude Code + Board Game Arena: A Constrained Development Experiment

This repository documents a method for developing a board game on [Board Game Arena](https://studio.boardgamearena.com) (BGA) using [Claude Code](https://claude.ai/claude-code) as the primary development tool — from published rules to a playable hotseat game, without manual code writing.

The first experiment uses **Quantum Tic-Tac-Toe** (Allan Goff, AJP 2006). Further experiments with other games will follow. But the point of this repo is not any specific game — it's the **method**: a reproducible workflow for using an AI coding assistant within a constrained, real-world environment.

## What this demonstrates

1. **Constraint-driven development** — BGA is a fixed environment: specific PHP framework, SFTP-only deployment, no shell access, Svelte-based UI. The AI must work within these constraints, not around them.
2. **Rules as spec** — The published game rules serve as the formal specification. No product manager, no design doc — just a paper from the American Journal of Physics.
3. **Browser as test harness** — With no SSH access to BGA servers, the browser (via the Claude in Chrome extension) is the only way to observe runtime behavior, read errors, and verify game state.
4. **Skill files as accumulated knowledge** — A `SKILL.md` file captures BGA-specific patterns, pitfalls, and framework behaviors discovered during development. It persists across sessions and prevents the same mistakes from being made twice.

## What this does NOT claim

- That AI can autonomously develop production software
- That the resulting game code is optimal or complete
- That this approach scales to complex games without significant human oversight

The experiment produced a working alpha with known limitations (see [Results](#results) and [Limitations](#limitations)).

## Prerequisites

To reproduce this workflow on your own BGA game:

- [Claude Code CLI](https://claude.ai/claude-code) (or Claude Code desktop/IDE extension)
- [Claude in Chrome](https://chrome.google.com/webstore/detail/claude-in-chrome) browser extension (for the test loop)
- A [BGA Studio](https://studio.boardgamearena.com) developer account with a registered game
- SSH key configured for BGA Studio SFTP access
- PHP installed locally (for `make check` / lint)
- Published rules for your target game

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
experiments/            ← results from additional game experiments (future)
```

## How to use this on another BGA game

1. **Copy `SKILL.md`** into your Claude Code skills directory (`~/.claude/skills/bga-alpha/SKILL.md`). This gives Claude Code the BGA framework knowledge it needs.
2. **Read `workflow.md`** to understand the sequence: scaffold download, config files, PHP states, JS client, deploy-test loop.
3. **Adapt the prompts** in `prompts/` to your game's rules. The structure (setup, mechanics, UI) transfers to any turn-based BGA game.
4. **Run the test loop** described in the skill file: deploy via `make deploy`, create a table in BGA Studio, add a hotseat player, read console errors, fix, repeat. Each cycle takes ~30 seconds.

## Results

### Experiment 1: Quantum Tic-Tac-Toe

*Completed in a single Claude Code session (~3 hours wall time):*

- Full BGA game implementation: 4 PHP state classes, 1 JS client, SQL schema, config files
- Working game mechanics: quantum move placement, entanglement graph, cycle detection (DFS), collapse cascade, victory detection
- 3 bugs found and fixed during the test loop (SQL comment stripping, table name conflict, `getCollectionFromDb` key collision)
- First complete game played end-to-end on BGA Studio (7 moves, 2 collapses, GoOn0 wins)

*Metrics:*

- Lines of code produced: ~850 (PHP + JS + SQL + CSS)
- Deploy-test cycles: ~15
- Human interventions: ~5 (SFTP key setup, Express Stop on stuck tables, browser extension reconnect)

### Further experiments

Additional games will be tested using the same skill file and workflow. Results will be added here and in the `experiments/` directory as they are completed.

## Limitations

- **Simplified rules only** — simultaneous win scoring (half-point variant) not implemented
- **No tests** — all validation done via the browser test loop; no unit tests
- **Minimal UI** — collapse choice is text-only (no board highlighting or preview)
- **BGA-specific knowledge required** — the skill file encodes hard-won BGA framework knowledge; without it, the AI hits the same pitfalls repeatedly
- **Browser extension fragility** — the Claude in Chrome connection dropped once mid-session; manual reconnect needed
- **No AI opponent** — 2-player hotseat only

## Next steps

- [ ] Implement full scoring variant (half-point for simultaneous wins)
- [ ] Better collapse UI (highlight cycle path, preview collapse result)
- [ ] BGA built-in tutorial integration
- [ ] Extract a generic BGA skill file (not quantum-tic-tac-toe-specific)
- [ ] Add unit tests for graph algorithms
- [ ] Document token/cost usage per phase

## License

MIT License. See [LICENSE](LICENSE).

The game *Quantum Tic-Tac-Toe* was invented by Allan Goff. Game rules are credited and attributed; this repository contains only the BGA implementation code, not the game design itself.
