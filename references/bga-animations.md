# BgaAnimations — BGA Library Reference

**Type:** JS Client (Modern ESM)
**Doc:** https://en.doc.boardgamearena.com/BgaAnimations
**When to use:** Animate element movement, scoring popups, scale/rotate transitions. Required dependency for BgaCards and BgaDice.

## Setup

```javascript
const BgaAnimations = await importEsmLib('bga-animations', '1.x');

this.animationManager = new BgaAnimations.Manager({
    animationsActive: () => this.bga.gameui.bgaAnimationsActive(),
});
```

## API

- `await animationManager.slideAndAttach(element, destination)` — animate element to target and reparent
- `await animationManager.displayScoring(element, score, color)` — show scoring animation popup

## Example

```javascript
// Move a meeple to a new board position
async notif_moveMeeple(args) {
    const meepleDiv = document.getElementById(`meeple-${args.playerId}`);
    const targetDiv = document.getElementById(`slot-${args.slot}`);
    await this.animationManager.slideAndAttach(meepleDiv, targetDiv);
}

// Display scoring animation
async notif_score(args) {
    const targetDiv = document.getElementById(`player-score-${args.playerId}`);
    await this.animationManager.displayScoring(targetDiv, `+${args.points}`, '#00ff00');
}
```

## Pitfalls

- **Use CSS class selectors for animated elements** — not ID or parent-based selectors. Animations may clone elements, causing visual glitches with non-class selectors.
- **Always check `bgaAnimationsActive()`** — the Manager constructor takes this callback and skips animations when disabled
- **Promise-based** — all methods return promises; use `await` for sequencing
- **Required by BgaCards and BgaDice** — always load this first when using those libraries
- **Semantic versioning** — use `'1.x'` for stability
