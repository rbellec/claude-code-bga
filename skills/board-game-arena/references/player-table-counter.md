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
}

// In setupNewGame():
$players = $this->loadPlayersNatural();
$this->playerEnergy->initDb(array_keys($players), 3); // initial value = 3
$this->roundCounter->initDb(1); // initial value = 1

// In getAllDatas():
public function getAllDatas(int $currentPlayerId): array {
    $result = [];
    $this->playerEnergy->fillResult($result);
    $this->roundCounter->fillResult($result);
    return $result;
}
```

## API — PlayerCounter

- `initDb(array $playerIds, int $initialValue = 0)` — create DB table (setupNewGame only)
- `get(int $playerId): int` — get one player's value
- `set(int $playerId, int $value, ?NotificationMessage)` — set + notify
- `inc(int $playerId, int $inc, ?NotificationMessage)` — increment + notify
- `getAll(): array` — all player values `[playerId => value]`
- `setAll(int $value, ?NotificationMessage)` — set for all players
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
        playerId: playerId,
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
- `playerId`, `player_name` — (PlayerCounter only)

## Pitfalls

- **Pass `null` as message** to skip frontend notification
- **Zero increment** (`inc=0`) sends no notification
- **`fillResult` field name** defaults to counter name; override with 2nd parameter
- **Default `playerScore`** counter exists but must be queried explicitly in `getAllDatas()`: `SELECT player_score AS score`
- **Visibility options** — `"self"` shows `-` for other players; `"private"` hides from all
