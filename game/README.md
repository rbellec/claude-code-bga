# Game Source Code

This directory contains the actual BGA game implementation produced during the experiment.

## What's here

The complete source code for **Quantum Tic-Tac-Toe** as deployed to BGA Studio:

- PHP backend (Game.php, state classes)
- JavaScript client (Game.js)
- Database schema (dbmodel.sql)
- Configuration files (gameinfos.inc.php, stats.json, etc.)
- Assets (CSS, SVG)
- Makefile (lint + deploy)

## What's NOT here

- The BGA scaffold (`bga_initial_code_template/`) — download your own from BGA Studio
- Runtime files or BGA framework code — proprietary to BGA
- Credentials or SSH keys

## How this relates to the experiment

This code was produced entirely by Claude Code from a single session. It is a working alpha, not production-ready. Known limitations are documented in the root README.

The code should be read alongside `workflow.md` (which documents the order of creation) and the `prompts/` directory (which shows what instructions produced each component).
