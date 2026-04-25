# BgaDice — BGA Library Reference

**Type:** JS Client (Modern ESM)
**Doc:** https://en.doc.boardgamearena.com/BgaDice
**When to use:** Games with dice — rendering, rolling animations, and movement between areas.

## Setup

Requires BgaAnimations as a dependency.

```javascript
const BgaAnimations = await importEsmLib('bga-animations', '1.x');
const BgaDice = await importEsmLib('bga-dice', '1.x');

this.animationManager = new BgaAnimations.Manager({
    animationsActive: () => this.bga.gameui.bgaAnimationsActive(),
});

this.diceManager = new BgaDice.Manager({
    animationManager: this.animationManager,
    type: 'my-game-die',           // CSS class prefix
});
```

## Stock Types

```javascript
// Linear display for dice
this.diceStock = new BgaDice.LineStock(this.diceManager, document.getElementById('dice-area'));
```

## API

- `await stock.addDice(dice)` — add dice to stock
- `await stock.rollDice(dice, settings)` — animate rolling

## Data Structure

Dice objects require: `{ id, face, location }`

## CSS

```css
.my-game-die[data-face="1"] { background-position: ...; }
.my-game-die[data-face="2"] { background-position: ...; }
/* etc. */
```

## Pitfalls

- **Requires BgaAnimations** — must load BgaAnimations first
- **Semantic versioning** — use `'1.x'` for stability
- **TypeScript** — `.d.ts` file available for autocomplete
