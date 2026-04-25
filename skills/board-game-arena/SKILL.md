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
  - Test: `sftp -i ~/.ssh/id_rsa -P 2022 -o IdentitiesOnly=yes USERNAME@1.studio.boardgamearena.com`
  - **SSH key safety:** never read or list private files (`~/.ssh/id_rsa`, etc.). To get deployment config (username, key path), look at Makefiles of existing BGA projects in the workspace, or ask the user.
- Chrome extension "Claude in Chrome" connected
- `make check` and `make deploy` working in the project directory
- Game rules document available (provided by user)
- Graphics / assets available in `img/` (SVG preferred)
- Git repository initialized (or empty directory — will be initialized in step 1.3)

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

**→ Commit now** (scaffold + .gitignore).

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

### 1.3 Makefile and git init

If the repository has no commits yet, initialize it now: `git init`, create a `.gitignore` (exclude `bga_initial_code_template/`), and make the initial commit with the scaffold and project structure.

Create a Makefile with `check` (PHP lint) and `deploy` (SCP to BGA Studio) targets. See `references/makefile-template.md` for the full template.

BGA Studio is **SFTP-only** (no SSH shell) — rsync does not work. Use `scp` with explicit file lists.

**→ Commit now** (Makefile + config files).

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
    // Game constants — prefer class constants over define() in material.inc.php
    // (define() creates globals invisible inside the namespace)
    public const MY_CONSTANT = 1;

    public function __construct() {
        parent::__construct();
        // Minimal — do NOT call initGameStateLabels here
    }

    protected function initTable(): void {
        // Using $this->bga->globals for all state (any type, JSON-serialized)
        // initGameStateLabels is legacy and int-only — avoid it
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
- Namespace: `Bga\Games\GameName` — **PascalCase**, matching the scaffold (not lowercase)
- **Constants:** use `public const` in Game.php, NOT `define()` in `material.inc.php`. `define()` creates global constants that are invisible inside the game's namespace. Reference as `self::MY_CONST` in Game.php, `Game::MY_CONST` in States.
- **Globals:** use `$this->bga->globals->set/get` (any type, JSON-serialized). `initGameStateLabels` is legacy and **int-only** — these are two distinct systems, do not mix them
- `material.inc.php` is auto-included by the framework — **never** `include()` it manually
- No `self::` on instance methods — PHP 8.4 generates warnings that corrupt JSON output. Use `$this->` everywhere. (`self::CONST` for class constants is fine)
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
        // Only id and type — NO name, description, or transitions
        parent::__construct($game, id: 20, type: StateType::ACTIVE_PLAYER);
    }

    public function onEnteringState(array $args): void { }

    #[PossibleAction]
    public function actDoSomething(string $param): string {
        $playerId = $this->game->getCurrentPlayerId();
        // validate...
        // do work...
        // Send notification for every action
        $this->game->bga->notify->all('somethingDone', clienttranslate('${player_name} did something'), [
            'player_id' => $playerId,
            'player_name' => $this->game->getPlayerNameById($playerId),
        ]);
        return NextState::class; // return the next state class, not a string
    }

    // Action parameter validation (autowired — no action.php needed)
    #[PossibleAction]
    public function actSelectCards(#[IntArrayParam] array $ids): string { ... }

    #[PossibleAction]
    public function actMove(#[StringParam(enum: ['up','down','left','right'])] string $dir): string { ... }
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

// Scores (write before transitioning to state 99)
$this->bga->playerScore->set($pid, $score);

// DB helpers
$rows = $this->getCollectionFromDb("SELECT * FROM my_table");  // not getObjectListFromDB(sql, true) — broken
$row  = $this->getObjectFromDB("SELECT * FROM my_table WHERE id=$id");
```

### 2.5 BGA Libraries — Quick Reference

When you need a game component, check this table FIRST. Each library has a detailed reference in `references/`. **Read the reference file before implementing** — it contains setup code, full API, and pitfalls.

| Need | Library | Reference | Side |
|------|---------|-----------|------|
| Cards/tiles (server logic) | **Deck** | `references/deck.md` | PHP |
| Cards/tiles (client display) | **BgaCards** | `references/bga-cards.md` | JS |
| Per-player counters (money, resources) | **PlayerCounter** | `references/player-table-counter.md` | PHP |
| Game-wide counter (round, phase) | **TableCounter** | `references/player-table-counter.md` | PHP |
| Numeric display with animation | **Counter** | `references/counter.md` | JS |
| Dice rolling & display | **BgaDice** | `references/bga-dice.md` | JS |
| Item collections (hand, market) | **Stock** | `references/stock.md` | JS |
| Token placement areas | **Zone** | `references/zone.md` | JS |
| Scrollable/infinite board | **Scrollmap** | `references/scrollmap.md` | JS |
| Move animations & scoring | **BgaAnimations** | `references/bga-animations.md` | JS |
| End-game score sheet | **BgaScoreSheet** | `references/bga-score-sheet.md` | JS |
| Auto-size text on cards | **BgaAutofit** | `references/bga-autofit.md` | JS |
| Collapsible sections | **ExpandableSection** | `references/expandable-section.md` | JS |
| Drag-and-drop (legacy) | **Draggable** | `references/draggable.md` | JS |

> **Workflow:** identify the need → find the library in the table → read `references/<file>.md` → implement using the patterns from the reference.

### 2.6 Game.js structure

```javascript
// modules/js/Game.js — ES6, no framework dependencies
export class Game {  // MUST be named exactly "Game"
    constructor() {
        // called once when the game loads
    }

    setup(gamedatas) {
        // 1. Build board/pieces from gamedatas
        for (const [pid, player] of Object.entries(gamedatas.players)) {
            // render player areas
        }

        // 2. Connect click handlers
        this.bga.gameui.connectClass('my-piece', 'onclick', 'onPieceClick');

        // 3. Setup notification handlers (auto-detects notif_* methods)
        this.bga.notifications.setupPromiseNotifications();

        // 4. Render initial state (onEnteringState is NOT called after setup)
        const stateArgs = gamedatas.gamestate?.args;
        // Use stateArgs to render the complete initial UI
    }

    onEnteringState(stateName, args) {
        // show/hide UI per state (called on state transitions, NOT after setup)
    }

    onLeavingState(stateName) { }

    onUpdateActionButtons(stateName, args) {
        // Add action buttons — REQUIRED for multiactive states
        if (stateName === 'playerTurn') {
            this.bga.statusBar.addActionButton(_('Pass'), () => {
                this.bga.actions.performAction('actPass');
            }, { color: 'secondary' });
        }
    }

    // Notification handlers — auto-registered by setupPromiseNotifications()
    // Name = 'notif_' + the notification type from PHP notify->all('somethingDone', ...)
    async notif_somethingDone(args) {
        // args = PHP notification args; values are STRINGS — parseInt() before arithmetic
        await this.bga.gameui.slideToObject('token', 'target').play().promise;
    }

    // Action handler — ONLY from user events, NEVER from notifications/loops/callbacks
    onPieceClick(e) {
        const id = e.currentTarget.id;
        this.bga.actions.performAction('actSelectPiece', { id: parseInt(id, 10) });
    }
}
```

Assets: reference as `g_gamethemeurl + 'img/file.svg'` or `this.bga.images.getImgUrl('file.svg')`

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

**gameinfos.inc.php** — essential fields:
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

Config rules: no comments in JSON, no trailing commas, names are auto-translated (no `totranslate()` needed). After changes: deploy AND reload via BGA Studio manage page.

### 2.8 BGA Framework — Deep Reference

For detailed API beyond the patterns above, load the relevant reference file.

| Topic | Reference | When to load |
|-------|-----------|--------------|
| JS full API (DOM, animations, tooltips, dialogs, panels) | `references/js-framework.md` | Building complex UI |
| PHP full API (DB, players, states, scoring, undo) | `references/php-framework.md` | Advanced game logic |
| Notification system (PHP + JS) | `references/notifications.md` | Custom notification handling |
| Config files (options, prefs, stats, gameinfos) | `references/config-files.md` | Setting up or modifying config |
| Translations & i18n | `references/translations.md` | Adding translatable strings |
| BGA Studio Guidelines (layout, a11y, UX) | `references/guidelines.md` | Polishing UI / preparing for review |
| Rules clarification process | `references/rules-clarification.md` | Analyzing rules, managing author Q&A |

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

Click "Play with friends" on the lobby page, then "Express start" on the table page:

```javascript
// Step 1: On the lobby page (/lobby?game=GAME_ID), click "Play with friends"
const pwf = Array.from(document.querySelectorAll('a')).find(a => a.textContent.includes('Play with friends'));
pwf?.click();
```

Wait 5 seconds — URL changes to `/table?table=N`.

```javascript
// Step 2: On the table page, click "Express start" (overriding confirm dialog)
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

From the **game page** (not lobby), use `gameui.ajaxcall`:

```javascript
gameui.ajaxcall('/table/table/quitgame.html', {table: TABLE_ID, neutralized: true, s: 'table_quitgame'}, gameui,
  () => console.log('quit ok'),
  () => console.log('quit err')
);
```

Note: `mainsite` is not defined on game pages — use `gameui` instead. Direct `fetch()` fails (CSRF), but `gameui.ajaxcall` adds the token automatically.

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

**Commit automatically after each completed step** — do not wait for the user to ask. Each commit should capture a coherent, working (or at least lint-passing) state.

Commit points (at minimum):
- After scaffold download and project structure setup (1.1–1.3)
- After rules analysis and questions document (1.4)
- After each implemented game state or feature (e.g., "feat: implement draft phase", "feat: add scoring")
- After each successful bug fix cycle in Phase 3
- After any significant refactoring

```bash
git add -A && git commit -m "feat: description of what was done

Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>"
```

**Key rule:** smaller, frequent commits are better than large ones. If a step involves multiple independent changes (e.g., implementing two unrelated game states), commit each separately.

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
1. Quit **all** game table instances — either manually via the browser, or programmatically from the game page using `gameui.ajaxcall('/table/table/quitgame.html', {table: N, neutralized: true}, gameui, ok, err)`
2. Create a fresh table — the correct `dbmodel.sql` will be applied
3. If the table name conflicts with a BGA internal table, rename it (e.g., `moves` → `q_moves`) and redeploy before creating the fresh table

**Programmatic quit works** from the game page via `gameui.ajaxcall` (which adds the CSRF token automatically). Direct `fetch()` fails due to CSRF. The `mainsite` global is available on the lobby page, `gameui` on the game page.

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
- **Never call `getCurrentPlayerId()`** in `setupNewGame()`, `zombieTurn()`, or args methods — causes "Not logged" errors
- **Cannot change active player** inside ACTIVE_PLAYER states — use a GAME-type state for that
- **Args methods must return arrays** — never int/string, never modify DB in args
- **At least one notification per action** is required (state transitions count)
- **Notification size limit**: 128KB total across bundled requests
- **`getCollectionFromDb()` uses first column as array key** — duplicate values silently overwrite. Always put a unique column first.
- **String comparison**: use `===` not `==` — hex colors like `'4baae2'` miscast with `==`
- **Action autowiring** (new framework): `#[PossibleAction]` + typed parameters = no action.php needed
- **Forbidden action parameter names**: `$args`, `$activePlayerId`, `$currentPlayerId` — reserved by framework
- **`escapeStringForDB($str)`** — mandatory for any player-supplied string in SQL
- **`getObjectListFromDB($sql)`** — simple array (no key issues); use instead of `getCollectionFromDb` when first column may have duplicates
- **`activeNextPlayer()` / `changeActivePlayer()`** — only works in GAME-type states, not ACTIVE_PLAYER
- **Use `clienttranslate()`** for all notification messages — literal text only, no variables inside

### JavaScript / Client

- **`setup()` runs before `onEnteringState()`** — render complete initial state in setup using `gamedatas.gamestate.args`
- **Call `setupPromiseNotifications()`** in `setup()` — auto-detects all `notif_*` methods
- **In MULTIPLE_ACTIVE_PLAYER states**: active players are NOT active yet in `onEnteringState` — use `onUpdateActionButtons` for active-dependent UI
- **`performAction` only from user events** — NEVER from notifications, loops, callbacks, or state methods
- **`slideToObject()` returns dojo animation** — must call `.play()` on the result
- **Slide methods incompatible with CSS transforms** (scale, zoom, rotate)
- **Check `bgaAnimationsActive()`** before animating
- **`parseInt(value, 10)`** — notification args are strings, `+=` will concatenate instead of add
- **Hotseat shares `gamedatas`** across players — no per-player client data
- **Read-only detection**: check `isCurrentPlayerSpectator() || typeof g_replayFrom != 'undefined' || g_archive_mode`
- **Prefix CSS classes** with game name — e.g., `mygame_selected`, not `selected`
- **`attachToNewParent` clones** the element — destroys original references and dojo.connect handlers
- **Never call setState in notifications** — causes race conditions and breaks replays

### Zombie handling

- Never call `getCurrentPlayerId()` in `zombieTurn()` — use the `$active_player_id` parameter
- States must have a `"zombiePass"` transition (exact spelling)
- Cannot end game from zombie method — must continue game logic

### Debug tips

- **Express stop** on the table config page kills a stuck game (overrides confirm with `window.confirm = () => true`)
- **Temporary debug actions** (`#[PossibleAction]`) are effective for testing end-game / specific positions — call via `gameui.ajaxcall`, remove before commit
- **Full SQL/request logs**: `/1/GAME_NAME/GAME_NAME/logaccess.html?table=N`
- **CSS state class**: `#overall-content` gets `.gamestate_playerTurn` etc. — use for conditional visibility without JS
