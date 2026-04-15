# Zone — BGA Library Reference

**Type:** JS Client (Classic Dojo)
**Doc:** https://en.doc.boardgamearena.com/Zone
**When to use:** Organize game elements (tokens, meeples) within board areas with automatic positioning patterns.

## Setup

```javascript
const [zone] = await importDojoLibs(['ebg/zone']);

this.myZone = new ebg.zone();
this.myZone.create(this.bga.gameui, 'zone_div_id', itemWidth, itemHeight);
this.myZone.setPattern('grid'); // or 'diagonal', 'verticalfit', 'horizontalfit', 'ellipticalfit', 'custom'
```

HTML container must have a fixed width:
```html
<div id="zone_div_id" style="width: 200px;"></div>
```

## API

- `placeInZone(objectId, weight?)` — add element with optional sort weight
- `removeFromZone(objectId, destroy?, to?)` — remove; destroy=true removes DOM element; `to` = animate to target
- `removeAll()` — clear and destroy all items
- `getItemNumber()` — count of items
- `getAllItems()` — array of item IDs

## Pattern Modes

| Pattern | Behavior |
|---------|----------|
| `grid` | Horizontal wrap (default) |
| `diagonal` | Stacked with `item_margin` offset (5px default) |
| `verticalfit` | Single column; overlaps if space constrained |
| `horizontalfit` | Single row; overlaps if space constrained |
| `ellipticalfit` | Circular arrangement; concentric if overcrowded |
| `custom` | Manual positioning via `itemIdToCoords()` |

### Custom Pattern

```javascript
this.myZone.setPattern('custom');
this.myZone.itemIdToCoords = function(i, controlWidth) {
    return { x: (i % 4) * 60, y: Math.floor(i / 4) * 40, w: 50, h: 30 };
};
```

## Example

```javascript
// Place tokens in a board area
for (const token of gamedatas.tokens) {
    dojo.place(`<div id="token_${token.id}" class="my-token"></div>`, 'game_play_area');
    this.myZone.placeInZone(`token_${token.id}`, token.weight);
}

// Move token out
this.myZone.removeFromZone(`token_${tokenId}`, false, 'target_div');
```

## Pitfalls

- **Fixed width required** — the zone container needs explicit CSS width
- **Diagonal pattern** doesn't respect boundaries — items can overflow
- **Consider Stock for responsive layouts** — Zone is simpler but less flexible
