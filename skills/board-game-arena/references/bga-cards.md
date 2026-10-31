# BgaCards — BGA Library Reference

**Type:** JS Client (Modern ESM)
**Doc:** https://en.doc.boardgamearena.com/BgaCards
**When to use:** Display cards with animations, selection, and movement between stocks (hands, decks, discard). Pairs with PHP ItemManager (`references/item-manager.md`; Deck on legacy projects) for full card game support. The `importEsmLib` lines go at the top of `Game.js`, before any class declaration.

## Setup

Requires BgaAnimations as a dependency.

```javascript
// In constructor or setup:
const BgaAnimations = await importEsmLib('bga-animations', '1.x');
const BgaCards = await importEsmLib('bga-cards', '1.x');

this.animationManager = new BgaAnimations.Manager({
    animationsActive: () => this.bga.gameui.bgaAnimationsActive(),
});

this.cardsManager = new BgaCards.Manager({
    animationManager: this.animationManager,
    type: 'mygame-card',          // CSS class prefix
    getId: (card) => card.id,
    setupFrontDiv: (card, div) => {
        // Set card face appearance
        div.dataset.type = card.type;
        div.dataset.value = card.type_arg;
    },
    setupBackDiv: (card, div) => {
        // Set card back appearance (optional)
    },
    isCardVisible: (card) => true, // true = show front, false = show back
});
```

## Stock Types

```javascript
// Linear arrangement (market, tableau)
this.tableStock = new BgaCards.LineStock(this.cardsManager, document.getElementById('table'));

// Player hand (fan layout)
this.handStock = new BgaCards.HandStock(this.cardsManager, document.getElementById('hand'));

// Grid layout
this.gridStock = new BgaCards.GridStock(this.cardsManager, document.getElementById('grid'));

// Fixed named slots (board positions, tableau columns) — element must start empty
this.boardStock = new BgaCards.SlotStock(this.cardsManager, document.getElementById('board'), {
    slotsIds: ['slot1', 'slot2', 'slot3'],                 // required (numbers or strings)
    mapCardToSlot: (card) => `slot${card.location_arg}`,   // required — auto-places cards
    slotClasses: ['mygame-slot'],                          // optional
    direction: 'row',                                      // 'row' (default) | 'column'
});

// Manual positioning — you compute each card's coordinates yourself
this.mapStock = new BgaCards.ManualPositionStock(this.cardsManager,
    document.getElementById('map'), {},
    (element, cards, lastCard, stock) => { /* set each card div's left/top */ });

// Card deck (stacked, shows count)
this.deckStock = new BgaCards.Deck(this.cardsManager, document.getElementById('deck'));

// Void (removed cards — limited animations)
this.voidStock = new BgaCards.VoidStock(this.cardsManager);

// Also available: DiscardDeck, AllVisibleDeck, ScrollableStock, CardCounter
```

## API

### Adding/Removing (async — use await)
- `await stock.addCards(cards)` / `await stock.addCard(card)` — add to stock
- `await stock.removeCards(cards, settings)` / `removeCard(card)` — remove; animation settings incl. `fadeOut`, `slideTo`
- `await stock.addCards(cards, { fromStock: otherStock })` — animate from another stock
- SlotStock: `await stock.addCard(card, { slot: 'slot2' })` — overrides `mapCardToSlot`; `setSlotsIds(ids)` re-creates slots (**empties the stock**), `addSlotsIds(ids)` adds without touching existing ones

Card objects require `{ id, type, type_arg, location, location_arg }`.

### Selection (sync)
- `stock.setSelectableCards(cards)` — define which cards can be selected
- `stock.getSelection()` — get currently selected cards
- `stock.unselectAll()` — clear selection

### Events
```javascript
stock.onSelectionChange = (selection, lastChange) => {
    // selection: array of selected cards
    // lastChange: the card that was just selected/unselected
};
```

## Example

```javascript
// In setup(gamedatas):
this.handStock = new BgaCards.LineStock(this.cardsManager, document.getElementById('my-hand'));
const handCards = gamedatas.hand; // keyed objects accepted since bga-cards 1.0.14 (send ->values() from PHP anyway)
await this.handStock.addCards(handCards);

// Playing a card (in notification handler):
async notif_cardPlayed(args) {
    const card = args.card;
    await this.tableStock.addCards([card], { fromStock: this.handStock });
}
```

## CSS

Cards get CSS classes based on `type` from the Manager config:
```css
.mygame-card-front[data-type="king"] {
    background-position: -0px -0px;
}
.mygame-card-front[data-value="2"] {
    /* sprite position based on value */
}
```

## Pitfalls

- **Keyed objects are accepted by `addCards` since 1.0.14**, but numeric fields sent by PHP Deck arrive as **strings** — `BgaCards.sort` and your comparisons expect ints (ItemManager objects are properly typed). Send `->values()` / `array_values()` lists anyway so JSON keeps the order
- **Fields are strings from PHP** — cast to int for sorting: `card.type_arg = parseInt(card.type_arg, 10)`
- **`addCards` is async, `setSelectableCards` is sync** — if cards are added in `setup()` and selected in `onEnteringState()`, the selection can run before the cards exist. You can't `await` the sync call; the documented workaround is to promisify with `setTimeout` before selecting
- **`setSlotsIds` empties the stock** — use `addSlotsIds` to extend without losing cards
- **`isCardVisible` is confusing** — it controls which face shows, not visibility. Default returns `card.type`; usually set to `() => true`
- **VoidStock has limited animations** — prefer `removeCards()` on other stocks
- **Requires BgaAnimations** — always load it first
- **Semantic versioning** — use `'1.x'` to get safe bug fixes
