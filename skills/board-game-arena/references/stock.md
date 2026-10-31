# Stock — BGA Library Reference

**Type:** JS Client (Classic Dojo) — **DEPRECATED (October 2026)**
**Doc:** https://en.doc.boardgamearena.com/Stock
**When to use:** Only on a project that already uses it — the doc page is *"only kept here for reference of old projects using it, but don't use it for new projects."* New games display cards/tokens with BgaCards (`references/bga-cards.md`).

## Setup

```javascript
const [Stock] = await importDojoLibs(['ebg/stock']);   // resolves to the constructors (d.ts example)

this.playerHand = new Stock();                          // wiki legacy form: new ebg.stock()
this.playerHand.create(this.bga.gameui, $('myhand'), cardWidth, cardHeight);
this.playerHand.image_items_per_row = 13; // columns in sprite sheet
this.playerHand.setSelectionMode(1);      // 0=none, 1=single, 2=multiple

// Register item types (one per card type)
this.playerHand.addItemType(typeId, weight, this.bga.images.getImgUrl('cards.png'), spritePosition);
```

## API

### Adding/Removing
- `addToStock(type)` — add generic item (no unique ID)
- `addToStockWithId(type, id, from?)` — add tracked item
- `removeFromStock(type, to?, noupdate?)` — remove by type (wiki-only method, not in the d.ts `Stock` class)
- `removeFromStockById(id, to?)` — remove by ID
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
dojo.connect(this.playerHand, 'onChangeSelection', this, 'onHandSelection'); // legacy 4-arg object form

onHandSelection(controlName, itemId) {
    const items = this.playerHand.getSelectedItems();
    if (items.length > 0) {
        this.bga.actions.performAction('actPlayCard', { id: items[0].id });
        this.playerHand.unselectAll();
    }
}

// Custom element setup
this.playerHand.onItemCreate = dojo.hitch(this, 'setupCard');
setupCard(itemDiv, typeId, itemDivId) {   // 3rd param = the item's DOM id, not the card id
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
