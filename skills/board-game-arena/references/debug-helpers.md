# Debug Helpers — BGA Reference

**Type:** PHP + Studio UI
**When to use:** Inspecting state during dev, working around the premium gate
on the end-game stats panel, or otherwise poking the running game from the
Studio toolbar.

## The `debug_<name>` pattern

Any `public function debug_<name>(): void` declared on `Game.php` becomes
callable from the **Debug input** at the right of the Studio table. Type
`debug_<name>` (no parentheses) and hit Enter — output goes to the chat log
via `notify->all` and is visible regardless of premium status.

Useful for dumping stats mid-game, triggering a scenario without replaying
from scratch, or reading a DB table without opening the SQL log.

```php
// In Game.php — strip before release, or guard with a TEST flag.
public function debug_dumpStats(): void
{
    $msg = 'rounds: ' . $this->bga->tableStats->get('total_rounds');
    foreach ($this->loadPlayersBasicInfos() as $pid => $info) {
        $cards = $this->bga->playerStats->get('cards_played', $pid);
        $msg .= " | {$info['player_name']}: {$cards}";
    }
    // Raw string is fine here — debug helpers are dev-only, not translated.
    $this->bga->notify->all('log', $msg, []);
}
```

## Premium-gate workaround for the end-game stats panel

**Symptom:** during dev playtest, the BGA end-game stats panel displays
"Go premium to see game statistics" instead of the actual values, even when
stats are written correctly.

Two ways out:

1. Click **"Become premium"** on your Studio account — Studio grants premium
   for free in dev. Solves the panel display once for all on that account.
2. Use a `debug_dumpStats` helper (template above) — gives live values in
   the chat log and works mid-game. Useful when option 1 isn't available
   (shared/non-premium account, demo).

## Pitfalls

- **Don't ship `debug_*` to production** — they're public PHP, callable by
  any client that knows the name. Strip before tagging or guard with
  `if (!$this->isStudio()) return;`.
- **The toolbar input is silent on typos** — if nothing happens, check
  spelling and that the method is `public`.
- **Don't copy the raw-string `notify->all('log', ...)` pattern into
  player-facing notifications** — see `references/translations.md`.
