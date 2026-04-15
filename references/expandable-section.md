# ExpandableSection — BGA Library Reference

**Type:** JS Client (Classic Dojo)
**Doc:** https://en.doc.boardgamearena.com/ExpandableSection
**When to use:** Collapsible UI sections for optional info (rules summary, detailed scoring, help panels).

## Setup

```javascript
const [expandablesection] = await importDojoLibs(['ebg/expandablesection']);

this.helpSection = new ebg.expandablesection();
this.helpSection.create(this.bga.gameui, 'my_expandable');
```

### Required HTML

```html
<div id="my_expandable">
    <a href="#" class="expandabletoggle expandablearrow">
        <div class="icon20"></div>
    </a>
    <div class="expandablecontent">
        Content that can be expanded/collapsed.
    </div>
</div>
```

## API

- `create(gameui, elementId)` — initialize
- `expand()` — show content
- `collapse()` — hide content
- `toggle()` — switch state

## Pitfalls

- **Built-in click handler** — the toggle link auto-wires; no manual event needed
- **ID requirements** — inner elements need proper class names for dojo.query to find them
- **Set initial state in `setup()`** — don't rely on defaults
