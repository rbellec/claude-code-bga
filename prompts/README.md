# Prompt Templates

This directory contains the key prompts used at each phase of the BGA development workflow. They are templates — game-specific details need adapting, but the structure transfers.

## How to use

Each prompt file follows this format:

1. **Context** — what phase of development this prompt targets
2. **Prerequisites** — what should already be in place before using it
3. **The prompt** — the actual text given to Claude Code
4. **Expected outcome** — what files/changes the prompt should produce
5. **Notes** — what to watch for, common adjustments needed

## Adapting to your game

The prompts reference Quantum Tic-Tac-Toe specifics (spooky marks, entanglement graph, etc.). When adapting:

- Replace game-specific mechanics with your own rules
- Keep the structural elements: state machine design, DB schema reasoning, client render pattern
- The setup prompt (`01-setup.md`) is the most reusable; the mechanics prompt (`02-core-mechanics.md`) is the most game-specific

## Prompt sequence

| File | Phase | What it produces |
|------|-------|------------------|
| `01-setup.md` | Project scaffolding | Makefile, config files, empty state classes, first deploy |
| `02-core-mechanics.md` | Game logic | Game.php, state implementations, DB queries |
| `03-ui.md` | Client rendering | Game.js, CSS, notification handlers |
