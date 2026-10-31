---
name: board-game-arena
description: Implement a board game on Board Game Arena (BGA) Studio, from rules+graphics to a playable game. Covers project setup, new framework (2025+) architecture, PHP state machine, JS client, and the automated deploy→test loop.
disable-model-invocation: false
---

# Board Game Arena — From Rules to Playable Game

You are helping implement a board game on Board Game Arena Studio, using the **new BGA framework (2025+)**. The workflow has three phases: **Setup**, **Implementation**, and **Test Loop**.

---

## Prerequisites (verify before starting)

- BGA Studio account created, game registered at https://studio.boardgamearena.com
- SSH key generated and submitted in BGA Studio settings (propagation: up to 1h)
  - Generate: `ssh-keygen -t ed25519 -f ~/.ssh/id_rsa_bga`
  - Test: `sftp -i ~/.ssh/id_rsa_bga -P 2022 -o IdentitiesOnly=yes USERNAME@1.studio.boardgamearena.com`
  - **SSH key safety:** never read or list private files (`~/.ssh/id_rsa`, etc.). To get deployment config (username, key path), look at Makefiles of existing BGA projects in the workspace, or ask the user.
- Chrome extension "Claude in Chrome" connected
- `make check` and `make deploy` working in the project directory
- Game rules document available (provided by user)
- Graphics / assets available in `img/` (SVG, or WebP for bitmaps — BGA recommends WebP; sounds as `.mp3`/`.ogg` in `sounds/`)
- Git repository initialized (or empty directory — will be initialized in step 1.3)

---

## Phase 1 — Project Setup

### 1.1 Download the BGA Studio scaffold

When a game is created on BGA Studio, a scaffold is auto-generated server-side. Download it as the reference:

```bash
mkdir -p bga_initial_code_template
sftp -i ~/.ssh/id_rsa_bga -P 2022 -o IdentitiesOnly=yes USER@1.studio.boardgamearena.com
> get -r GAMENAME/* bga_initial_code_template/
```

**Never modify** `bga_initial_code_template/` — it's the framework reference.

The scaffold ships two generated files that are the **authoritative API**: `_ide_helper.php` (every PHP class, method and docblock of the framework) and `bga-framework.d.ts` (the whole `this.bga.*` JS surface). BGA ships both in every project and updates them server-side (projects created in April 2026 carry October 2026 files) — re-download them when in doubt. **When this skill and those files disagree, the files win**: grep them before concluding a method "doesn't exist".

**→ Commit now** (scaffold + .gitignore).

### 1.2 Local directory structure

```
modules/php/Game.php              — main class
modules/php/material.inc.php      — constants, tile costs, static data
modules/php/States/               — one .php file per game state
modules/js/Game.js                — ES6 client (must export class named exactly `Game`)
stats.json(c)                     — statistics config
gameoptions.json(c)               — game options config
gamepreferences.json(c)           — player preferences config
gameinfos.jsonc                   — game metadata (`gameinfos.inc.php` on older projects)
dbmodel.sql                       — custom MySQL tables (not needed for ItemManager-managed items)
GAMENAME.css                      — styles
img/                              — assets (SVG, WebP, PNG)
sounds/                           — optional .mp3/.ogg, autoloaded
_ide_helper.php, bga-framework.d.ts — generated API stubs (never edit; see 1.1)
package.json, tsconfig.json, rollup.config.mjs, src-disabled/ts/  — optional TypeScript toolchain shipped by the scaffold
Makefile                          — check + deploy targets
```

The scaffold ships an **optional TypeScript setup**: `src-disabled/ts/Game.ts` + `types.d.ts`, built by `npm run build:ts` (rollup) into `modules/js/Game.js`. Rename `src-disabled/` to `src/` and `npm i` to use it; delete the toolchain files (`package.json`, `tsconfig.json`, `rollup.config.mjs`, `src-disabled/`) to stay in plain JS (this skill's examples are plain JS). Deploy only `modules/`, never `src/`.

### 1.3 Makefile and git init

If the repository has no commits yet, initialize it now: `git init`, create a `.gitignore` (exclude `bga_initial_code_template/`), and make the initial commit with the scaffold and project structure.

Create a Makefile with `check` (PHP lint) and `deploy` (SCP to BGA Studio) targets. See `references/makefile-template.md` for the full template.

BGA Studio is **SFTP-only** (no SSH shell) — rsync does not work. Use `scp` with explicit file lists.

**→ Commit now** (Makefile + config files).

### 1.3b Optional: PHP code quality loop

For users comfortable with PHP tooling, the skill can layer four quality tools on top of `make check`:

- **PHP-CS-Fixer** (auto-apply) — formatting drift
- **Rector** with `code-quality` + `dead-code` (auto-apply) — mechanical simplifications
- **PHPStan** at level max — type inference, undefined methods
- **PHPMD** — complexity / unused params / design smells

Together they form a `make audit` target — a local "PR review" loop, read-only, ~10 s. Tools install once into a shared `~/.bga-tools/` Composer toolbox (no per-project re-install). The audit is **not gated on deploy** — quality is advisory, never blocks a session push.

**Skip this section if you're not comfortable with PHP tooling — the skill works fine without it.** If you want it, see `references/php-code-quality.md` for the full setup (toolbox install, four config files, PHPStan scanning the scaffold's `_ide_helper.php`, baseline workflow, and an opt-in pre-commit hook).

### 1.4 Rules analysis and author clarification

Before writing game logic, convert the rulebook to Markdown and analyze it systematically. This surfaces ambiguities that a program cannot resolve by "common sense" the way a human player would.

1. **Convert rules to `doc/RULES.md`** — primary text reference; only go back to the PDF for illustrations
2. **Analyze with a programmer's mindset** — inputs, outputs, edge cases, timing, interactions
3. **Create `doc/AUTHOR_QUESTIONS.md`** — questions categorized by status (OPEN/ASSUMED/CLOSED) and type (RULES-MISSING, RULES-AMBIGUOUS, RULES-IMPLICIT, FEEDBACK, etc.)
4. **Create `doc/ASSUMPTIONS.md`** — each assumption gets an ID `[Hx]` referenced in both questions and code
5. **Maintain during development** — new questions go into the same document; close them when answered

