# PHP Framework — BGA Reference

**Type:** PHP Server
**Doc:** https://en.doc.boardgamearena.com/Main_game_logic:_yourgamename.game.php
**When to use:** Full API reference for server-side game logic beyond the basics in SKILL.md.

## Database Access

```php
// Query
$this->DbQuery("UPDATE my_table SET col='val' WHERE id=$id");

// Single value
$count = (int)$this->getUniqueValueFromDB("SELECT COUNT(*) FROM my_table");

// Associative array (keyed by FIRST column — must be unique!)
$rows = $this->getCollectionFromDb("SELECT id, name, value FROM my_table");

// Simple array (no key issues — use when first column may have duplicates)
$rows = $this->getObjectListFromDB("SELECT col1, col2 FROM my_table");

// Single row
$row = $this->getObjectFromDB("SELECT * FROM my_table WHERE id=$id");

// Two-level nested array (keyed by first two columns)
$grid = $this->getDoubleKeyCollectionFromDB("SELECT x, y, value FROM board");

// SQL injection prevention (for player-supplied strings)
$safe = $this->escapeStringForDB($userInput);
```

## Player Information

```php
// Basic info
$this->getPlayerCount();                     // total players
$this->getActivePlayerId();                  // current turn holder — returns a STRING; prefer the magic int $activePlayerId
$this->getCurrentPlayerId($bReturnNullIfNotLogged = false); // requester — STRING, throws when no one is logged (zombie,
                                             // setupNewGame, args methods); prefer magic int $currentPlayerId (SKILL 2.3)
$this->isCurrentPlayerZombie();
$this->loadPlayersBasicInfos();              // cached player array (player_no, player_zombie, player_eliminated, …)
$this->getPlayerNameById($pid);
$this->getPlayerColorById($pid);
$this->getPlayerNameByNo($no);               // by seat number ("player no", 1-based turn order)

// "Player no" variants: act methods can take the magic `int $activePlayerNo` /
// `int $currentPlayerNo` instead of the id; notify, playerStats and PlayerCounter
// switch to no-addressing with ->setUseNo(true) (JS side: this.bga.players.getPlayerByNo & co)

// Turn order navigation
$this->getPlayerAfter($pid);                 // next in natural order
$this->getPlayerBefore($pid);                // previous
$this->getNextPlayerTable();                 // [pid => next_pid] for all
$this->getPrevPlayerTable();                 // [pid => prev_pid] for all
```

## Active Player Management

```php
// Single active player (only in GAME-type states!)
$this->activeNextPlayer();                   // advance turn
$this->activePrevPlayer();                   // go back
$this->gamestate->changeActivePlayer($pid);  // set specific player

// Multiple active players — $next is a state class (NextState::class), a state id,
// or a callable; legacy transition names only work with states.inc.php
$this->gamestate->setAllPlayersMultiactive();
$this->gamestate->setPlayersMultiactive($ids, $next, $exclusive);
$this->gamestate->setPlayerNonMultiactive($pid, $next);   // returns true when it moved to $next
$this->gamestate->updateMultiactiveOrNextState($next);     // go to $next if nobody is active any more
$this->gamestate->getActivePlayerList();     // active player IDs — as strings, cast them
$this->gamestate->isPlayerActive($pid);      // check

// Current state as an object
$this->gamestate->getCurrentMainStateId();
$this->gamestate->getCurrentMainStateClass();        // state class instance (ignores private states)
$this->gamestate->getCurrentStateClass($playerId);   // the player's private state if in one

// CRITICAL: Cannot change active player inside ACTIVE_PLAYER states
// Use a GAME-type state for that
```

## State Transitions

```php
// In legacy states (states.inc.php)
$this->gamestate->nextState('transitionName');
$this->gamestate->jumpToState($stateId);     // advanced — skip transition

// In new framework state classes — return from act methods / onEnteringState:
return NextState::class;                     // preferred (method typed `: string`)
return 99;                                   // by ID — only from a method NOT typed `: string` (strict_types)
return GameResult::coop(score: $s, win: $won); // from the end-score state: ends the game (see Scoring)
// 'transitionName' strings only resolve on states declaring `transitions` (legacy states.inc.php)

// Reload cached state after manual DB manipulation
$this->gamestate->reloadState();
```

