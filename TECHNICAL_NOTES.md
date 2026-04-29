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
$rows = $this->getCollectionFromDb("SELECT from_position, to_position FROM mygame_moves");
// Returns: ['0' => ['from_position'=>'0','to_position'=>'4'], '0' => ['from_position'=>'0','to_position'=>'8']]
//                                                                    ^-- overwrites the first row!
// Actual result: only one row with from_position=0 survives.
```

If two rows share the same value in the first column, the later row silently overwrites the earlier one. The array has fewer entries than expected, with no error or warning.

**Fix:** always put a unique column first:

```php
$rows = $this->getCollectionFromDb("SELECT move_number, from_position, to_position FROM mygame_moves");
// Returns: ['1' => [...], '2' => [...], '3' => [...]]  — all rows preserved
```

**Impact on graph algorithms:** if an adjacency list is built from a query missing rows, cycle detection will fail to find cycles that genuinely exist. The graph appears acyclic when it is not.

---

## Why never name a custom table `moves`

BGA creates several internal tables per game instance at table-creation time. The internal schema for a `moves` table (if it exists) has different columns than any game-specific moves table. `CREATE TABLE IF NOT EXISTS` silently succeeds without creating the correct schema — the BGA internal table already satisfies the `IF NOT EXISTS` check.

Known BGA-managed tables to avoid: `moves`, `player`, `global`, `stats`, `gamelog`.

**Fix:** prefix with game name or use unambiguous names: `mygame_moves`, `mygame_board`, `mygame_tiles`, etc.

---

## Why programmatic quit must use `gameui.ajaxcall`

BGA protects state-changing endpoints (like `quitgame.html`) with a CSRF token bound to the user session. The token is not visible from injected JavaScript, so a direct `fetch()` or `dojo.xhrPost()` is rejected.

`gameui.ajaxcall(...)`, available on game pages, automatically attaches the CSRF token from the page context, so it succeeds where raw `fetch()` fails.

Two consequences:
- The call must be issued from the **game page** (`/1/GAME_NAME?table=N`), not from the lobby. On the lobby page `gameui` is undefined; the equivalent is `mainsite`.
- It must be issued from the browser context (Claude in Chrome `javascript_tool`), not from a Node/curl script — the token lives in the page session.

If the game page is not reachable (e.g., `setupNewGame` crashes before the page loads), fall back to manually clicking Quit in the browser, or wait for BGA's auto-abandon timeout (a few minutes after repeated `setupNewGame` failures).

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
