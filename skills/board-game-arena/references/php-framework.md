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
$this->getActivePlayerId();                  // current turn holder
$this->getCurrentPlayerId();                 // requester (actions + getAllDatas only!)
$this->loadPlayersBasicInfos();              // cached player array
$this->getPlayerNameById($pid);
$this->getPlayerColorById($pid);

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

// Multiple active players
$this->gamestate->setAllPlayersMultiactive();
$this->gamestate->setPlayersMultiactive($ids, $nextTransition, $exclusive);
$this->gamestate->setPlayerNonMultiactive($pid, $nextTransition);
$this->gamestate->getActivePlayerList();     // array of active player IDs
$this->gamestate->isPlayerActive($pid);      // check

// CRITICAL: Cannot change active player inside ACTIVE_PLAYER states
// Use a GAME-type state for that
```

## State Transitions

```php
// In legacy states (states.inc.php)
$this->gamestate->nextState('transitionName');
$this->gamestate->jumpToState($stateId);     // advanced — skip transition

// In new framework state classes — return from act methods:
return NextState::class;                     // preferred
return 99;                                   // by ID
return 'transitionName';                     // legacy compatibility

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

**Reserved (magic) parameter names** — the framework auto-fills these in act methods, so they can never carry client-sent values: `$args`, `$activePlayerId`/`$active_player_id`, `$activePlayerNo`/`$active_player_no`, `$currentPlayerId`/`$current_player_id`, `$currentPlayerNo`/`$current_player_no`. (In PRIVATE-state `getArgs`/`onEnteringState`, `$playerId`/`$player_id`/`$playerNo`/`$player_no` are magic too.)

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

// Legacy (int only — DO NOT mix with above)
$this->initGameStateLabels(['key' => 10]);
$this->setGameStateValue('key', $n);
$this->getGameStateValue('key');
```

## Scoring & Ranking

```php
// Standard scoring (highest wins)
$this->bga->playerScore->set($pid, $score);
$this->bga->playerScoreAux->set($pid, $tiebreaker);

// Reverse scoring (lowest wins)
return GameResult::individualRanking($players, reverseScore: true);

// Cooperative
// Set 'is_coop' => 1 in gameinfos.inc.php
// All winners get same score; losers get 0

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

// User error — rolls back DB, prevents notifications, shows error to player
throw new UserException(clienttranslate('You cannot do that'));

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
    // Simplest robust pattern: delegate to one of the state's act methods.
    // NEVER call getCurrentPlayerId() here — use $playerId parameter
    // Cannot end game from zombie — must continue
    return $this->actPass();
}
```

## Database Migrations (post-release)

```php
public function upgradeTableDb(string $fromVersion): void {
    if (version_compare($fromVersion, '2504151200', '<')) {
        $this->DbQuery("ALTER TABLE my_table ADD COLUMN new_col INT DEFAULT 0");
    }
}
```

## Pitfalls

- **DB values are strings** — always cast: `(int)$row['value']`
- **`getCurrentPlayerId()` only in actions and `getAllDatas()`** — "Not logged" errors elsewhere
- **`getCollectionFromDb` keys by first column** — duplicates silently overwrite
- **`getObjectListFromDB`** — use for simple arrays without key issues
- **Active player changes only in GAME states** — not in ACTIVE_PLAYER states
- **Args methods: return arrays only, never modify DB, never call getCurrentPlayerId()**
- **Notification payload limit: 128KB** across bundled requests
- **`escapeStringForDB()`** for any player-supplied string
- **String comparison: `===` not `==`** — hex colors like `'4baae2'` miscast with `==`
- **`material.inc.php` is auto-included** — never `include()` it manually
- **Constants: `public const` not `define()`** — define() globals invisible in namespace
