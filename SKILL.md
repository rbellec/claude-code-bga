---
name: bga-alpha
description: Implement a board game alpha on Board Game Arena (BGA) Studio, from rules+graphics to a playable hotseat game. Covers project setup, new framework (2025+) architecture, PHP state machine, JS client, and the automated deploy→test loop.
disable-model-invocation: false
---

# BGA Alpha — From Rules to Playable Game

You are helping implement a board game on Board Game Arena Studio, using the **new BGA framework (2025+)**. The workflow has three phases: **Setup**, **Implementation**, and **Test Loop**.

---

## Prerequisites (verify before starting)

- BGA Studio account created, game registered at https://studio.boardgamearena.com
- SSH key generated and submitted in BGA Studio settings (propagation: up to 1h)
  - Generate: `ssh-keygen -t ed25519 -f ~/.ssh/id_rsa_bga`
  - Test: `sftp -i ~/.ssh/id_rsa -P 2022 -o IdentitiesOnly=yes USERNAME@1.studio.boardgamearena.com`
- Chrome extension "Claude in Chrome" connected
- `make check` and `make deploy` working in the project directory
- Game rules document available (provided by user)
- Graphics / assets available in `img/` (SVG preferred)
- Git repository initialized (commits will be made at each milestone)

---

## Phase 1 — Project Setup

### 1.1 Download the BGA Studio scaffold

When a game is created on BGA Studio, a scaffold is auto-generated server-side. Download it as the reference:

```bash
mkdir -p bga_initial_code_template
sftp -i ~/.ssh/id_rsa -P 2022 -o IdentitiesOnly=yes USER@1.studio.boardgamearena.com
> get -r GAMENAME/* bga_initial_code_template/
```

**Never modify** `bga_initial_code_template/` — it's the framework reference.

### 1.2 Local directory structure

```
modules/php/Game.php              — main class
modules/php/material.inc.php      — constants, tile costs, static data
modules/php/States/               — one .php file per game state
modules/js/Game.js                — ES6 client (must export class named exactly `Game`)
stats.json                        — statistics config
gameoptions.json                  — game options config
gamepreferences.json              — player preferences config
gameinfos.inc.php                 — game metadata
dbmodel.sql                       — custom MySQL tables
GAMENAME.css                      — styles
img/                              — assets (SVG, PNG)
Makefile                          — check + deploy targets
```

### 1.3 Makefile template

```makefile
BGA_USER   := USER
BGA_HOST   := 1.studio.boardgamearena.com
BGA_PORT   := 2022
BGA_GAME   := GAMENAME
BGA_REMOTE := $(BGA_GAME)
BGA_SCP    := scp -i ~/.ssh/id_rsa -P $(BGA_PORT) -o IdentitiesOnly=yes

DEPLOY_ROOT := gameinfos.inc.php dbmodel.sql stats.json gameoptions.json gamepreferences.json
DEPLOY_PHP  := modules/php/Game.php modules/php/material.inc.php $(wildcard modules/php/States/*.php)
DEPLOY_JS   := modules/js/Game.js

check:
	@php -l modules/php/Game.php
	@php -l modules/php/material.inc.php
	@for f in modules/php/States/*.php; do php -l "$$f" || exit 1; done
	@php -l gameinfos.inc.php
	@echo "✓ PHP OK"

deploy: check
	$(BGA_SCP) $(DEPLOY_ROOT) $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/
	$(BGA_SCP) GAMENAME.css $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/GAMENAME.css
	$(BGA_SCP) $(DEPLOY_PHP) $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/modules/php/
	$(BGA_SCP) modules/php/States/*.php $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/modules/php/States/
	$(BGA_SCP) $(DEPLOY_JS) $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/modules/js/
	$(BGA_SCP) img/* $(BGA_USER)@$(BGA_HOST):$(BGA_REMOTE)/img/
	@echo "✓ Deployed"
```

BGA Studio is **SFTP-only** (no SSH shell) — rsync does not work. Use `scp` with explicit file lists.

---

## Phase 2 — Implementation

### 2.1 Game.php structure

