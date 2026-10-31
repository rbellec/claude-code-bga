# ItemManager — BGA Library Reference

**Type:** PHP Server (new framework, 2026+)
**Doc:** https://en.doc.boardgamearena.com/ItemManager · https://en.doc.boardgamearena.com/Collection
**When to use:** Any game objects stored in DB — cards, tiles, dice, tokens, meeples — with locations, ordering, picking and shuffling. **The successor of Deck** (Deck and Stock were deprecated in October 2026): it maps a PHP class of your choice to a table, so items come back as typed objects instead of string-valued arrays. Pairs with BgaCards on the client. Reference game: Hearts (Studio project `hearts`).

## Setup

### 1. The item class — attributes describe the table

```php
<?php
declare(strict_types=1);
namespace Bga\Games\GameName;   // autoloaded from modules/php/ (SKILL 2.1)

use Bga\GameFramework\Components\ItemManager\Item;
use Bga\GameFramework\Components\ItemManager\ItemField;
use Bga\GameFramework\Components\ItemManager\ItemFieldKind;

#[Item]                                   // #[Item('mygame_card')] to name the table (default: short class name)
class Card
{
    #[ItemField(kind: ItemFieldKind::ID)]
    public int $id;                       // generated automatically

    #[ItemField(kind: ItemFieldKind::LOCATION, locationIndex: 0)]
    public string $location;              // 'deck', 'hand', … — first part: non-null string|int

    #[ItemField(kind: ItemFieldKind::LOCATION, locationIndex: 1)]
    public ?int $locationArg;             // e.g. player id — extra parts MUST be nullable (int|string|null if it may hold names)

    #[ItemField(kind: ItemFieldKind::ORDER)]
    public int $order;                    // zero-based position inside the exact location, kept dense

    #[ItemField] public int $suit;        // custom fields: bool/int/float/string → scalar columns,
    #[ItemField] public int $value;       //   arrays/objects/enums → JSON (reconstructed as their class)
    #[ItemField] public bool $faceUp = false;

    public string $displayName;           // no attribute → not persisted
}
```

The three kinds `ID`, `LOCATION`, `ORDER` are mandatory; names are yours. `#[ItemField(dbField: 'card_type')]` maps to an existing column (Deck migration), `serialize: true` stores PHP-serialized objects instead of JSON.

### 2. The manager — created in the constructor, table created in `setupNewGame`

```php
use Bga\GameFramework\Components\ItemManager\ItemLocation;
use Bga\GameFramework\Components\ItemManager\ItemManager;
use Bga\GameFramework\Components\ItemManager\Location;   // Location::from / Location::filter

class Game extends \Bga\GameFramework\Table
{
    public ItemManager $cards;

    public function __construct() {
        parent::__construct();
        $this->cards = $this->bga->itemManagerFactory->createItemManager(
            Card::class,
            locations: ItemLocation::getDefaults(),   // deck, discard, table, hand — deck auto-refills from discard
        );
        // or explicit: locations: [new ItemLocation('deck'), new ItemLocation('bag', randomPick: true), new ItemLocation('played*')]
    }

    protected function setupNewGame($players, $options = []) {
        // … player rows, reloadPlayersBasicInfos() …
        $this->cards->initDb();              // creates the table — do NOT declare it in dbmodel.sql
        $rows = [];
        foreach ([1, 2, 3, 4] as $suit) {
            foreach (range(2, 14) as $value) {
                $rows[] = ['location' => 'deck', 'suit' => $suit, 'value' => $value];   // keys = property names
            }
        }
        $this->cards->createItems($rows);   // 'item_nbr' => 3 creates copies; rows are randomized, no ids returned
        $this->cards->shuffle('deck');
        foreach (array_keys($players) as $pid) {
            $this->cards->pickItems(5, 'deck', ['hand', $pid]);
        }
        // …
    }
}
```

Locations are written `'deck'`, `['hand', $playerId]` or `Location::from('hand', $playerId)`. Register every location before use (unknown location → configuration exception). A name ending in `*`/`%` is a prefix wildcard (`'played*'` accepts `played1`, `played2`…), but prefer a second LOCATION field when the suffix is data such as a player id.

## API

Methods returning several items return a **`Collection`** keyed by item id (see below); `->values()` gives a zero-based list — **use it before sending items to the client**, otherwise JSON turns the keyed array into an object.

### Picking / moving / ordering
- `pickItem($from, $to): ?Card` · `pickItems(int $n, $from, $to): Collection` — top items; honors `randomPick` and `autoReshuffleFrom`; fewer items when the source runs out
- `moveItem($itemOrId, $to, ?int $order = null)` — appended by default, `order: 0` prepends; updates the object's location/order too
- `moveItems($itemsOrIds, $to, bool $prepend = false)` — one ordered block
- `moveAllItemsInLocation($from /* null = everything */, $to)`
- `setItemOrder($itemOrId, int $order)` · `getMaxOrderInLocation($location)`
- `shuffle($location)` — randomizes ORDER only; `changeIds($itemsOrIds): array` (old ⇒ new) / `changeIdsForLocation($location)` — re-hide items players have seen

### Querying
- `getItemById(int $id): ?Card` · `getItemsByIds(array $ids, ?string $sortByField, bool $reversed)`
- `getItemsInLocation($location, bool $reversed = false, ?int $limit = null, ?string $sortByField = null)` — ORDER ascending by default
- `getItemOnTop($location): ?Card` · `getItemsOnTop(int $n, $location)` · `countItemsInLocation($location): int`
- `getItemsByFieldName('suit', 2)` / `getItemsByFieldName('suit', [1, 2])` · `getItemsByFieldNames(['suit' => 1, 'location' => 'hand', 'locationArg' => $pid])` (AND)
- `getAllItems(?int $limit = null)` — no ordering guaranteed
- Read filters: `getItemsInLocation('hand')` (all hands), `Location::filter('hand', [12, 34])`, `Location::filter([['deck', 'discard']])` — filters are read-only; moving/picking need a concrete location

