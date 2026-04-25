# Scrollmap — BGA Library Reference

**Type:** JS Client (Classic Dojo)
**Doc:** https://en.doc.boardgamearena.com/Scrollmap
**When to use:** Games with large or expandable boards that need scrolling/panning (Carcassonne, Saboteur, Taluva-style).

## Setup

```javascript
const [scrollmap] = await importDojoLibs(['ebg/scrollmap']);

this.scrollmap = new ebg.scrollmap();
this.scrollmap.create(
    $('map_container'),
    $('map_scrollable'),
    $('map_surface'),
    $('map_scrollable_oversurface')
);
this.scrollmap.setupOnScreenArrows(150); // scroll distance per arrow click
```

### Required HTML

```html
<div id="map_container">
    <div id="map_scrollable"></div>
    <div id="map_surface"></div>
    <div id="map_scrollable_oversurface"></div>
    <div class="movetop"></div>
    <div class="movedown"></div>
    <div class="moveleft"></div>
    <div class="moveright"></div>
</div>
```

### Layer Model

| Layer | Scrolls | Interactive | Use for |
|-------|---------|-------------|---------|
| `map_scrollable_oversurface` | Yes | Yes (clickable) | Tokens, interactive elements |
| `map_scrollable` | Yes | No | Background tiles, board art |
| `map_surface` | No | No | Fixed overlay (grid lines) |

## API

- `create(container, undersurface, surface, onsurface)` — initialize (must call before use)
- `scroll(dx, dy, duration?, delay?)` — relative scroll (default 350ms)
- `scrollto(x, y, duration?, delay?)` — absolute scroll to coordinates
- `enableScrolling()` / `disableScrolling()` — toggle pan/scroll

## Example

```javascript
// Place a tile on the scrollable board
const tileHtml = `<div id="tile_${id}" class="board-tile" 
    style="left:${x*100}px; top:${y*100}px;"></div>`;
dojo.place(tileHtml, 'map_scrollable_oversurface');

// Scroll to center on a specific tile
this.scrollmap.scrollto(-x * 100 + 400, -y * 100 + 300);
```

## Pitfalls

- **`create()` auto-scrolls to (0,0)** with 350ms animation — wait before positioning
- **`map_surface` must match container dimensions** exactly
- **Oversurface covers surface** — if oversurface fully covers surface, panning breaks. Fix: set `pointer-events: none` on oversurface, `pointer-events: auto` on individual interactive elements
- **Mobile touch** — add `touch-action: none` to container to prevent page scroll interference