```php
<?php
declare(strict_types=1);
namespace Bga\Games\GAMENAME;

use Bga\GameFramework\Table;

class Game extends Table
{
    public function __construct() {
        parent::__construct();
        // Minimal — do NOT call initGameStateLabels here
    }

    protected function initTable(): void {
        // Put initGameStateLabels here, NOT in constructor
        $this->initGameStateLabels([
            'turn_number' => 10,
        ]);
    }

    public function setupNewGame(array $players, array $options = []): mixed {
        // 1. Set player colors
        $colors = $this->getGameinfos()['player_colors'];
        foreach ($players as $pid => $player) {
            $color = array_shift($colors);
            $this->DbQuery("UPDATE player SET player_color='$color' WHERE player_id='$pid'");
        }
        // 2. reloadPlayersBasicInfos BEFORE any INSERT or initStat
        $this->reloadPlayersBasicInfos();
        // 3. Custom tables
        // 4. initStat
        return StartState::class;
    }

    public function getAllDatas(int $currentPlayerId): array {
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
- Namespace: `Bga\Games\GAMENAME` (lowercase, exact game name)
- `initGameStateLabels` goes in `initTable()`, NOT constructor
- `material.inc.php` is auto-included by the framework — **never** `include()` it manually
- Guard all `define()`: `defined('X') || define('X', val)`
- No `self::` on instance methods — PHP 8.4 generates warnings that corrupt JSON output. Use `$this->` everywhere
- `{$var}` not `${var}` in strings (PHP 8.4)
- `implode(separator, array)` not `implode(array, separator)` (PHP 8.4)

### 2.2 State class pattern

```php
<?php
declare(strict_types=1);
namespace Bga\Games\GAMENAME\States;

use Bga\GameFramework\StateType;
use Bga\GameFramework\States\GameState;
use Bga\GameFramework\States\PossibleAction;
use Bga\GameFramework\UserException;
use Bga\Games\GAMENAME\Game;

class PlayerTurn extends GameState
{
    function __construct(protected Game $game) {
        // Only id and type — NO name, description, or transitions
        parent::__construct($game, id: 20, type: StateType::ACTIVE_PLAYER);
    }

    public function onEnteringState(array $args): void { }

    #[PossibleAction]
    public function actDoSomething(string $param): string {
        $playerId = $this->game->getCurrentPlayerId();
        // validate...
        // do work...
        return NextState::class; // return the next state class, not a string
    }
}
```

**Critical state rules:**
- State constructor takes ONLY `id` and `type` — no name, description, or transitions
- Act methods return the **next state class** (e.g., `return ResolveEyes::class`)
- This applies to BOTH `ACTIVE_PLAYER` and `MULTIPLE_ACTIVE_PLAYER` states
- For `MULTIPLE_ACTIVE_PLAYER`: the framework automatically waits for all players when the action returns a state class
- **Never** use `setPlayerNonMultiactive($id, 'transitionName')` — transition names don't exist in the new framework
- **Never** override `argGameEnd()` or `stGameEnd()` — they are `final`. Use a state 98 (`computeScores`) instead
- State IDs 1 and 99 are reserved (gameSetup, gameEnd)

### 2.3 Multiactive state — getCurrentPlayerId

In `MULTIPLE_ACTIVE_PLAYER` actions, the client does **not** send `activePlayerId`. Do not add it as a PHP parameter:

```php
// ✗ Wrong — client never sends this parameter
public function actChoose(int $activePlayerId, string $choice): string { ... }

// ✓ Correct
public function actChoose(string $choice): string {
    $playerId = $this->game->getCurrentPlayerId(); // framework provides it
    ...
    return NextState::class;
}
```

### 2.4 New framework API

```php
// Globals (replaces setGameStateValue/getGameStateValue)
$this->bga->globals->set('turn_number', $n);
$n = (int)$this->bga->globals->get('turn_number');

// Notifications (3 params)
$this->bga->notify->all('eventName', clienttranslate('${player_name} did X'), [
    'player_name' => $this->getPlayerNameById($pid),
]);
$this->bga->notify->player($pid, 'eventName', clienttranslate('msg'), [...]);

// Stats
$this->bga->tableStats->inc('turns_played', 1);
$this->bga->playerStats->inc('tiles_placed', 1, $pid);

