# Deck — BGA Library Reference

**Type:** PHP Server
**Doc:** https://en.doc.boardgamearena.com/Deck
**When to use:** Any game with cards or tiles that need shuffling, drawing, hands, or discard piles.

## Setup

**Constructor** (create in constructor, NOT `initTable()`):
```php
$this->cards = $this->deckFactory->createDeck('mygame_card');
$this->cards->autoreshuffle = true;
$this->cards->autoreshuffle_trigger = ['obj' => $this, 'method' => 'onReshuffle'];
$this->cards->autoreshuffle_custom = ['deck' => 'discard']; // optional: custom source
```

**Database table** (`dbmodel.sql` — exact column names required):
```sql
CREATE TABLE IF NOT EXISTS `mygame_card` (
  `card_id` INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
  `card_type` VARCHAR(20) NOT NULL,
  `card_type_arg` INT(11) NOT NULL,
  `card_location` VARCHAR(20) NOT NULL,
  `card_location_arg` INT(11) NOT NULL,
  PRIMARY KEY (`card_id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 AUTO_INCREMENT=1;
```

## API

### Creation (setupNewGame only)
- `createCards($cards, $location='deck', $location_arg=null)` — bulk create
  Format: `[['type' => 'king', 'type_arg' => 1, 'nbr' => 12], ...]`

### Picking
- `pickCard($location, $player_id)` — 1 card to player hand
- `pickCards($nbr, $location, $player_id)` — N cards to player hand, returns picked cards
- `pickCardForLocation($from, $to, $location_arg=0)` — flexible destination
- `pickCardsForLocation($nbr, $from, $to, $location_arg=0, $no_deck_reform=false)` — batch flexible

### Moving
- `moveCard($id, $location, $location_arg=0)` — single
- `moveCards($cards, $location, $location_arg=0)` — batch (array of IDs)
- `insertCard($id, $location, $location_arg)` — insert at specific position
- `insertCardOnExtremePosition($id, $location, $bOnTop)` — top or bottom
- `moveAllCardsInLocation($from, $to, $from_arg=null, $to_arg=0)` — move all
- `moveAllCardsInLocationKeepOrder($from, $to)` — preserve order
- `playCard($id)` — shortcut: move to discard pile top

### Retrieval
- `getCard($id)` — single card array
- `getCards($array)` — multiple by IDs
- `getCardsInLocation($location, $location_arg=null, $order_by=null)` — location contents
- `getPlayerHand($player_id)` — alias for hand location
- `getCardOnTop($location)` — peek top card
- `getCardsOnTop($nbr, $location)` — peek N cards from top
- `getCardsOfType($type, $type_arg=null)` — filter by type
- `getCardsOfTypeInLocation($type, $type_arg, $location, $location_arg)` — combined filter

### Counting
- `countCardInLocation($location, $location_arg=null)` — count
- `countCardsInLocations()` — all locations summary
- `countCardsByLocationArgs($location)` — distribution by location_arg

### Shuffling
- `shuffle($location)` — randomize and reset location_arg

## Example

```php
// In setupNewGame():
$cards = [
    ['type' => 'king',   'type_arg' => 1, 'nbr' => 12],
    ['type' => 'wizard', 'type_arg' => 2, 'nbr' => 8],
];
$this->cards->createCards($cards, 'deck');
$this->cards->shuffle('deck');

// Deal 8 cards to each player
foreach ($players as $pid => $player) {
    $this->cards->pickCards(8, 'deck', $pid);
}

// In an action: play a card
$card = $this->cards->getCard($cardId);
if ($card['location'] !== 'hand' || (int)$card['location_arg'] !== $playerId) {
    throw new UserException('Not your card');
}
$this->cards->playCard($cardId);
```

## Encoding Subtypes

The Deck only has `type` (string) and `type_arg` (int). Encode subtypes in the type name:
`"guard_g1"`, `"guard_g11"`, `"jester_jM"`. Then parse:

```php
public static function parseCard(array $card): array {
    $deckType = $card['type'];
    $baseType = self::CARD_TYPE_TO_BASE[$deckType] ?? $deckType;
    $subtype = str_contains($deckType, '_') ? substr($deckType, strpos($deckType, '_') + 1) : null;
    return [
        'card_id' => $card['id'],
        'card_type' => $baseType,
        'card_value' => (int)$card['type_arg'],
        'card_subtype' => $subtype,
    ];
}
```

## Pitfalls

- **Create Deck in constructor**, not `initTable()` — otherwise references break
- **`createCards()` only in `setupNewGame()`** — never in constructor
- **`pickCards()` returns the drawn cards** — use for notifications
- **`autoreshuffle`** handles deck-empty automatically with callback
- **Increase `varchar(16)` to `varchar(20)`** if using long type names or player IDs in locations — silent truncation is hard to debug
- **Schema change = fresh table** — old tables keep old schema; quit all tables and create new ones
- **`getCardsInLocation` returns a map keyed by card_id** (not an array) — use `array_values()` before sending to JS if order matters
- **No reshuffle on peek** — `getCardOnTop()` won't trigger autoreshuffle when deck is empty
- **Multiple decks** — create separate DB tables; call `createDeck()` multiple times
- **Private info** — use `notify->player()` for hidden card distributions, not `notify->all()`
