# BGA Alpha — Technical Notes

Detailed explanations for the pitfalls listed in `SKILL.md`. Reference this file when you need to understand *why* a rule exists.

---

## Why no SQL comments in dbmodel.sql

**Root cause:** BGA Studio executes `dbmodel.sql` by stripping all newlines, then passing the result as a single string to MySQL. This means:

```sql
-- Before newline stripping:
CREATE TABLE `t` (
  `a` INT NOT NULL,   -- first column
  `b` INT NOT NULL,
  PRIMARY KEY (`a`)
) ENGINE=InnoDB;
```

becomes:

```sql
-- After newline stripping (what MySQL actually sees):
CREATE TABLE `t` (  `a` INT NOT NULL,   -- first column  `b` INT NOT NULL,  PRIMARY KEY (`a`)) ENGINE=InnoDB;
```

MySQL sees `-- first column  \`b\` INT ...` as a single-line comment, eating `b` and `PRIMARY KEY`. The table is created with only column `a`.

**This is silent** — `CREATE TABLE IF NOT EXISTS` reports success, and MySQL doesn't warn about missing columns. The first symptom is a runtime error like `Unknown column 'b' in 'field list'` when the game tries to query it.

**The fix:** write zero `--` comments anywhere in `dbmodel.sql`. If you need to document the schema, use a separate Markdown file.

---

## Why getCollectionFromDb needs a unique first column

`getCollectionFromDb($sql)` returns a PHP associative array keyed by the **first column** of the result set. This is a BGA framework behavior, not standard PDO.

```php
// This query:
$rows = $this->getCollectionFromDb("SELECT square1, square2 FROM q_moves");
// Returns: ['0' => ['square1'=>'0','square2'=>'4'], '0' => ['square1'=>'0','square2'=>'8']]
//                                                         ^-- overwrites the first row!
// Actual result: only one row with square1=0 survives.
```

If two rows share the same value in the first column, the later row silently overwrites the earlier one. The array has fewer entries than expected, with no error or warning.

**Fix:** always put a unique column first:

```php
$rows = $this->getCollectionFromDb("SELECT move_number, square1, square2 FROM q_moves");
// Returns: ['1' => [...], '2' => [...], '3' => [...]]  — all rows preserved
```

**Impact on graph algorithms:** if an adjacency list is built from a query missing rows, cycle detection will fail to find cycles that genuinely exist. The graph appears acyclic when it is not.

---

## Why never name a custom table `moves`

BGA creates several internal tables per game instance at table-creation time. The internal schema for a `moves` table (if it exists) has different columns than any game-specific moves table. `CREATE TABLE IF NOT EXISTS` silently succeeds without creating the correct schema — the BGA internal table already satisfies the `IF NOT EXISTS` check.

Known BGA-managed tables to avoid: `moves`, `player`, `global`, `stats`, `gamelog`.

**Fix:** prefix with game name or use unambiguous names: `q_moves`, `qttt_board`, `mygame_tiles`, etc.

---

## Why programmatic quit of game tables fails

BGA's table lobby uses Svelte + a CSRF token bound to the Svelte session. Actions like quitting a table require a valid CSRF token that is not accessible from injected JavaScript (`dojo.xhrPost`, `fetch`, etc.).

Consequence: you cannot quit a game table programmatically from Claude in Chrome. The user must click the Quit or "Express stop" button manually in the browser.

Workaround: BGA also auto-abandons broken tables after a timeout (a few minutes if `setupNewGame` fails repeatedly). For schema bugs, it is often faster to wait or ask the user to manually quit.

---

## How to find the correct GAME_ID for the lobby URL

There are two different numeric IDs:
- The `game_id` in the page source / JS globals — this is the **internal engine ID**, not the lobby ID
- The lobby `game=N` parameter — this is the **public game ID** used in all lobby/studio URLs

Get the lobby ID from the studiogame page:
```javascript
Array.from(document.querySelectorAll('a'))
  .find(a => a.href.match(/lobby\?game=\d+/))?.href
// → "https://studio.boardgamearena.com/lobby?game=14487"
```

---

## Table view vs game URL in BGA Studio

When navigating to a table in BGA Studio, two different pages exist:

- `/tableview?table=N` — new Svelte-based lobby view, no classic Dojo element IDs (`invite_friend`, `player_select_-99` are absent)
- `/table?table=N` — classic Dojo table page with the standard invite/hotseat UI

BGA Studio redirects `/table?table=N` → `/tableview?table=N` on the first load, but the classic URL becomes accessible after the Svelte app initializes. Navigate to `/table?table=N` for automation (invite_friend, startgame buttons).

The actual game URL (after `setupNewGame` runs) is:
`/1/GAME_NAME?table=N&testuser=PLAYER_ID`
