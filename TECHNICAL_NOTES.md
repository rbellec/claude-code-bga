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

PHP 8.4 emits deprecation warnings on patterns common in older BGA code, notably static-style calls to framework **instance** methods (`self::getPlayerNameById(...)`, `static::activeNextPlayer()`) and legacy string interpolation (`${var}`). Hence the rules: `$this->` for every framework method, `{$var}` in strings. Nuance from `_ide_helper.php`: the DB helpers (`DbQuery`, `getObjectFromDB`, `getCollectionFromDB`, …) are declared `static`, which is why the scaffold's own `static::DbQuery(...)` does not warn — but `$this->` works on static methods too, so one uniform rule costs nothing. (`self::CONST` for class constants is fine — constants are genuinely static.)

---

## Why `PlayerStats::init` takes no player id

The signature is `init(string|array $nameOrNames, $value, bool $updateTableStat = false)` — it initializes the stat for **all** players in one call. A player id passed as the third argument lands in `$updateTableStat`: under `declare(strict_types=1)` (which the skill prescribes) that is a `TypeError`; without strict types the int casts to `true`, which makes the framework also try to initialize a **table** stat with the same name and crashes `setupNewGame` if none exists. Either way the table never starts.

`inc` and `set` are the asymmetric counterparts: they **do** take a `$playerId`, and it is required.

---

## Why `reloadPlayersBasicInfos()` before INSERT/initStat in setupNewGame

The framework caches player basic infos (name, color, order) once per request. `setupNewGame` mutates the `player` table after that cache was populated — typically the color UPDATE loop. Every helper that reads the cache afterwards (`getPlayerNameById`, `loadPlayersBasicInfos`, stat initialization iterating players) would see stale data unless the cache is refreshed. Call `reloadPlayersBasicInfos()` right after the color updates, before any custom INSERT or `initStat`.

---

## Why Express-start test tables end by themselves ("abandon")

"Express start" creates a **real-time** table with a short clock — not a turn-based one. In real-time mode, BGA treats players whose client is no longer on the game page as absent, and when every player with a positive clock is absent long enough, it ends the table on its own: *"All players (with a positive clock) choosed to abandon this game"*, displayed as "End of game: Tie" with scores 0/0.

During an automated test session this is easy to trigger without noticing: navigating both hotseat players away to logaccess/manage between two checks, or switching `?testuser=` back and forth in a single tab (which briefly flags the other player offline each time). The resulting Tie 0/0 looks exactly like a scoring bug in your own code — it isn't.

**The rule (SKILL 3.8):** keep one connected client per player — two browser tabs, one per `testuser` — and don't park the game unattended mid-test. An unexpected "abandon" ending means recreate the table, not debug the game.

---

## Why creating a table can silently fail

The shared Studio test accounts can be blocked by a table that is still open — including a table from a **different** Studio project, left over from a previous session. The lobby gives no feedback: the "Create" click just does nothing.

Because there is no error message, this failure mode gets misattributed — either to the automation ("the click didn't register") or to the game code ("hotseat creation is broken"). Before either assumption, find the leftover table (browser history, the bottom-of-page banner, or `/table?table=N`) and click **"Express stop"** on its config page with `window.confirm` overridden. A `quitgame.html` ajaxcall is not sufficient for a game in progress — only Express stop actually frees the accounts.

---

## Why `performAction` must never fire from a notification handler

**Immediate reason (known):** a notification handler runs on every client,
including spectators and replay viewers. Firing a game action from it means the
action is sent again by whoever happens to be watching, and in the player's own
client it races with the action they are about to take — double-submits,
`_you can't perform this action at this moment_`, or a state advanced twice.

**Later, more expensive reason:** it is what makes a game impossible to write a
tutorial for. A BGA tutorial is not code — it is the **replay of a real game**,
annotated by a player with comment bubbles, and the reader is made to **click
the moves themselves**. The BGA *Tutorials checklist* therefore requires that
"each action of the replay should be replayable by triggering it manually", and
names the cause of failure explicitly: `ajaxcall()` / `bgaPerformAction()`
triggered programmatically from a notification handler or a callback.

The same page names two sibling causes: **several ways to trigger the same
action with different arguments** (the author reproduces one path, the archive
recorded another), and **missing `ID`s or overlapping elements** (nothing to
anchor a bubble to).

**Why it is expensive to find late:** none of these three conditions appears in
the Pre-release checklist or anywhere in the Studio UI, so a game reaches alpha,
beta and gold without them ever being checked. They surface when a volunteer
tries to write a tutorial and gives up. The fix is a **new version of the
game**, which invalidates the replay the half-written tutorial was built on —
the author has to start over, unless a *Tutorial game release override* can be
granted, and that only works if the new version stays compatible with the move
notifications of the old one.

Audit procedure: `skills/board-game-arena/references/tutorial-ready.md`.

---

## Why the generated stubs beat the wiki (and why the skill was wrong twice)

Two files land at the root of every Studio project and are refreshed by BGA on the server: `_ide_helper.php` (every PHP class, signature and `@deprecated` note) and `bga-framework.d.ts` (the typed `this.bga.*` surface). They are generated from the framework; the wiki is written by hand and lags.

The skill paid for ignoring this twice. In July 2026 a wiki-based pass *removed* the `tooltip` / `confirm` / `autoclick` options of `addActionButton` as "not documented" — they had been typed in the d.ts since at least April. The hand-written PHPStan stub in `php-code-quality.md` was wrong on most lines (enum vs class, phantom methods, wrong namespaces), so PHPStan approved code that crashed on Studio. Both were fixed in October 2026 by diffing against the stubs.

**Rule:** when the skill, the wiki and the stubs disagree, the stubs win; a running project wins over all three for behavior the stubs do not state (visibility quirks, runtime errors). The maintenance procedure is in `CLAUDE.md`.

---

## Why act methods take the player id as a magic parameter

`getCurrentPlayerId()` reads the HTTP session: it returns a **string** and throws ("Not logged") when there is no request behind the call — which is exactly the situation of `zombie()` (server-side, triggered by a timeout), `setupNewGame` and the args methods. The framework therefore autowires typed magic parameters on act methods: `int $activePlayerId` in `ACTIVE_PLAYER` states, `int $currentPlayerId` in `MULTIPLE_ACTIVE_PLAYER` / `PRIVATE` states (documented August 2026), plus the `*No` seat-number variants and `array $args` (the state's `getArgs()`).

Because the id is then an ordinary parameter, `zombie(int $playerId)` can delegate to the act method with a default move in both state types — no duplicated logic, no helper extraction. The earlier skill advice ("multiactive states must call `getCurrentPlayerId()`, so zombie cannot delegate") predates the `$currentPlayerId` documentation.

---

## Why `setupNewGame` must INSERT the player rows

At `setupNewGame` time the `player` table is **empty**: the framework hands you `$players` (id ⇒ name, avatar…) and expects the game to insert the rows, which is also where the colors are assigned. The scaffold does it with one multi-row `INSERT INTO player (player_id, player_color, player_name)`, then `reattributeColorsBasedOnPreferences()` and `reloadPlayersBasicInfos()`. An earlier version of the skill only ran `UPDATE player SET player_color` — on a fresh table that updates nothing and every later `getPlayerNameById()` fails. All real projects built with the skill had silently kept the scaffold's INSERT; the skeleton now matches them.
