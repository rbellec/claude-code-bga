# Stock — BGA Library Reference

**Type:** JS Client (Classic Dojo)
**Doc:** https://en.doc.boardgamearena.com/Stock
**When to use:** Display sets of game elements (cards, tokens, tiles) with sprite-based rendering, selection, and animation. Classic alternative to BgaCards — use BgaCards for new card games instead.

## Setup

```javascript
const [stock] = await importDojoLibs(['ebg/stock']);

this.playerHand = new ebg.stock();
this.playerHand.create(this.bga.gameui, $('myhand'), cardWidth, cardHeight);
this.playerHand.image_items_per_row = 13; // columns in sprite sheet
this.playerHand.setSelectionMode(1);      // 0=none, 1=single, 2=multiple

// Register item types (one per card type)
this.playerHand.addItemType(typeId, weight, g_gamethemeurl + 'img/cards.png', spritePosition);
```

## API

### Adding/Removing
- `addToStock(type, from?)` — add generic item (no unique ID)
- `addToStockWithId(type, id, from?)` — add tracked item
- `removeFromStock(type, to?, noupdate?)` — remove by type
- `removeFromStockById(id, to?, noupdate?)` — remove by ID
- `removeAll()` — clear all
- `removeAllTo(to)` — clear with animation to target

### Selection
- `setSelectionMode(mode)` — 0=none, 1=single, 2=multiple
- `setSelectionAppearance(type)` — `'border'` | `'disappear'` | `'class'`
- `getSelectedItems()` — returns `[{type, id}, ...]`
- `selectItem(id)`, `unselectItem(id)`, `unselectAll()`
- `isSelected(id)` — boolean

### Query
- `count()` — total items
- `getAllItems()` — all items array
- `getItemById(id)` — single item or null
- `getItemDivId(id)` — DOM element ID string
- `getPresentTypeList()` — type count map

### Display
- `updateDisplay(from?)` — refresh layout
- `changeItemsWeight(newWeights)` — reorder: `{typeId: weight}`
- `setOverlap(hPercent, vPercent)` — overlap items
- `resizeItems(w, h, bgW, bgH)` — resize

## Events

```javascript
dojo.connect(this.playerHand, 'onChangeSelection', this, 'onHandSelection');

onHandSelection(controlName, itemId) {
    const items = this.playerHand.getSelectedItems();
    if (items.length > 0) {
        bga.actions.performAction('actPlayCard', { id: items[0].id });
        this.playerHand.unselectAll();
    }
}

// Custom element setup
this.playerHand.onItemCreate = dojo.hitch(this, 'setupCard');
setupCard(cardDiv, typeId, cardId) {
    // add tooltips, custom HTML, etc.
}
```

## Example

```javascript
// In setup(gamedatas):
for (const card of Object.values(gamedatas.hand)) {
    this.playerHand.addToStockWithId(card.type_id, card.id);
}

// Moving between stocks:
const fromDivId = this.tableStock.getItemDivId(card.id);
this.playerHand.addToStockWithId(card.type_id, card.id, fromDivId);
this.tableStock.removeFromStockById(card.id);
```

## Pitfalls

- **Inter-stock movement:** add to destination FIRST (with `from` = source div ID), THEN remove from source
- **Safari sprite alignment:** specify `background-size` explicitly (e.g., `"2800% 500%"` for 28x5 grid)
- **Batch removals:** pass `noupdate=true`, then call `updateDisplay()` once
- **Weight sorting:** identical weights = unpredictable order — assign unique weights
- **Prefer BgaCards for new projects** — Stock is classic/legacy, BgaCards has better animation support
