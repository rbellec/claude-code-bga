# BgaScoreSheet — BGA Library Reference

**Type:** JS Client (Modern ESM)
**Doc:** https://en.doc.boardgamearena.com/BgaScoreSheet
**When to use:** Display an animated end-game score breakdown with categories (like Wingspan, Terraforming Mars score sheets).

## Setup

```javascript
const BgaScoreSheet = await importEsmLib('bga-score-sheet', '1.x');

this.scoreSheet = new BgaScoreSheet.ScoreSheet(
    document.getElementById('score-sheet-container'),
    {
        animationsActive: () => this.bga.gameui.bgaAnimationsActive(),
        playerNameWidth: 120,
        playerNameHeight: 30,
        entryLabelWidth: 150,
        entryLabelHeight: 30,
        players: gamedatas.players,
        entries: [
            // 'property' must match the keys of the score data objects
            { property: 'buildings', label: 'Buildings' },
            { property: 'cards', label: 'Cards' },
            { property: 'bonus', label: 'Bonus' },
            { property: 'total', label: 'Total', labelClasses: 'mygame-score-total', scoresClasses: 'mygame-score-total' },
        ],
        scores: gamedatas.endScores, // null if game not finished
        classes: 'mygame-score-sheet',                    // optional CSS classes on the sheet
        onScoreDisplayed: (property, playerId, score) => {
            // called after each score cell finishes animating
        },
    }
);
```

## API

- `await scoreSheet.setScores(scoreData, options)` — display scores with animation

### Score Data Format
```javascript
{
    [playerId]: {
        buildings: 12,
        cards: 8,
        bonus: 3,
        total: 23,
    },
    // ...other players
}
```

## Example

```javascript
// In notification handler for game end:
async notif_gameEnd(args) {
    await this.scoreSheet.setScores(args.scores, {
        startBy: this.bga.players.getCurrentPlayerId(),
    });
}
```

## Pitfalls

- **Entry field is `property`, not `name`** — it must match the score-data keys exactly, or cells stay empty
- **No `isBold` option** — style via `labelClasses` / `scoresClasses` (per entry) or `classes` (whole sheet)
- **Respects replay settings** — animations auto-skip in replay mode
- **Semantic versioning** — use `'1.x'` for stability
