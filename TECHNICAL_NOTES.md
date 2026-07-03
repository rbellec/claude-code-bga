# BGA Skill — Technical Notes

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

Three different pages exist around a table in BGA Studio:

- `/tableview?table=N` — Svelte-based table/spectator view; no `game_play_area`, cannot play
- `/table?table=N` — classic table configuration page; this is where the "Express start" button lives
- `/1/GAME_NAME?table=N` — the actual game (player view, once `setupNewGame` has run); add `&testuser=PLAYER_ID` to play as another hotseat player

The automated flow (SKILL.md §3.4–3.5) is: lobby → "Play with friends" (lands on `/table?table=N`) → "Express start" → BGA redirects to `/tableview` → navigate explicitly to `/1/GAME_NAME?table=N` to reach the player view. The redirect to `/tableview` after Express start is why the navigation step to the player view is always needed.

---

## Why PHP warnings corrupt the JSON response (always `$this->`, never `self::`/`static::`)

BGA action endpoints answer with a raw JSON body. PHP prints warnings and deprecation notices directly into the HTTP output, so any warning emitted during a request lands in front of the JSON — the client then fails with `JSON_ERROR_SYNTAX` even though the server-side action usually completed. The game UI breaks with no obvious server error.

PHP 8.4 emits deprecation warnings on patterns common in older BGA code, notably static-style calls to framework instance methods (`self::DbQuery(...)`, `static::DbQuery(...)`) and legacy string interpolation (`${var}`). Hence the rules: `$this->` for every instance method, `{$var}` in strings. (`self::CONST` for class constants is fine — constants are genuinely static.)

---

## Why `PlayerStats::init` takes no player id

The signature is `init($name, $value, bool $updateTableStat = false)` — it initializes the stat for **all** players in one call. Passing a player id as the third argument silently casts the int to `true`, which makes the framework also try to initialize a **table** stat with the same name. If no such table stat exists, `setupNewGame` crashes.

`inc` and `set` are the asymmetric counterparts: they **do** take a `$playerId`, and it is required.

---

## Why `reloadPlayersBasicInfos()` before INSERT/initStat in setupNewGame

The framework caches player basic infos (name, color, order) once per request. `setupNewGame` mutates the `player` table after that cache was populated — typically the color UPDATE loop. Every helper that reads the cache afterwards (`getPlayerNameById`, `loadPlayersBasicInfos`, stat initialization iterating players) would see stale data unless the cache is refreshed. Call `reloadPlayersBasicInfos()` right after the color updates, before any custom INSERT or `initStat`.
