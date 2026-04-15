# Draggable — BGA Library Reference

**Type:** JS Client (Classic Dojo — Legacy)
**Doc:** https://en.doc.boardgamearena.com/Draggable
**When to use:** Drag-and-drop for tile/token placement. **Legacy** — consider Pointer Events API or HTML5 Drag & Drop for new games.

## Setup

```javascript
const [draggable] = await importDojoLibs(['ebg/draggable']);

const drag = new ebg.draggable();
drag.create(this.bga.gameui, 'draggable_element_id', 'draggable_element_id');
```

## Events

```javascript
dojo.connect(drag, 'onStartDragging', this, (itemId, left, top) => {
    // drag started
});

dojo.connect(drag, 'onDragging', this, (itemId, left, top, dx, dy) => {
    // during drag
});

dojo.connect(drag, 'onEndDragging', this, (itemId, left, top, bDragged) => {
    // bDragged: true if actually moved, false if just clicked
    if (bDragged) {
        // handle drop
    }
});
```

## Pitfalls

- **Legacy component** — BGA docs explicitly recommend modern alternatives
- **Modern alternatives:**
  - **Pointer Events API** — mobile-compatible, recommended
  - **HTML5 Drag & Drop** — desktop only, has Chrome event quirks
- **No built-in drop zones** — you must implement hit-testing yourself in `onEndDragging`