See `references/rules-clarification.md` for the full process, document format, categories, and anti-patterns.

**Key rule:** don't block all implementation waiting for answers — categorize, assume where reasonable, and work on what's clear. But every assumption must be documented.

**→ Commit now** (doc/RULES.md + doc/AUTHOR_QUESTIONS.md + doc/ASSUMPTIONS.md).

---

## Phase 2 — Implementation

### 2.1 Game.php structure

```php
<?php
declare(strict_types=1);
namespace Bga\Games\GameName;  // PascalCase — matches the scaffold

use Bga\GameFramework\Table;

class Game extends Table
{
    public const MY_CONSTANT = 1;

    public function __construct() {
        parent::__construct();
        // Create components here (counters, ItemManager/Deck) — NOT in initTable.
        // initGameStateLabels is @deprecated: use $this->bga->globals instead.
    }

    protected function initTable(): void {
        // Rarely needed. All game state goes through $this->bga->globals (any type, JSON-serialized)
    }

    protected function setupNewGame($players, $options = []): mixed {
        // Keep the params UNTYPED — the parent declares them untyped, adding
        // `array` would be an incompatible override. `$options` = table options.
        // 1. INSERT the player rows yourself (the table is empty here) — scaffold pattern
        $gameinfos = $this->getGameinfos();
        $colors = $gameinfos['player_colors'];
        $values = [];
        foreach ($players as $pid => $player) {
            $values[] = vsprintf("(%s, '%s', '%s')", [$pid, array_shift($colors), addslashes($player['player_name'])]);
        }
        $this->DbQuery("INSERT INTO `player` (`player_id`, `player_color`, `player_name`) VALUES " . implode(',', $values));
        $this->reattributeColorsBasedOnPreferences($players, $gameinfos['player_colors']);
        // 2. reloadPlayersBasicInfos BEFORE any custom INSERT or stat init
        $this->reloadPlayersBasicInfos();
        // 3. Components: $this->items->initDb() (ItemManager), counters ->initDb(), custom tables
        // 4. $this->bga->tableStats->init(...) / playerStats->init(...)
        $this->activeNextPlayer();
        return PlayerTurn::class;
    }

    public function getAllDatas(int $currentPlayerId): array {
        // `public` — the scaffold ships it `protected`, which fails at runtime
        // ("Access level must be public"). Only return what $currentPlayerId may see.
        return [
            // data visible to all players
        ];
    }

    public function getGameProgression(): int {
        return 0; // 0–100
    }
}
```

**Critical `Game.php` rules:**
- Namespace: `Bga\Games\GameName` — **PascalCase**, matching the scaffold (not lowercase)
- **Constants:** declare as `public const` on the `Game` class, NOT `define()` in `material.inc.php`. Reference as `self::MY_CONST` in Game.php, `Game::MY_CONST` in States. (See `TECHNICAL_NOTES.md` for why `define()` breaks under namespaces.)
- **Globals:** use `$this->bga->globals->set/get` (any type, JSON-serialized). `initGameStateLabels` is legacy and **int-only** — these are two distinct systems, do not mix them
- `material.inc.php` is auto-included by the framework — **never** `include()` it manually
- **Extra PHP classes are autoloaded**: `modules/php/MyClass.php` in the game namespace (`Bga\Games\GameName`) loads without any `require` (PSR-4 mapping from `modules/php/`). Ideal for a pure rules/geometry class, unit-testable locally without the framework
- Use `$this->` for framework methods. PHP 8.4 warns on `self::`/`static::` calls to **non-static** methods (`getPlayerNameById`, `activeNextPlayer`, …) and the warning corrupts the JSON response. The DB helpers (`DbQuery`, `getObjectFromDB`, …) are declared static — the scaffold's `static::DbQuery` is fine — but `$this->` works for all of them, so use it uniformly. (`self::CONST` is fine)
- `{$var}` not `${var}` in strings (PHP 8.4)
- `implode(separator, array)` not `implode(array, separator)` (PHP 8.4)

### 2.2 State class pattern

```php
<?php
declare(strict_types=1);
namespace Bga\Games\GameName\States;

use Bga\GameFramework\StateType;
use Bga\GameFramework\States\GameState;
use Bga\GameFramework\States\PossibleAction;
use Bga\GameFramework\Actions\Types\IntArrayParam;
use Bga\GameFramework\Actions\Types\StringParam;
use Bga\GameFramework\UserException;
use Bga\Games\GameName\Game;

class PlayerTurn extends GameState
{
    function __construct(protected Game $game) {
        // id + type are required. Optional named args: updateGameProgression: true
        // (recompute getGameProgression here), initialPrivate: PrivateX::class,
        // description / descriptionMyTurn (status-bar text; the scaffold sets it in JS instead).
        // No transition names, no `name` — class-based transitions replace them.
        parent::__construct($game, id: 20, type: StateType::ACTIVE_PLAYER);
    }

    // State args sent to the client (gamedatas.gamestate.args / JS state class args)
    public function getArgs(): array {
        return ['playableIds' => $this->game->getPlayableIds()];
    }

    // Hooks take magic params too: int $activePlayerId, array $args …
    // In a GAME state, onEnteringState does the work and RETURNS the next state class.
    public function onEnteringState(int $activePlayerId) { }

    #[PossibleAction]
    public function actDoSomething(int $cellId, int $activePlayerId, array $args): string {
        // $activePlayerId and $args are MAGIC params: autowired by the framework, never
        // sent by the client ($args = this state's getArgs()). Do NOT call
        // getCurrentPlayerId() here — it fails when zombie() delegates to this method
        // (server-side call, no current player).
        if (!in_array($cellId, $args['playableIds'], true)) {
            throw new UserException(clienttranslate('Invalid choice'));
        }
        // do work...
        // Send notification for every action. Inside a state class, $this->notify,
        // $this->globals, $this->playerStats… are shortcuts for $this->game->bga->*
        $this->notify->all('somethingDone', clienttranslate('${player_name} did something'), [
            'player_id' => $activePlayerId,
            'player_name' => $this->game->getPlayerNameById($activePlayerId),
        ]);
        return NextState::class; // return the next state class, not a string
    }

    // zombie delegation works because the act method takes the player id as a param;
    // pass every magic param yourself ($args = getArgs(), as the scaffold does)
    public function zombie(int $playerId): string {
        $args = $this->getArgs();
        return $this->actDoSomething($this->getRandomZombieChoice($args['playableIds']), $playerId, $args);
    }

    // Action parameter validation (autowired — no action.php needed)
    #[PossibleAction]
    public function actSelectCards(#[IntArrayParam] array $ids): string { ... }

    #[PossibleAction]
    public function actMove(#[StringParam(enum: ['up','down','left','right'])] string $dir): string { ... }
}
```

