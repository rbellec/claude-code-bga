# Development Workflow

<!-- PLACEHOLDER — Document the step-by-step workflow used during the experiment.

     This file should describe the actual sequence of work, not just
     the ideal process. Include what worked, what didn't, and what
     was discovered along the way.

     Suggested structure:

     ## Phase 0 — Prerequisites
     - BGA account setup, SSH key, game registration
     - Estimated time: (manual steps, not automatable)

     ## Phase 1 — Scaffold and Configuration
     - Download BGA scaffold via SFTP
     - Create local project structure
     - Write config files (gameinfos.inc.php, dbmodel.sql, stats.json, etc.)
     - First `make deploy` + verify on BGA Studio
     - Commit milestone

     ## Phase 2 — Core Game Logic
     - Game.php: state machine, graph algorithms, victory detection
     - State classes: PlayerTurn, CollapseChoice, CheckVictory, ComputeScores
     - Deploy + first table creation attempt
     - Bugs encountered at this stage and how they were diagnosed

     ## Phase 3 — Client (JS + CSS)
     - Game.js: board rendering, click handlers, notifications
     - CSS: grid layout, spooky/classical mark styling
     - Deploy + first visual test

     ## Phase 4 — Test Loop and Bug Fixes
     - The deploy→create→test→fix cycle
     - Each bug: symptom, diagnosis method, root cause, fix
     - Timeline: how many cycles, how long each took

     ## Phase 5 — First Complete Game
     - Moves played, cycles triggered, collapses executed
     - Victory detected and game ended correctly

     ## Observations
     - What the AI did well
     - Where human intervention was needed
     - What the skill file prevented vs what it couldn't prevent
-->
