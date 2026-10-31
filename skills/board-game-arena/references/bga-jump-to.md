# BgaJumpTo — BGA Library Reference

**Type:** JS Client (Modern ESM)
**Doc:** https://en.doc.boardgamearena.com/BgaJumpTo · API: https://x.boardgamearena.net/data/game-libs/bga-jump-to/1.x/docs/index.html · Demo: https://x.boardgamearena.net/data/game-libs/bga-jump-to/1.x/demo/index.html
**When to use:** A foldable side panel of shortcuts to the game's boards (main board, each player's table). **Only when the game has many boards that don't fit on screen** — with few elements it just pollutes the game area. Example games: Akropolis, Knarr, Path of Civilization.

## Setup

```javascript
// Top of Game.js, before any class declaration
const BgaJumpTo = await importEsmLib('bga-jump-to', '1.x');
const LOCAL_STORAGE_JUMP_TO_FOLDED_KEY = 'mygame-folded';   // remembers folded/unfolded

// In setup():
const entries = [
    new BgaJumpTo.Entry(_('Main board'), 'table-center', {
        color: '#224757',
        backgroundImage: `url('${this.bga.images.getImgUrl('board.jpg')}')`,
    }),
    ...BgaJumpTo.BgaPlayerEntries(this.bga, {           // one entry per player, in table order
        entryTarget: (playerId, player) => `player-table-${player.no}`,   // DOM id to scroll to
    }),
];
this.jumpToManager = new BgaJumpTo.Manager({
    localStorageFoldedKey: LOCAL_STORAGE_JUMP_TO_FOLDED_KEY,
    entries,
});
```

## Pitfalls

- **Targets must be stable DOM ids** that exist after `setup()` — same requirement as tutorial anchors (`references/tutorial-ready.md`)
- `BgaPlayerEntries` is customizable (label, color, target per player); keep it consistent with your player-table ids
- Semver — require `'1.x'`; TypeScript d.ts at `https://x.boardgamearena.net/data/game-libs/bga-jump-to/1.x/dist/bga-jump-to.d.ts`