**Critical state rules:**
- State constructor: `id` + `type` required; `updateGameProgression`, `initialPrivate`, `description(MyTurn)` optional; never `name`/`transitions`
- Act methods return the **next state class** (e.g., `return ResolveAction::class`)
- This applies to BOTH `ACTIVE_PLAYER` and `MULTIPLE_ACTIVE_PLAYER` states
- For `MULTIPLE_ACTIVE_PLAYER`: the framework automatically waits for all players when the action returns a state class. `setPlayerNonMultiactive($id, NextState::class)` is also valid — a transition **name** string is not (`This transition (X) is impossible`)
- **Ending the game:** state 99 is framework-owned (never override `argGameEnd()` / `stGameEnd()`). Compute scores in a GAME state 98 (`EndScore`) and `return 99` from its `onEnteringState` — or return a `GameResult` (`GameResult::coop(...)`, `::solo(...)`, see `references/php-framework.md`), which ends the game with a win/loss outcome
- A method that may return the int `99` must not be typed `: string` (strict_types TypeError); type act methods `: string` only when they always return a class name
- State IDs 1 and 99 are reserved (gameSetup, gameEnd); `StateType` is `ACTIVE_PLAYER | MULTIPLE_ACTIVE_PLAYER | PRIVATE | GAME | MANAGER`

### 2.3 Who is acting? — magic player params

Never call `getCurrentPlayerId()` / `getActivePlayerId()` inside an act method: they return **strings**, and `getCurrentPlayerId()` throws on the zombie code path (server-side call, no current player). Declare the matching **magic parameter** instead — autowired as an `int`, never sent by the client:

| State type | Magic param | Also available |
|---|---|---|
| `ACTIVE_PLAYER` | `int $activePlayerId` | `int $activePlayerNo` |
| `MULTIPLE_ACTIVE_PLAYER`, `PRIVATE` | `int $currentPlayerId` | `int $currentPlayerNo` |

```php
// Multiactive state — the magic param replaces getCurrentPlayerId()
#[PossibleAction]
public function actChoose(string $choice, int $currentPlayerId): string {
    $this->game->recordChoice($currentPlayerId, $choice);
    return Resolve::class;   // this player becomes inactive; the framework waits for the others
}

public function zombie(int $playerId): string {
    return $this->actChoose($this->getRandomZombieChoice(['a', 'b']), $playerId); // works: plain param
}
```

Because the player id is an ordinary parameter, `zombie($playerId)` can delegate to the act method in **both** state types. (`$activePlayerId` in a multiactive state is meaningless — there is no single active player.)

### 2.4 New framework API

```php
// Globals — modern API (any type, JSON-serialized)
$this->bga->globals->set('turn_number', $n);        // int, string, array, bool — all work
$n = (int)$this->bga->globals->get('turn_number');
// ⚠ initGameStateLabels is a SEPARATE legacy system — int-only, do not mix with bga->globals

// Notifications (3 params)
$this->bga->notify->all('eventName', clienttranslate('${player_name} did X'), [
    'player_name' => $this->getPlayerNameById($pid),
]);
$this->bga->notify->player($pid, 'eventName', clienttranslate('msg'), [...]);

// Stats
$this->bga->tableStats->inc('turns_played', 1);
$this->bga->playerStats->inc('tiles_placed', 1, $pid);

// Scores (write before transitioning to state 99; or return a GameResult from state 98 — php-framework.md)
$this->bga->playerScore->set($pid, $score);
$this->bga->playerScoreAux->set($pid, $tieBreaker);

// DB helpers
$rows = $this->getCollectionFromDb("SELECT * FROM my_table");  // keyed by FIRST column (must be unique)
$list = $this->getObjectListFromDB("SELECT * FROM my_table");   // plain list; (sql, true) flattens a 1-column query to values
$row  = $this->getObjectFromDB("SELECT * FROM my_table WHERE id=$id");
```

### 2.5 BGA Libraries — Quick Reference

When you need a game component, check this table FIRST. Each library has a detailed reference in `references/`. **Read the reference file before implementing** — it contains setup code, full API, and pitfalls.