## Action Validation Attributes

```php
use Bga\GameFramework\Actions\Types\IntParam;
use Bga\GameFramework\Actions\Types\StringParam;
use Bga\GameFramework\Actions\Types\IntArrayParam;
use Bga\GameFramework\Actions\Types\JsonParam;
use Bga\GameFramework\Actions\CheckAction;

#[PossibleAction]
public function actPlayCard(int $cardId): string { ... }

#[PossibleAction]
public function actSelectCards(#[IntArrayParam] array $ids): string { ... }
// If the array may be empty: type it `#[IntArrayParam] ?array $ids` and treat null as []

#[PossibleAction]
public function actMove(#[StringParam(enum: ['up','down','left','right'])] string $dir): string { ... }

#[PossibleAction]
public function actBid(#[IntParam(min: 0, max: 100)] int $amount): string { ... }

#[PossibleAction]
public function actSubmitData(#[JsonParam(class: MyDTO::class)] MyDTO $data): string { ... }

// Disable automatic checkAction (for multiactive "change mind" actions)
#[PossibleAction]
#[CheckAction(false)]
public function actCancel(): string { ... }
```

**Reserved (magic) parameter names** — the framework auto-fills these in act methods, so they can never carry client-sent values: `array $args` (the state's own `getArgs()` result — validate client input against it rather than recomputing), `$activePlayerId`/`$active_player_id`, `$activePlayerNo`/`$active_player_no`, `$currentPlayerId`/`$current_player_id`, `$currentPlayerNo`/`$current_player_no`. (In PRIVATE-state `getArgs`/`onEnteringState`, `$playerId`/`$player_id`/`$playerNo`/`$player_no` are magic too.)

## Inside a State Class

`GameState` exposes the framework sub-objects directly — `$this->bga`, `$this->notify`, `$this->globals`, `$this->tableOptions`, `$this->userPreferences`, `$this->tableStats`, `$this->playerStats`, `$this->deckFactory`, `$this->counterFactory`, `$this->playerScore`, `$this->playerScoreAux`, `$this->gamestate` — so `$this->notify->all(...)` and `$this->game->bga->notify->all(...)` are equivalent. Constructor options beyond `id`/`type`: `updateGameProgression: true` (recompute `getGameProgression()` when entering), `initialPrivate: PrivateX::class`, `description` / `descriptionMyTurn` (status-bar texts, `clienttranslate`d; the scaffold sets them client-side with `statusBar.setTitle` instead).

## Private Parallel States

```php
// In state class constructor:
parent::__construct($game, id: 30, type: StateType::MULTIPLE_ACTIVE_PLAYER,
    initialPrivate: PrivateChoose::class);

// In the MULTIPLE_ACTIVE_PLAYER state's onEnteringState:
$this->game->gamestate->initializePrivateStateForAllActivePlayers();

// In a private state's act method:
$this->game->gamestate->nextPrivateState($playerId, 'confirm');

// Exit parallel states for a player:
$this->game->gamestate->unsetPrivateState($playerId);
```

## Globals

```php
// Modern API (any type — JSON serialized)
$this->bga->globals->set('key', $value);    // int, string, array, bool
$this->bga->globals->get('key');
$this->bga->globals->inc('counter', 5);     // increment
$this->bga->globals->delete('key');

// Legacy (int only — DO NOT mix with above). Officially @deprecated in the framework
// stub: "use $this->bga->globals API instead. It's not the same DB table as
// GameStateValue, so if your game is in production and using it, keep it"
$this->initGameStateLabels(['key' => 10]);
$this->setGameStateValue('key', $n);
$this->getGameStateValue('key');
```

## Scoring & Ranking

```php
use Bga\GameFramework\GameResult\GameResult;

// Standard scoring (highest wins)
$this->bga->playerScore->set($pid, $score);
$this->bga->playerScoreAux->set($pid, $tiebreaker);

// Reverse scoring (lowest wins) — $players may be null (= all players), ids or Player objects
return GameResult::individualRanking($players, reverseScore: true, reverseScoreAux: false);

// Cooperative — set "is_coop": 1 in gameinfos; one team of all players, outcome decided by $win
return GameResult::coop($players /* or null = all */, score: $teamScore, win: $won);

// Solo (internally a coop result of one player) — see "solo_mode_ranked" in gameinfos
return GameResult::solo($playerId /* or null = the only player */, score: $score, win: $won);

// Where: return the GameResult from the END-SCORE state (98, GAME type) instead of `return 99;`
// — returning a GameResult from a state class ends the game. Plain `return 99` keeps the
// default ranking by player_score / player_score_aux.

// Elimination (mid-game removal)
$this->eliminatePlayer($pid);

// Tie-breaker formula in gameinfos.inc.php:
'tie_breaker_description' => 'Most money, then most buildings'
'tie_breaker_split' => [100, 10, 1]
// Score = 100 * money + 10 * buildings + cards
```

## Undo System

```php
// Enable in gameinfos.inc.php: 'db_undo_support' => true

// In act method (before irreversible logic):
$this->undoSavepoint();
// ... game logic ...
return NextState::class;

// Undo action:
#[PossibleAction]
#[CheckAction(false)]
public function actUndo(): string {
    $this->undoRestorePoint();
    return PlayTurn::class;
}

// Manual log removal (alternative):
$moveId = $this->bga->logs->getCurrentMoveId();
// ... revert state manually ...
$this->bga->logs->remove($moveId);
```

**Constraints:** single savepoint, can't undo in multiactive, active player must match.

## Randomization

```php
bga_rand($min, $max);           // MANDATORY for dice throws, recommended everywhere
random_int($min, $max);         // acceptable
// Deprecated, never use: rand(), mt_rand()
// shuffle() / array_rand() are accepted in review for shuffling
// (Deck::shuffle itself uses shuffle()), but are not cryptographically
// secure — prefer bga_rand for anything a player could exploit
```

## Extra Time

```php
$this->giveExtraTime($pid);             // standard (speed-dependent)
$this->giveExtraTime($pid, $seconds);   // custom (not recommended)
// Only affects real-time games

// Game mode detection
$isTurnBased = $this->bga->tableOptions->isTurnBased();
$isRealTime  = $this->bga->tableOptions->isRealTime();
```

## Table Options, Version, Language, Tournament

```php
$this->bga->tableOptions->get(100);            // ?int — null if the table predates the option
$this->bga->tableOptions->set(100, $value);    // ONLY for "random" options: store the value actually drawn
$this->bga->tableOptions->getGameVersion();    // int
$this->bga->tableOptions->getGameLanguage();   // ?int index into gameinfos language_dependency
                                               // (Table::getGameLanguage() is deprecated)
$this->bga->tournament->isTournament();
$this->bga->tournament->getInfo();             // [] outside a tournament; keys id, name, championship_name, …
$this->bga->tournament->getSeedInfo();         // tournament_seed / step_seed for deterministic randomness
```

## Player Color Preferences

Default colors come from `player_colors` in the gameinfos file. To honor players' favorite colors, set `favorite_colors_support` to true there and call:

```php
// In setupNewGame(), instead of the manual color-assignment loop:
$this->reattributeColorsBasedOnPreferences($players, $this->getGameinfos()['player_colors']);
$this->reloadPlayersBasicInfos();

// If some colors are interchangeable pairs (e.g. teams), also override:
public function getSpecificColorPairings(): array { ... }
```

## Error Handling

```php
use Bga\GameFramework\UserException;
use Bga\GameFramework\VisibleSystemException;
use Bga\GameFramework\NotificationMessage;

// User error — rolls back DB, prevents notifications, shows error to player
throw new UserException(clienttranslate('You cannot do that'));
// With placeholders: message + args through a NotificationMessage
throw new UserException(new NotificationMessage(clienttranslate('You need ${n} more coins'), ['n' => $missing]));

// System error — shows to player, triggers reload (untranslated, devs only)
throw new VisibleSystemException('Something broke');
// Also: Bga\GameFramework\SystemException (server logs only)
// The global \BgaVisibleSystemException / \BgaUserException names are the
// deprecated legacy equivalents (see references/legacy-migration.md)

// DB transactions are automatic:
// - Exception → full rollback, notifications NOT sent
// - Success → commit, notifications sent
// CRITICAL: Never use TRUNCATE or DROP (implicit commits break rollback)
```

## Zombie Handling

```php
// In state class — returns the next state, exactly like an act method:
public function zombie(int $playerId): string {
    // Make a valid default move for the disconnected player.
    // Simplest robust pattern: delegate to one of the state's act methods,
    // passing the player id through its $activePlayerId magic param.
    // NEVER call getCurrentPlayerId() on this code path — no current player.
    // Cannot end game from zombie — must continue
    return $this->actPass($playerId);
}

// The act method it delegates to (magic param autowired by the framework):
#[PossibleAction]
public function actPass(int $activePlayerId): string { ... }

// Choosing the default move — helpers on GameState, matching the declared Zombie Mode level:
$choice = $this->getRandomZombieChoice($legalMoveIds);                 // level 1: random legal choice
$choice = $this->getBestZombieChoice(['a' => 3, 'b' => 7], reversed: false); // level 2: best immediate points (random among ties)
```

## Debug: auto-play with zombies

```php
// From a debug_* helper (see references/debug-helpers.md): let the zombie code play
// moves until the callback returns true — the scaffold's debug_playOneMove() uses it
public function debug_playOneMove() {
    $this->bga->debug->playUntil(fn(int $count) => $count == 1);
}
```

## Database Migrations (post-release)

```php
// Param stays UNTYPED (parent declares it so); it is the int version "YYMMDDHHMM" without the dash
public function upgradeTableDb($from_version) {
    if ($from_version <= 2504151200) {
        // DBPREFIX_ is mandatory: the query is replayed on every running table's DB
        $this->applyDbUpgradeToAllDB("ALTER TABLE `DBPREFIX_mygame_board` ADD COLUMN `new_col` INT DEFAULT 0");
    }
}
// ItemManager tables: $this->items->upgradeTableDbAddColumns(['newField']) — see references/item-manager.md
// Test it from Studio with a debug helper: public function debug_testUpgradeTableDb(int $v = 0) { $this->upgradeTableDb($v); }
```

## Environment & moderation

```php
$this->getBgaEnvironment();          // 'studio' | 'prod' — guard debug_* helpers
$this->logTextForModeration($playerId, $text); // any free text a player can type (names, messages)
```

## Pitfalls

- **DB values are strings** — always cast: `(int)$row['value']`
- **`getCurrentPlayerId()` only in actions and `getAllDatas()`** — "Not logged" errors elsewhere; returns a string — prefer the magic `int $currentPlayerId` / `$activePlayerId` params (SKILL 2.3)
- **Deck is deprecated** — new projects use ItemManager (`references/item-manager.md`)
- **`getCollectionFromDb` keys by first column** — duplicates silently overwrite
- **`getObjectListFromDB`** — use for simple arrays without key issues
- **Active player changes only in GAME states** — not in ACTIVE_PLAYER states
- **Args methods: return arrays only, never modify DB, never call getCurrentPlayerId()**
- **Notification payload limit: 128KB** across bundled requests
- **`escapeStringForDB()`** for any player-supplied string
- **String comparison: `===` not `==`** — hex colors like `'4baae2'` miscast with `==`
- **`material.inc.php` is auto-included** — never `include()` it manually
- **Constants: `public const` not `define()`** — define() globals invisible in namespace
- **`_ide_helper.php` (scaffold root) is the authoritative PHP API** — grep it before declaring a method missing or inventing one; it also carries the `@deprecated` notes
