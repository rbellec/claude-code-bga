# Counter — BGA Library Reference

**Type:** JS Client (Classic Dojo)
**Doc:** https://en.doc.boardgamearena.com/Counter
**When to use:** Display numeric values (scores, resources) with animated transitions. Loaded by default in all BGA games — no import needed.

## Setup

```javascript
// No import needed — available globally
const counter = new ebg.counter();
counter.create('my_counter_div');

// With settings (modern framework):
counter.create('my_counter_div', {
    value: 0,
    tableCounter: false,
    playerCounter: false,
    playerId: null,
});
```

## API

- `create(targetId, settings?)` — bind to DOM element
- `getValue()` — current value (int)
- `incValue(delta)` — increment with animation
- `setValue(value)` — set instantly (no animation)
- `toValue(value)` — set with animation
- `disable()` — display `'-'`

### Configuration
```javascript
counter.speed = 300; // animation duration in ms (default: 100)
```

## Example

```javascript
// Per-player score counters in setup:
this.scoreCounters = {};
for (const [pid, player] of Object.entries(gamedatas.players)) {
    this.scoreCounters[pid] = new ebg.counter();
    this.scoreCounters[pid].create(`score_${pid}`);
    this.scoreCounters[pid].setValue(player.score);
}

// Update on notification:
notif_scoreUpdate(args) {
    this.scoreCounters[args.player_id].toValue(parseInt(args.newScore, 10));
}
```

## Pitfalls

- **No import needed** — Counter is auto-loaded in every BGA game
- **Notification args are strings** — always `parseInt(value, 10)` before arithmetic
- **Works with PlayerCounter/TableCounter** — pass `playerCounter`/`tableCounter` in settings for auto-sync
