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

## Why custom tables must use a game-specific prefix

BGA creates its own internal tables for every new game instance, **before** `dbmodel.sql` runs. If your `dbmodel.sql` declares a table that collides with one of these names, `CREATE TABLE IF NOT EXISTS` reports success but creates nothing — the BGA-managed table already satisfies the `IF NOT EXISTS` check, so your columns are silently dropped on the floor. The first symptom is a runtime error like `Unknown column 'X' in 'field list'`.

Known reserved names (non-exhaustive): `moves`, `player`, `global`, `stats`, `gamelog`, `replaysavepoint`, plus anything starting with `bga_` (`bga_globals`, `bga_user_preferences`, `bga_player_counters`, …).

**Fix:** prefix every custom table with a game-specific tag — `mygame_moves`, `mygame_board`, `mygame_tiles`, etc.

## Why prefer `public const` over `define()` for game constants

The new BGA framework puts game code under a namespace (`Bga\Games\GameName`). PHP `define()` registers constants in the **global** namespace, which makes them invisible from inside the namespace unless qualified with a leading backslash. Class constants declared with `public const` live on the class itself and are reached via `self::MY_CONST` (inside Game.php) or `Game::MY_CONST` (from State classes), with no namespace gymnastics.

A second benefit: class constants are typed and IDE-autocompleted, while `define()` produces untyped globals.

**Fix:** declare game constants as `public const` on the `Game` class. Reserve `material.inc.php` for static tabular data (tile costs, deck contents) — and even there, prefer arrays of class constants when feasible.

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