### Updating
- Changing a property does **not** write the DB: `$card->faceUp = true; $this->cards->updateItem($card, 'faceUp');` (`null` fields = all but id)
- `updateItems($items, $fields)` · `updateAllItems('faceUp', false)`
- `upgradeTableDbAddColumns(['newField'])` from `upgradeTableDb()` when a field is added after release (adds columns only)

### Factory options
`createItemManager(string $className, ?callable $classNameResolver = null, array $locations = [], ?callable $dbUpdateCallback = null, ?callable $countChangeCallback = null)` — `classNameResolver: fn(?array $row) => match ((int)($row['type'] ?? 0)) { 1 => ExtraTurnCard::class, default => null }` instantiates subclasses per row (must extend the managed class). The two callbacks mirror DB writes / per-location counts; ordinary games don't need them.

## Collection (`Bga\GameFramework\Helpers\Collection`)

Extends `ArrayObject` (iterate, `count()`, `$c[$id]`). Methods return **new** collections and keep keys unless stated:

- Convert: `keys()`, `values()` (zero-based list), `all()` (keyed array), `first()`, `last()`, `random()`, `isEmpty()`, `has($key)`, `contains($value)`
- Callbacks `fn($value, $key)`: `filter`, `map`, `find`, `findKey`, `some`, `every`, `count(fn)`, `sort(uasort-comparator)`; `reduce(fn($acc, $value), $init)`
- Field helpers (array key, `getX()` getter, or public property): `where('suit', 'heart')`, `where('suit', ['heart','diamond'])`, `where('location', 'hand%')` (case-insensitive wildcard), `whereNot`, `whereNull`, `countWhere`, `sortBy('value', 'DESC')`, `groupBy('suit' | fn)`, `pluck('id')`, `update('visible', false)` (clones objects — does **not** persist, call `updateItems` after)
- Combine: `add($obj, ?int $id)`, `merge($other)` (this wins on key clash), `diff($remove, ?callable $eq)`, `slice($offset, $len)`, `take($n)`
- Comparisons are strict by default; `where($f, '10', strict: false)` for loose
- PHPDoc generics for IDEs/PHPStan: `/** @var Collection<Card> $hand */`

## Example — hand to client, play a card

```php
// getAllDatas(): private hand as a plain list
$result['hand'] = $this->cards->getItemsInLocation(['hand', $currentPlayerId])->values();

// act method
#[PossibleAction]
public function actPlayCard(int $cardId, int $activePlayerId): string {
    $card = $this->game->cards->getItemById($cardId);
    if ($card === null || $card->location !== 'hand' || $card->locationArg !== $activePlayerId) {
        throw new UserException(clienttranslate('Not your card'));
    }
    $this->game->cards->moveItem($card, 'table');
    $this->notify->all('cardPlayed', clienttranslate('${player_name} plays ${card_name}'), [
        'player_id' => $activePlayerId,
        'player_name' => $this->game->getPlayerNameById($activePlayerId),
        'card' => $card,                       // objects serialize to JSON with their public fields
        'card_name' => $this->game->cardName($card), 'i18n' => ['card_name'],
    ]);
    return NextPlayer::class;
}
```

## Migrating from Deck

Keep the old table by mapping the five legacy columns, then add the `order` column on running tables:

```php
#[Item('card')]
class Card {
    #[ItemField(kind: ItemFieldKind::ID, dbField: 'card_id')] public int $id;
    #[ItemField(kind: ItemFieldKind::LOCATION, locationIndex: 0, dbField: 'card_location')] public string $location;
    #[ItemField(kind: ItemFieldKind::LOCATION, locationIndex: 1, dbField: 'card_location_arg')] public ?int $location_arg;
    #[ItemField(kind: ItemFieldKind::ORDER)] public int $order;
    #[ItemField(dbField: 'card_type')] public int $type;
    #[ItemField(dbField: 'card_type_arg')] public int $type_arg;
}
// upgradeTableDb($from_version): ALTER TABLE `DBPREFIX_card` ADD `order` INT DEFAULT 0, MODIFY `card_location_arg` INT NULL;
// then copy card_location_arg into `order` for the deck rows and NULL it (see the doc page) — and
// remove the CREATE TABLE from dbmodel.sql, initDb() owns the table now.
```

Code changes: `$card['type']` → `$card->type`, and values are now **properly typed** (Deck returned strings for numeric columns) — `(int)` casts and `==` comparisons that relied on strings need a review.

## Pitfalls

- **No `dbmodel.sql` entry for managed tables** — `initDb()` creates them; declaring both breaks setup
- **`initDb()` before any query or `createItems()`**, at the beginning of `setupNewGame`; the manager itself is created in the constructor
- **Changing a property is not saved** until `updateItem()` / `updateItems()`; `Collection::update()` only touches the PHP clones
- **`->values()` before sending to the client** — a Collection keyed by id becomes a JSON object, not an array
- **`createItems()` returns nothing** and randomizes row order — query afterwards if you need the ids; never infer hidden types from creation order
- **Extra LOCATION parts must be nullable**, indexes start at 0 and are consecutive; filters (`Location::filter`, omitted parts) are read-only
- **Unregistered location = exception** — register everything up front (or use a `*` prefix wildcard)
- **Typed values**: compare with `===` against ints, not `'1'` strings as with Deck
- **Items in notifications**: pass the object (public fields serialize) — don't rebuild arrays by hand
- Pre-2026 projects keep using Deck (`references/deck.md`); don't mix both on the same table
