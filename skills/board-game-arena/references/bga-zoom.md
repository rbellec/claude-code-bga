# BgaZoom — BGA Library Reference

**Type:** JS Client (Modern ESM)
**Doc:** https://en.doc.boardgamearena.com/BgaZoom · API: https://x.boardgamearena.net/data/game-libs/bga-zoom/1.x/docs/index.html · Demo: https://x.boardgamearena.net/data/game-libs/bga-zoom/1.x/demo/index.html
**When to use:** Zoom −/+ controls on the game area plus an **auto-zoom** that shrinks a wide fixed layout on load so it fits the screen. Alternative to the framework's `game_interface_width.autoscale` (gameinfos) when you want player-controlled zoom. Example game: Azul: Queen's Garden.

## Setup

```javascript
// Top of Game.js, before any class declaration
const BgaZoom = await importEsmLib('bga-zoom', '1.x');
const LOCAL_STORAGE_ZOOM_KEY = 'mygame-zoom';   // remembers the player's zoom on their device

// In setup():
this.zoomManager = new BgaZoom.Manager({
    element: document.getElementById('full-table'),   // the element that gets zoomed
    zoomControls: { color: 'black' },
    localStorageZoomKey: LOCAL_STORAGE_ZOOM_KEY,
    autoZoom: {
        expectedWidth: 1600,   // width the layout needs; zoomed out on load until it fits
        minZoomLevel: 0.25,
    },
});
```

## Pitfalls

- **Scaling uses CSS `zoom`** — legacy dojo animations (`slideToObject`…) break inside the zoomed element; BgaAnimations works. `getBoundingClientRect()` is unreliable inside a zoomed element; the framework's `getBoundingClientRectIgnoreZoom(el)` only compensates the framework autoscale, so see the lib's docs/demo for geometry inside a BgaZoom element
- **Don't stack it on framework autoscale** — if you manage scaling with BgaZoom, set `"game_interface_width": { "autoscale": false }` in `gameinfos.jsonc` (the doc names BgaZoom as the legitimate reason to disable autoscale)
- Handles light/dark theme since 1.0.1; semver — require `'1.x'`
- TypeScript: d.ts at `https://x.boardgamearena.net/data/game-libs/bga-zoom/1.x/dist/bga-zoom.d.ts`
