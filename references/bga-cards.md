# BgaCards — BGA Library Reference

**Type:** JS Client (Modern ESM)
**Doc:** https://en.doc.boardgamearena.com/BgaCards
**When to use:** Display cards with animations, selection, and movement between stocks (hands, decks, discard). Pairs with PHP Deck for full card game support.

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

// Card deck (stacked, shows count)
this.deckStock = new BgaCards.Deck(this.cardsManager, document.getElementById('deck'));

// Void (removed cards — limited animations)
this.voidStock = new BgaCards.VoidStock(this.cardsManager);
```

## API

### Adding/Removing (async — use await)
- `await stock.addCards(cards)` — add cards array to stock
- `await stock.removeCards(cards, settings)` — remove with optional animation
- `await stock.addCards(cards, { fromStock: otherStock })` — animate from another stock

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
const handCards = Object.values(gamedatas.hand); // PHP returns map, convert to array
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

- **PHP Deck returns maps, not arrays** — always `Object.values()` or `array_values()` in PHP before sending to JS
- **Fields are strings from PHP** — cast to int for sorting: `card.type_arg = parseInt(card.type_arg, 10)`
- **`addCards` is async** but `setSelectableCards` is sync — don't mix without `await`
- **`isCardVisible` is confusing** — it controls which face shows, not visibility. Default returns `card.type`; usually set to `() => true`
- **VoidStock has limited animations** — prefer `removeCards()` on other stocks
- **Requires BgaAnimations** — always load it first
- **Semantic versioning** — use `'1.x'` to get safe bug fixes