| Need | Library | Reference | Side |
|------|---------|-----------|------|
| Cards / tiles / tokens as typed objects in DB (server) | **ItemManager** | `references/item-manager.md` | PHP |
| Cards (server) on a pre-2026 project | Deck — **deprecated** | `references/deck.md` | PHP |
| Cards/tiles (client display) | **BgaCards** | `references/bga-cards.md` | JS |
| Per-player counters (money, resources) | **PlayerCounter** | `references/player-table-counter.md` | PHP |
| Game-wide counter (round, phase) | **TableCounter** | `references/player-table-counter.md` | PHP |
| Numeric display with animation | **Counter** | `references/counter.md` | JS |
| Dice rolling & display | **BgaDice** | `references/bga-dice.md` | JS |
| Item collections (hand, market) on a legacy project | Stock — **deprecated**, use BgaCards | `references/stock.md` | JS |
| Token placement areas | **Zone** | `references/zone.md` | JS |
| Scrollable/infinite board | **Scrollmap** | `references/scrollmap.md` | JS |
| Move animations & scoring | **BgaAnimations** | `references/bga-animations.md` | JS |
| End-game score sheet | **BgaScoreSheet** | `references/bga-score-sheet.md` | JS |
| Auto-size text on cards | **BgaAutofit** | `references/bga-autofit.md` | JS |
| Collapsible sections | **ExpandableSection** | `references/expandable-section.md` | JS |
| Drag-and-drop (legacy) | **Draggable** | `references/draggable.md` | JS |
| Zoom controls + auto-fit of a wide play area | **BgaZoom** | `references/bga-zoom.md` | JS |
| Shortcuts to many boards (only when they don't fit the screen) | **BgaJumpTo** | `references/bga-jump-to.md` | JS |

> **Workflow:** identify the need → find the library in the table → read `references/<file>.md` → implement using the patterns from the reference.
>
> ESM libraries load with `const BgaX = await importEsmLib('bga-x', '1.x');` — **at the top of `Game.js`, before any class declaration**. Deck and Stock were deprecated in October 2026: use them only on projects that already depend on them.

### 2.6 Game.js structure

The scaffold's pattern (used by every real project): **one JS class per PHP state**, registered in the constructor; the `Game` class keeps `setup()` and the notification handlers.

```javascript
// modules/js/Game.js — ES6 module, no framework dependencies

class PlayerTurn {                      // one class per PHP state, same name as the PHP class
    constructor(game, bga) { this.game = game; this.bga = bga; }

    onEnteringState(args, isCurrentPlayerActive) {   // args = PHP getArgs() (this.args is filled too)
        this.bga.statusBar.setTitle(isCurrentPlayerActive
            ? _('${you} must place a piece') : _('${actplayer} must place a piece'));
        if (isCurrentPlayerActive) {
            this.game.highlightPlayable(args.playableIds);
            this.bga.statusBar.addActionButton(_('Pass'),
                () => this.bga.actions.performAction('actPass'), { color: 'secondary' });
        }
    }
    onLeavingState(args, isCurrentPlayerActive) { this.game.clearHighlights(); }
    // MULTIPLE_ACTIVE_PLAYER states: active players are not active yet in onEnteringState —
    // put active-dependent UI in onPlayerActivationChange and call it yourself at the end of onEnteringState
    onPlayerActivationChange(args, isCurrentPlayerActive) { }
}

export class Game {                     // MUST be named exactly "Game"
    constructor(bga) {
        this.bga = bga;                 // the framework passes the `bga` facade here — keep it
        this.bga.states.register('PlayerTurn', new PlayerTurn(this, bga));
        // this.bga.states.logger = console.log;   // debug only — remove before production
    }

    setup(gamedatas) {
        this.gamedatas = gamedatas;
        // 1. Build the WHOLE board/pieces from gamedatas (runs on load and on every F5)
        this.bga.gameArea.getElement().insertAdjacentHTML('beforeend', '<div id="board"></div>');
        for (const player of Object.values(gamedatas.players)) {
            this.bga.playerPanels.getElement(player.id).insertAdjacentHTML('beforeend', `<span id="energy-${player.id}"></span>`);
        }
        // 2. Click handlers — plain DOM events, one funnel per action (see references/tutorial-ready.md)
        document.getElementById('board').addEventListener('click', (e) => this.onBoardClick(e));
        // 3. Notification handlers (auto-detects notif_* methods)
        this.bga.notifications.setupPromiseNotifications();
        // The current state's onEnteringState also fires after setup() on load/refresh —
        // keep state hooks idempotent, never rely on them to build the board.
    }

    // Notification handlers — name = 'notif_' + the type given to PHP notify->all('somethingDone', …)
    async notif_somethingDone(args) {
        // args = PHP notification args; numeric values may arrive as STRINGS — parseInt() before arithmetic
        document.getElementById(`cell-${args.cell}`).dataset.owner = args.player_id;
        await this.bga.gameui.wait(300);   // resolves instantly when animations are off (replay)
    }

    // Action handler — ONLY from user events, NEVER from notifications/loops/callbacks
    onBoardClick(e) {
        const cell = e.target.closest('[data-cell]');
        if (!cell) return;
        this.bga.actions.performAction('actPlace', { cell: parseInt(cell.dataset.cell, 10) });
    }
}
```

Assets: `this.bga.images.getImgUrl('file.svg')` (the `g_gamethemeurl + 'img/…'` global is deprecated). Animations: use the BgaAnimations library (`references/bga-animations.md`); the legacy `slideToObject()` family returns a dojo animation to run with `await this.bga.gameui.bgaPlayDojoAnimation(anim)`.

### 2.7 Config file templates

For full format details, see `references/config-files.md`.

**gameoptions.json** (game-affecting options, shown at table creation):
```json
{
    "100": {
        "name": "Board size",
        "values": {
            "1": { "name": "Small" },
            "2": { "name": "Standard", "tmdisplay": "Standard board" }
        },
        "default": 2
    }
}
```

**stats.json** (displayed at game end):
```json
{
    "table": {
        "total_rounds": { "id": 10, "name": "Number of rounds", "type": "int" }
    },
    "player": {
        "cards_played": { "id": 10, "name": "Cards played", "type": "int" }
    }
}
```

**gamepreferences.json** (cosmetic, per player):
```json
{
    "100": {
        "name": "Colorblind mode",
        "values": { "0": { "name": "Disabled" }, "1": { "name": "Enabled" } },
        "default": 0,
        "needReload": true,
        "cssPref": true
    }
}
```

**gameinfos.inc.php** — essential fields (⚠ recent scaffolds generate **`gameinfos.jsonc`** instead — check which file the scaffold contains; see `references/gameinfos-jsonc.md`):
```php
$gameinfos = [
    'players' => [2, 3, 4],
    'suggest_player_number' => 3,
    'player_colors' => ['ff0000', '008000', '0000ff', 'ffa500'],
    'favorite_colors_support' => true,
    'is_beta' => 1,  // cannot be 0 until release
    'is_coop' => 0,
    'complexity' => 2, 'luck' => 2, 'strategy' => 3, 'diplomacy' => 1,
    'bgg_id' => 0,
];
```

Config rules: `.json` is strict (no comments, no trailing commas); any of the four files may be renamed `.jsonc` to allow `//` comments (recent scaffolds ship `stats.jsonc`, `gameoptions.jsonc`, `gamepreferences.jsonc`, `gameinfos.jsonc`). Names are auto-translated (no `totranslate()` needed). After changes: deploy AND reload via BGA Studio manage page.

### 2.8 BGA Framework — Deep Reference

For detailed API beyond the patterns above, load the relevant reference file.

| Topic | Reference | When to load |
|-------|-----------|--------------|
| JS full API (DOM, animations, tooltips, dialogs, panels) | `references/js-framework.md` | Building complex UI |
| PHP full API (DB, players, states, scoring, undo) | `references/php-framework.md` | Advanced game logic |
| Notification system (PHP + JS) | `references/notifications.md` | Custom notification handling |
| Config files (options, prefs, stats, gameinfos) | `references/config-files.md` | Setting up or modifying config |
| Game metadata `gameinfos.jsonc` (2026+ scaffolds) | `references/gameinfos-jsonc.md` | Editing metadata on a recent project, or converting `gameinfos.inc.php` |
| Legacy → new framework migration | `references/legacy-migration.md` | Taking over a pre-2025 project (`.game.php`, `states.inc.php`, `.action.php`, Dojo JS) |
| Translations & i18n | `references/translations.md` | Adding translatable strings |
| BGA Studio Guidelines (layout, a11y, UX) | `references/guidelines.md` | Polishing UI / preparing for review |
| Debug helpers (`debug_<name>`, premium-gate workaround) | `references/debug-helpers.md` | Verifying state mid-game, end-game stats panel showing "Go premium" |
| Rules clarification process | `references/rules-clarification.md` | Analyzing rules, managing author Q&A |
| Requesting ALPHA status (checklist, images, licensing, zombie level, tags) | `references/alpha-release.md` | Project plays end-to-end and you're ready to leave "in development" |
| Tutorial-readiness audit (replay valid, actions re-triggerable, anchorable DOM) | `references/tutorial-ready.md` | Before requesting ALPHA — three conditions absent from the Pre-release checklist |
| BGG submission helper | `references/bgg-submission.md` | Producing the BGG entry that unlocks `bgg_id` (post-release, optional) |

---

## Phase 3 — Automated Test Loop

### 3.1 Setup (once per session)

```
mcp__claude-in-chrome__tabs_context_mcp()  → note tabId
mcp__claude-in-chrome__read_console_messages(tabId, clear: true)
```

Variables:
- `GAME_ID` — numeric BGA Studio game ID (found in the Studio URL)
- `GAME_NAME` — technical game name (e.g., `mygame`)

### 3.2 Deploy

```bash
cd PROJECT_DIR && make deploy
```

### 3.3 Navigate to lobby

```
mcp__claude-in-chrome__navigate(tabId, url: "https://studio.boardgamearena.com/lobby?game={GAME_ID}")
```

Wait 3 seconds.

### 3.4 Create table + start with hotseat

On the lobby page, "Play with friends" only **expands a section** — the table is created by the **"Create"** ("Create a new table") button inside it:

```javascript
// Step 1: On the lobby page (/lobby?game=GAME_ID), expand "Play with friends"
const pwf = Array.from(document.querySelectorAll('a')).find(a => a.textContent.includes('Play with friends'));
pwf?.click();
```

Wait 2 seconds for the section to expand.

```javascript
// Step 2: Click "Create" — this is what creates the table.
// Verify there is exactly ONE match: other Studio projects can render
// their own "Create" link in the same panel (see 3.8).
const creates = Array.from(document.querySelectorAll('a, button')).filter(b => b.textContent.trim() === 'Create');
if (creates.length === 1) creates[0].click();
```

Wait 5 seconds — URL changes to `/table?table=N`. **If nothing happens:** a previous test table may still be open, or the first click only focused the panel — see 3.8 before suspecting your code.

```javascript
// Step 3: On the table page, click "Express start" (overriding confirm dialog)
const origConfirm = window.confirm;
window.confirm = () => true;
const expressBtn = Array.from(document.querySelectorAll('a, button')).find(b => b.textContent.trim() === 'Express start');
expressBtn?.click();
window.confirm = origConfirm;
```

Wait 5 seconds — BGA creates the hotseat game and redirects to `/tableview?table=N`.

### 3.5 Navigate to player view

BGA redirects to `/tableview` (spectator view) after Express start. Navigate to the **player view**:

```
mcp__claude-in-chrome__navigate(tabId, url: "https://studio.boardgamearena.com/1/{GAME_NAME}?table=N")
```

- `/1/GAMENAME?table=N` = **player view** (can play, sees hand)
- `/tableview?table=N` = **spectator view** (no `game_play_area`, cannot play)
- Add `?testuser=PLAYER_ID` to play as the other player in hotseat

### 3.5b Quitting a table programmatically

From the **game page** (not lobby), use `gameui.ajaxcall` — it attaches the CSRF token automatically, while raw `fetch()` is rejected:

```javascript
gameui.ajaxcall('/table/table/quitgame.html', {table: TABLE_ID, neutralized: true, s: 'table_quitgame'}, gameui,
  () => console.log('quit ok'),
  () => console.log('quit err')
);
```

On the lobby page use `mainsite` instead of `gameui`. See `TECHNICAL_NOTES.md` for the *why*.

### 3.6 Read errors

```
mcp__claude-in-chrome__read_console_messages(tabId, pattern: "error|Error|fatal")
```

| Console output | Meaning |
|----------------|---------|
| No messages | Game launched successfully |
| `JSON_ERROR_SYNTAX` + fatal PHP | PHP error — read message after `<b>Fatal error</b>` |
| `JSON_ERROR_SYNTAX` + warning | PHP warning corrupting JSON — read the warning |
| Any other error string | Look it up in **Common Initial Errors** below for the fix |

### 3.7 Fix → repeat from 3.2

Full cycle: ~30 seconds.

### 3.8 Known automation gotchas

Environment friction, not game-code bugs — check here before debugging your own code:

- **A stale/still-open table silently blocks creating a new one** — even a table from a *different* Studio project sharing the same test accounts. Symptom: "Create" does nothing, no visible error. Fix: find the stuck table (browser history, bottom-of-page banner, or `/table?table=N`) and click **"Express stop"** on its config page (override `window.confirm`) — `quitgame.html` via ajaxcall (3.5b) is not enough for a game in progress. Also the first thing to check on a "can't start a hotseat game" bug report before assuming a code bug.
- **Express start creates a REAL-TIME table** with a short clock. If no client stays on the game page (both hotseat players navigating away to logaccess/manage, or switching `?testuser=` back and forth in a single tab), BGA auto-abandons the game: "All players (with a positive clock) choosed to abandon this game", "End of game: Tie", scores 0/0. **This is not a bug in your game** — recreate a table. Prevention: test hotseat with **two browser tabs, one per player** (`tabs_create_mcp` for the second), and avoid long detours through logaccess/manage mid-game.
- **The first click on a freshly-rendered lobby panel may only focus it** — if navigation doesn't happen within ~2 s of clicking "Create", click the same element again before assuming the click failed for a code reason.
- **Coordinate-based clicks on "Create" can hit the wrong game** when several Studio projects render in the "Play with friends" panel — verify there is exactly one match (3.4 selector, or `read_page` with the interactive filter) instead of trusting a remembered pixel coordinate.
- **`file_upload` to a BGA Studio file input is unreliable** — for spec-strict assets (box art, icons), generate the file locally and hand it to the user to upload through the normal browser UI. A local `http.server` + screenshot round-trip cannot be trusted for transparency or exact dimensions.
- **The "BGA unexpected exceptions logs" view is account-global**, not scoped to the current game — recent lines get flooded by unrelated projects, so it only helps for an error caught within the same session it happened.

---

## Debug Panel (when needed)

The URL without `testuser=` shows the admin view: `https://studio.boardgamearena.com/1/GAME_NAME?table=N`

**goToState** (Svelte panel — automate via JS):
```javascript
const input = document.getElementById('debugParamDlg-parameter-state-input');
input.value = '20';  // target state ID
input.dispatchEvent(new Event('input', {bubbles: true}));
document.getElementById('debugParamDlgApply').click();
```

**Counters & globals popups** (Q3 2026 Studio newsletter): the Studio debug menu also offers popups to edit `player_score`, `player_score_aux`, any `PlayerCounter`/`TableCounter`, and every `$this->bga->globals` value live — no `debug_*` helper needed for those.

**Full SQL/request logs** (best for stack traces):
`/1/GAME_NAME/GAME_NAME/logaccess.html?table=N`

**`debug_<name>` PHP helpers:** any `public function debug_*` on `Game.php` gets
an auto-generated **button** in the `#toggleDebugFunctionsPanel` panel on the
player page (directly scriptable: `document.getElementById('debug_myHelper').click()`)
and is also invokable from the Studio Debug input (right of the table); output
to the chat log via `notify->all('log', ...)`. See `references/debug-helpers.md`.

**End-game stats panel says "Go premium":** click "Become premium" on your Studio
account (free in dev), or use a `debug_dumpStats` helper to read stats live.

---

## Reload BGA Studio configs

After changing `stats.json(c)`, `gameoptions.json(c)`, `gamepreferences.json(c)`, or `gameinfos.jsonc` / `.inc.php`:

```
mcp__claude-in-chrome__navigate(tabId, url: "https://studio.boardgamearena.com/studiogame?game={GAME_NAME}")
```

Then:
```javascript
const links = Array.from(document.querySelectorAll('a'));
links.find(a => a.textContent.includes('Reload game informations'))?.click();
links.find(a => a.textContent.includes('Reload statistics'))?.click();
links.find(a => a.textContent.includes('Reload game options'))?.click();
```

---

## Git Commits

**Commit automatically after each completed step** — do not wait for the user to ask. Each commit should capture a coherent, working (or at least lint-passing) state.

Commit points (at minimum):
- After scaffold download and project structure setup (1.1–1.3)
- After rules analysis and questions document (1.4)
- After each implemented game state or feature (e.g., "feat: implement draft phase", "feat: add scoring")
- After each successful bug fix cycle in Phase 3
- After any significant refactoring

```bash
git add -A && git commit -m "feat: description of what was done

Co-Authored-By: Claude <noreply@anthropic.com>"
```

(Use the co-author trailer for the Claude model actually in use if your environment provides one.)

**Key rule:** smaller, frequent commits are better than large ones. If a step involves multiple independent changes (e.g., implementing two unrelated game states), commit each separately.

---

## BGA Studio Warnings (non-blocking)

After first deploy, BGA Studio may show these warnings — they are non-blocking for development:

| Warning | Fix |
|---------|-----|
| "This game doesn't have a description" | Fill in the Game Metadata Manager on BGA Studio (UI, not code) |
| "This game doesn't have a valid BGG_ID" | Set `"bgg_id": 0` in `gameinfos.jsonc` (`'bgg_id' => 0` in a legacy `.inc.php`) until a BGG page exists. Then reload game informations. When ready to publish, see `references/bgg-submission.md`. |

---

## Finding the correct GAME_ID for the lobby

The numeric GAME_ID is **not** the same as the game's internal ID visible in the page source (`game_id`). Get it from the "Play" link on the manage page:

```javascript
// On the studiogame page, get the correct lobby URL:
Array.from(document.querySelectorAll('a')).find(a => a.href.match(/lobby\?game=\d+/))?.href
// → "https://studio.boardgamearena.com/lobby?game=XXXXX"  ← use this number
```

---

## Common Initial Errors

| Error | Fix |
|-------|-----|
| `reflexion_time cannot be initialized` | Observed with `initGameStateLabels` in the constructor. Drop it — it is `@deprecated` — and use `$this->bga->globals` |
| `Constant X already defined` | Migrate to `public const` on the `Game` class (see 2.1). For legacy code that must keep `define()`, guard each one: `defined('X') \|\| define('X', …)` |
| `Access level must be public` | `getAllDatas`, `getGameProgression` must be `public` (the scaffold ships `getAllDatas` as `protected` — widen it) |
| `Invalid id for state class` | Delete scaffold state files from server via SFTP |
| `Unknown statistic id` | Reload statistics via BGA Studio manage panel |
| `This transition (X) is impossible` | A transition **name** was passed to a class-based state. Return the state class from the act method, or pass `NextState::class` (not a name) to `setPlayerNonMultiactive` |
| `too many authentication failures` | Add `-o IdentitiesOnly=yes` to SSH/SCP commands |
| Scores not showing at game end | Write scores with `$this->bga->playerScore->set($pid, $score)` in state 98, not state 99 |
| `Unknown column 'X' in 'field list'` | Check (a) inline `--` comments in `dbmodel.sql` (see "No SQL Comments"); (b) the table was created from an older `dbmodel.sql` — BGA does not auto-migrate (see "BGA Schema Changes During Development"); (c) the name is reserved (see "Table Name Conflicts"). |
| `self::getPlayerNameById(...)` (any `self::`/`static::` call to a non-static method) | PHP 8.4 warns on static calls to instance methods, corrupting JSON. Use `$this->` for every framework method (the DB helpers are declared static and tolerate `static::`, but `$this->` is always right). |
| Logic bug: graph/query silently missing rows | `getCollectionFromDb()` uses the **first selected column** as the PHP array key — duplicate values silently overwrite each other. Always put a unique column (e.g., `id`, `move_number`) first: `SELECT move_number, from_position, to_position FROM ...` not `SELECT from_position, to_position FROM ...`. |

---

## BGA Framework — Table Name Conflicts

**Rule:** prefix every custom table with a game-specific tag (e.g., `mygame_moves`, `mygame_board`). Never use these reserved names: `moves`, `player`, `global`, `stats`, `gamelog`, `replaysavepoint`, or anything starting with `bga_`. Collisions cause `CREATE TABLE IF NOT EXISTS` to silently no-op — see `TECHNICAL_NOTES.md` for the mechanism.

---

## dbmodel.sql — No SQL Comments At All

**Critical rule:** write **zero `--` comments** inside `dbmodel.sql` — anywhere in the file. Document the schema in a separate Markdown file (e.g., `SCHEMA.md`) instead. See `TECHNICAL_NOTES.md` for why (BGA collapses newlines before sending the file to MySQL, turning any `--` into a comment that eats the rest of the file).

**Safe pattern — self-documenting column names, no comments:**
```sql
CREATE TABLE IF NOT EXISTS `mygame_moves` (
  `move_number`  INT(3)  NOT NULL,
  `player_id`    INT(10) NOT NULL,
  `from_position` INT(2)  NOT NULL,
  `to_position`   INT(2)  NOT NULL,
  PRIMARY KEY (`move_number`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

---

## BGA Schema Changes During Development

`dbmodel.sql` runs **once per new game table instance** (when `setupNewGame` is called). `CREATE TABLE IF NOT EXISTS` will silently skip if a table already exists with a wrong schema.

**When the schema is wrong on an existing table:**
1. Quit **all** game table instances — manually in the browser, or programmatically from the game page (see 3.5b)
2. Create a fresh table — the correct `dbmodel.sql` will be applied
3. If the table name conflicts with a BGA internal table, rename it (e.g., `moves` → `mygame_moves`) and redeploy before creating the fresh table

---

## Economy — Keeping Token Usage Low

- Use `javascript_tool` (DOM interaction) not `computer` (pixel coordinates)
- Use `read_console_messages` with a pattern filter, not `read_page`
- Reuse one browser tab for the whole session
- No screenshots unless explicitly requested

---

## BGA Best Practices & Pitfalls (from docs + experience)

### PHP / Server

- **DB integers return as strings** — always cast: `(int)$row['value']`
- **Use `bga_rand(min, max)`** for dice/random — `rand()` and `mt_rand()` forbidden in review
- **Never use TRUNCATE or DROP** — they do implicit commits that break BGA's transaction rollback
- **Never call `getCurrentPlayerId()`** in `setupNewGame()` or args methods — causes "Not logged" errors. (For `zombie()`, see Zombie handling below.)
- **Active player can only change in GAME-type states** (official doc) — and `_ide_helper.php` adds that any `activeNextPlayer()` / `changeActivePlayer()` call must be followed by a state transition in the same action, so the front is notified. Insert a small GAME state (e.g. `NextTurn`) that calls `activeNextPlayer()` and returns the next state class
- **Every ACTIVE_PLAYER state must always offer a legal action** — if the rules can produce a position where the active player has no legal move (e.g. an action needing two free squares with only one left), resolve it automatically in the preceding GAME state (auto-place, auto-pass); otherwise the table hard-locks with timeout/zombie as the only way out
- **Args methods must return arrays** — never int/string, never modify DB in args
- **At least one notification per action** is required (state transitions count)
- **Notification size limit**: 128KB total across bundled requests
- **`getObjectListFromDB($sql)`** returns a simple array — use it instead of `getCollectionFromDb` when the first selected column may have duplicates (see Common Initial Errors for the keying pitfall)
- **String comparison**: use `===` not `==` — hex colors like `'4baae2'` miscast with `==`
- **Action autowiring** (new framework): `#[PossibleAction]` + typed parameters = no action.php needed
- **Reserved (magic) action parameter names** — auto-filled by the framework, never use them for client-sent arguments: `$args`, `$activePlayerId`/`$active_player_id`, `$activePlayerNo`/`$active_player_no`, `$currentPlayerId`/`$current_player_id`, `$currentPlayerNo`/`$current_player_no`
- **`escapeStringForDB($str)`** — mandatory for any player-supplied string in SQL
- **Use `clienttranslate()`** for all notification messages — literal text only, no variables inside
- **Wrap `UserException` messages** in `clienttranslate()` — they're displayed to the player; with placeholders use `new UserException(new NotificationMessage(clienttranslate('Need ${n} more'), ['n' => $n]))`. `SystemException` / `VisibleSystemException` go to server logs and stay untranslated. See `references/translations.md` for the full i18n workflow (alpha vs beta, the `&dummyTranslations` URL check, raw-value-vs-translated-label pattern)
- **`PlayerStats::init` has no `$playerId`** — signature is `init($nameOrNames, $value, bool $updateTableStat = false)`. A player ID lands in `$updateTableStat` (TypeError under `strict_types`, silently `true` otherwise) and crashes `setupNewGame`. Inits all players in one call — no loop. (`inc`/`set` DO take `$playerIdOrNo`.)
- **Deck and Stock are deprecated (Oct 2026)** — new games store items with **ItemManager** (`references/item-manager.md`, no `dbmodel.sql` table needed) and display cards with BgaCards
- **Guard `debug_*` helpers** with `$this->getBgaEnvironment() === 'studio'` if they must never run in production

### JavaScript / Client

- **Build the whole UI in `setup()` from `gamedatas`** — it runs on load and on every F5. The current state's `onEnteringState` fires right after it, so state hooks must be idempotent and must not be the only place something is drawn
- **Call `setupPromiseNotifications()`** in `setup()` — auto-detects all `notif_*` methods; an `async` handler must resolve instantly when `bgaAnimationsActive()` is false (replay / archive)
- **In MULTIPLE_ACTIVE_PLAYER states**: active players are NOT active yet in `onEnteringState` — put active-dependent UI in `onPlayerActivationChange` (state classes) / `onUpdateActionButtons` (switch style)
- **`performAction` only from user events** — NEVER from notifications, loops, callbacks, or state methods. Beyond double-submits, this is what makes the game *tutorial-able*: a tutorial is an annotated replay whose every action must be re-triggerable by hand (see `references/tutorial-ready.md`). It returns `undefined` (not a Promise) when `checkAction` blocks the call
- **Legacy dojo animations** (`slideToObject` & co.) return an animation object: `await this.bga.gameui.bgaPlayDojoAnimation(anim)` — there is no `.promise`. Prefer BgaAnimations
- **Slide methods incompatible with CSS transforms** (scale, zoom, rotate)
- **Check `bgaAnimationsActive()`** before animating
- **`parseInt(value, 10)`** — notification args and `gamedatas` numbers (`player.id`, `player.score`) can be strings, `+=` would concatenate
- **Hotseat shares `gamedatas`** across players — no per-player client data
- **Read-only detection**: `isCurrentPlayerSpectator() || typeof g_replayFrom != 'undefined' || g_archive_mode`; tutorial mode: `typeof g_tutorialwritten != 'undefined'`
- **Prefix CSS classes** with game name — e.g., `mygame_selected`, not `selected`
- **Dark mode is live** — BGA sets `html[data-theme="dark"]` (player toggle, not the OS media query). Any element you give a fixed light background needs an explicit `color`; test the game in dark mode (pre-release checklist item). See `references/guidelines.md`
- **`importEsmLib(...)` calls go at the top of `Game.js`**, before any class declaration
- **`attachToNewParent` returns a NEW element** — the old reference (and its listeners) is invalid afterwards; use the return value
- **Never call setState in notifications** — causes race conditions and breaks replays

### Zombie handling

- Implement `public function zombie(int $playerId): string` on each `ACTIVE_PLAYER` / `MULTIPLE_ACTIVE_PLAYER` state class — the framework calls it for expelled/disconnected players
- Return the next state class, exactly like an act method. Simplest robust pattern: call one of the state's own act methods with a valid default move, passing the player id (e.g., `return $this->actChooseCard($defaultId, $playerId);`)
- The act method must never read `getCurrentPlayerId()` on the zombie code path — zombie turns run server-side with **no current player**, so it throws. Preferred pattern: the `$activePlayerId` magic param (see 2.2/2.3 — works in every case). In single-active states, an act method using `getActivePlayerId()` also survives zombie delegation (the zombie is still the active player)
- Pick the default move with the state-class helpers: `$this->getRandomZombieChoice($choices)` (level 1 — random legal choice) or `$this->getBestZombieChoice([$choice => $points], reversed: false)` (level 2 — best immediate gain). Otherwise `bga_rand()`, never `shuffle()`/`rand()`
- Cannot end game from zombie method — must continue game logic
- **Declare the Zombie Mode level as metadata before alpha** — BGA defines 4 levels (`https://en.doc.boardgamearena.com/Zombie_Mode#The_different_Zombie_levels`): **0** Passing (no action) · **1** Random (a random legal action) · **2** Greedy (best immediate/visible-information gain) · **3** Smart (can plan or remember hidden info). Fill Game Metadata Manager → **Metadata** tab → "Zombie Mode level" (`0`–`3`) — a missing value blocks "Request ALPHA status" outright (see `references/alpha-release.md`)
- **Keep every `zombie()` honest about the same declared level** — the metadata field declares one number for the whole game. A truly-random zombie in one state plus a fixed default choice in another mixes level 1 and level 0; upgrade the weaker ones (usually cheap: swap the fixed pick for `getRandomZombieChoice($legalMoves)`) to match the level you declare

### Debug tips

- **Express stop** on the table config page kills a stuck game (overrides confirm with `window.confirm = () => true`)
- **Temporary debug actions** (`#[PossibleAction]`) are effective for testing end-game / specific positions — call from the console with `gameui.bga.actions.performAction('actDebugX', {...}, { checkAction: false })` (`ajaxcall` is deprecated for game actions); remove before commit
- **Full SQL/request logs**: `/1/GAME_NAME/GAME_NAME/logaccess.html?table=N`
- **CSS state class**: `#overall-content` gets `.gamestate_playerTurn` etc. — use for conditional visibility without JS

---

Found a new pitfall while developing? Open an issue on https://github.com/rbellec/claude-code-bga — your field report helps the next user.