// Scores (write before transitioning to state 99)
$this->bga->playerScore->set($pid, $score);

// DB helpers
$rows = $this->getCollectionFromDb("SELECT * FROM my_table");  // not getObjectListFromDB(sql, true) — broken
$row  = $this->getObjectFromDB("SELECT * FROM my_table WHERE id=$id");
```

### 2.5 Game.js structure

```javascript
// modules/js/Game.js — ES6, no framework dependencies
export class Game {  // MUST be named exactly "Game"
    constructor() {
        // called once when the game loads
    }

    setup(gamedatas) {
        // initial board rendering
    }

    onEnteringState(stateName, args) {
        // show/hide UI per state
    }

    onLeavingState(stateName) { }

    onUpdateActionButtons(stateName, args) {
        // add action buttons via gameui.addActionButton(...)
    }

    // Action handler
    _onMyAction() {
        bga.actions.performAction('actDoSomething', { param: 'value' });
    }
}
```

Assets: reference as `g_gamethemeurl + 'img/file.svg'`

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

### 3.4 Create table

```javascript
document.getElementById('joingame_create_{GAME_ID}').click()
```

Wait 5 seconds. URL should change to `/table?table=N`.

**If table already exists**: first quit via `document.getElementById('quit_game')?.click()`, wait 3 seconds, then retry.

### 3.5 Add hotseat player

```javascript
// Step a: open invite panel
document.getElementById('invite_friend').click()
// wait 2s
// Step b: select hotseat slot
document.getElementById('player_select_-99').click()
// wait 2s
// Step c: name and confirm
const i = document.getElementById('hotseat_player_name');
const b = document.getElementById('select_hotseat_name_btn');
if (i && b) { i.value = 'Player2'; i.dispatchEvent(new Event('input', {bubbles:true})); b.click(); }
```

Wait 10 seconds — BGA creates the game and runs `setupNewGame`.

### 3.6 Read errors

```
mcp__claude-in-chrome__read_console_messages(tabId, pattern: "error|Error|fatal")
```

| Console output | Meaning |
|----------------|---------|
| No messages | Game launched successfully |
| `JSON_ERROR_SYNTAX` + fatal PHP | PHP error — read message after `<b>Fatal error</b>` |
| `JSON_ERROR_SYNTAX` + warning | PHP warning corrupting JSON — read the warning |
| `reflexion_time cannot be initialized` | `initGameStateLabels` in constructor, or double `include()` of `material.inc.php` |
| `This transition (X) is impossible` | Used `setPlayerNonMultiactive(id, 'name')` — return state class instead |
| `Unknown statistic id` | Reload statistics via Manage Game panel |
| `Invalid id for state class` | Scaffold state files still on server — delete them via SFTP |

### 3.7 Fix → repeat from 3.2

Full cycle: ~30 seconds.

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

**Full SQL/request logs** (best for stack traces):
`/1/GAME_NAME/GAME_NAME/logaccess.html?table=N`

---

## Reload BGA Studio configs

After changing `stats.json`, `gameoptions.json`, `gamepreferences.json`, or `gameinfos.inc.php`:

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

Make a commit at each significant milestone:
- After scaffold download (step 1.1)
- After first successful `make deploy`
- After each bug fix cycle

```bash
git add -A && git commit -m "feat: description of what was done

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

---

## BGA Studio Warnings (non-blocking)

After first deploy, BGA Studio may show these warnings — they are non-blocking for development:

| Warning | Fix |
|---------|-----|
| "This game doesn't have a description" | Fill in the Game Metadata Manager on BGA Studio (UI, not code) |
| "This game doesn't have a valid BGG_ID" | Set `'bgg_id' => 0` in `gameinfos.inc.php` if no BGG page exists yet. Then reload game informations. |

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
| `reflexion_time cannot be initialized` | Move `initGameStateLabels` from constructor to `initTable()` |
| `Constant X already defined` | Add `defined('X') \|\|` guard before every `define()` in `material.inc.php` |
| `Access level must be public` | `getAllDatas`, `getGameProgression` must be `public` |
| `Invalid id for state class` | Delete scaffold state files from server via SFTP |
| `Unknown statistic id` | Reload statistics via BGA Studio manage panel |
| `This transition (X) is impossible` | Return state class from act method instead of calling `setPlayerNonMultiactive(id, 'name')` |
| `too many authentication failures` | Add `-o IdentitiesOnly=yes` to SSH/SCP commands |
| Scores not showing at game end | Write scores with `$this->bga->playerScore->set($pid, $score)` in state 98, not state 99 |
| `Unknown column 'X' in 'field list'` | The table was created from an older `dbmodel.sql`. BGA does NOT auto-migrate existing tables. Fix: rename the conflicting table (see below), then quit all game tables and create a fresh one. |
| `Unknown column 'square1'` on a `moves` table | BGA may auto-create a `moves` table internally. **Never name a custom table `moves`** — use a game-specific name like `q_moves`, `entanglements`, etc. |
| `static::DbQuery` or `self::DbQuery` | PHP 8.4 warns on static calls to instance methods, corrupting JSON. Always use `$this->DbQuery(...)`. Same applies to all other DB helpers. |
| Logic bug: graph/query silently missing rows | `getCollectionFromDb()` uses the **first selected column** as the PHP array key — duplicate values silently overwrite each other. Always put a unique column (e.g., `id`, `move_number`) first: `SELECT move_number, square1, square2 FROM ...` not `SELECT square1, square2 FROM ...`. |

---

## BGA Framework — Table Name Conflicts

BGA automatically creates several internal tables for each game instance. **Avoid these names** in `dbmodel.sql`:

| Reserved / risky name | Reason |
|----------------------|--------|
| `moves` | BGA may create a `moves` table internally with different columns |
| `player` | Standard BGA table — already exists |
| `global` | Standard BGA table |
| `stats` | Standard BGA table |
| `gamelog` | Standard BGA table |

**Rule:** Always prefix custom tables with a game-specific prefix (e.g., `q_moves`, `qttt_board`) or use unambiguous names that couldn't conflict with BGA internals.

---

## dbmodel.sql — No SQL Comments At All

**Critical rule:** Write **zero SQL comments** inside `dbmodel.sql` — no `--` comments anywhere in the file, not before statements, not between columns, not after the semicolon.

BGA strips newlines from `dbmodel.sql` before executing SQL. Any `--` comment, even on its own line, becomes an end-of-line comment that silently truncates everything that follows it on the collapsed single line.

**Safe pattern — no comments, self-documenting column names only:**
```sql
CREATE TABLE IF NOT EXISTS `q_moves` (
  `move_number`  INT(3)  NOT NULL,
  `player_id`    INT(10) NOT NULL,
  `square1`      INT(2)  NOT NULL,
  `square2`      INT(2)  NOT NULL,
  `collapsed_to` INT(2)  DEFAULT NULL,
  PRIMARY KEY (`move_number`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

If you need to document the schema, do it in a separate Markdown file (e.g., `SCHEMA.md`) — never inside `dbmodel.sql`. See `TECHNICAL_NOTES.md` for the full explanation.

---

## BGA Schema Changes During Development

`dbmodel.sql` runs **once per new game table instance** (when `setupNewGame` is called). `CREATE TABLE IF NOT EXISTS` will silently skip if a table already exists with a wrong schema.

**When the schema is wrong on an existing table:**
1. Quit **all** game table instances from the Studio lobby (click the Quit button manually in the browser — programmatic quit via JS injection fails due to BGA's Svelte/CSRF session handling)
2. Create a fresh table — the correct `dbmodel.sql` will be applied
3. If the table name conflicts with a BGA internal table, rename it (e.g., `moves` → `q_moves`) and redeploy before creating the fresh table

**Cannot quit tables programmatically** from injected JS — BGA's CSRF token for lobby actions is bound to the Svelte session and cannot be replicated via `dojo.xhrPost` or `fetch`. The user must click Quit manually in the browser.

---

## Economy — Keeping Token Usage Low

- Use `javascript_tool` (DOM interaction) not `computer` (pixel coordinates)
- Use `read_console_messages` with a pattern filter, not `read_page`
- Reuse one browser tab for the whole session
- No screenshots unless explicitly requested
