# ExpandableSection — BGA Library Reference

**Type:** JS Client (Classic Dojo)
**Doc:** https://en.doc.boardgamearena.com/ExpandableSection
**When to use:** Collapsible UI sections for optional info (rules summary, detailed scoring, help panels).

## Setup

```javascript
const [ExpandableSection] = await importDojoLibs(['ebg/expandablesection']);   // resolves to the constructor (legacy form: new ExpandableSection())

this.helpSection = new ExpandableSection();
this.helpSection.create(this.bga.gameui, 'my_expandable');
```

### Required HTML

All inner parts must have **ids** (in addition to the class names) — dojo.query needs them:

```html
<div id="my_expandable">
    <a href="#" id="my_expandable_toggle" class="expandabletoggle expandablearrow">
        <div id="my_expandable_icon" class="icon20"></div>
    </a>
    <div id="my_expandable_content" class="expandablecontent">
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
- **Every inner element needs an id** (toggle link, content div) — class names alone are not enough; without ids the component silently does nothing
- **Set initial state in `setup()`** — don't rely on defaults
