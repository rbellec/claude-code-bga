# PlayerCounter & TableCounter — BGA Library Reference

**Type:** PHP Server (Modern framework 2025+)
**Doc:** https://en.doc.boardgamearena.com/PlayerCounter_and_TableCounter
**When to use:** Track per-player resources (money, energy, lives) or game-wide values (round number, phase) with automatic DB persistence and JS notification.

## Setup

```php
// In Game.php — declare as properties
public PlayerCounter $playerEnergy;
public TableCounter $roundCounter;

public function __construct() {
    parent::__construct();
    $this->playerEnergy = $this->counterFactory->createPlayerCounter('energy');
    $this->roundCounter = $this->counterFactory->createTableCounter('round');
    // Full factory signature: createPlayerCounter(string $name, ?int $min = 0,
    //   ?int $max = null, CounterVisibility $visibility = VISIBLE, bool $useNo = false, ?bool $strict = null)
    // min/max bound the counter; visibility CounterVisibility::VISIBLE | SELF | HIDDEN (see Pitfalls);
    // useNo = address players by seat number instead of id; strict = throw when the
    // id/no passed to get/set/inc is unknown (also ->setUseNo() / ->setStrict() later)
}

// In setupNewGame() — use its $players argument directly:
$this->playerEnergy->initDb(array_keys($players), 3); // initial value = 3
$this->roundCounter->initDb(1); // initial value = 1

// In getAllDatas() — fillResult writes into $result['players'][$pid], so build that first:
public function getAllDatas(int $currentPlayerId): array {
    $result = [];
    $result['players'] = $this->getCollectionFromDb("SELECT player_id AS id, player_score AS score FROM player");
    $this->playerEnergy->fillResult($result, currentPlayerId: $currentPlayerId); // currentPlayerId needed for SELF visibility
    $this->roundCounter->fillResult($result);
    return $result;
}
```

## API — PlayerCounter

- `initDb(array $playerIdsOrNos, int $initialValue = 0)` — create DB table (setupNewGame only)
- `get(int $playerId): int` — get one player's value
- `set(int $playerId, int $value, ?NotificationMessage)` — set + notify
- `inc(int $playerId, int $inc, ?NotificationMessage)` — increment + notify
- `getAll(): array` — all player values `[playerId => value]`
- `setAll(int $value, ?NotificationMessage)` — set for all players (`set`/`inc`/`setAll` return the new value)
- `getMin()`, `getMax()`, `setVisibility(CounterVisibility)`, `isVisible()/isSelf()/isHidden()`, `setUseNo()`, `setStrict()`
- `fillResult(array &$result, ?string $fieldName, ?int $currentPlayerId)` — populate getAllDatas

## API — TableCounter

- `initDb(int $initialValue = 0)` — create DB table (setupNewGame only)
- `get(): int` — get value
- `set(int $value, ?NotificationMessage)` — set + notify
- `inc(int $inc, ?NotificationMessage)` — increment + notify
- `fillResult(array &$result, ?string $fieldName)` — populate getAllDatas

## JS Client Setup

Counters auto-update on the client when notifications include counter data. Create matching JS counters:

```javascript
// In setup(gamedatas):
this.energyCounters = {};
for (const [playerId, player] of Object.entries(gamedatas.players)) {
    this.energyCounters[playerId] = new ebg.counter();
    this.energyCounters[playerId].create(`energy-counter-${playerId}`, {
        value: player.energy,
        playerCounter: 'energy',
        playerId: parseInt(playerId, 10),   // Object.entries keys are strings; the setting is a number
    });
}
```

## Auto-notification Parameters

When a notification message is provided, counters automatically include these args:
- `name` — counter name
- `value` — new value
- `oldValue` — previous value
- `inc` — change amount
- `absInc` — absolute change
- `playerId` — (PlayerCounter only; typed args: `name`, `value`, `oldValue`, `inc`, `absInc`, `playerId`)

## Pitfalls

- **Pass `null` as message** to skip frontend notification
- **Zero increment** (`inc=0`) sends no notification
- **`fillResult` field name** defaults to counter name; override with 2nd parameter
- **Default `playerScore` / `playerScoreAux`** are PlayerCounters too: `$this->playerScore->fillResult($result, 'score')` or `SELECT player_score AS score`
- **Visibility** is the enum `CounterVisibility::VISIBLE` (default), `::SELF` (own value only — pass `$currentPlayerId` to `fillResult`), `::HIDDEN`; there is no `'private'`
