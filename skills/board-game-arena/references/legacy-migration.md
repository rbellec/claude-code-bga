# Legacy → New Framework Migration — BGA Reference

**Type:** Process (PHP + JS + config)
**Doc:** https://en.doc.boardgamearena.com/BGA_Studio_Migration_Guide
**When to use:** Taking over a project written for the pre-2025 framework (`.game.php` at the root, `states.inc.php`, `.action.php`, Dojo-class JS) and bringing it to the new framework this skill targets — including the case of a legacy scaffold with no code worth keeping, and reactivated archived projects.

The rest of the skill describes the **target** state; this file maps each legacy construct to it. Migrate incrementally — the framework tolerates mixed setups (e.g. legacy `states.inc.php` alongside new notification API) — but finish each row before relying on its new-framework behavior.

## Legacy scaffold, nothing to migrate

An old Studio project whose scaffold contains no game logic worth keeping is **rewritten from scratch** on the new framework, not migrated. The new framework is available on old projects too — the server pushes an up-to-date `_ide_helper.php` and `bga-framework.d.ts` even to a pre-2025 project.

1. Download and archive the server scaffold into `bga_initial_code_template/` (SKILL 1.1) — do this **first** (vital on a reactivated project, see below)
2. Purge the legacy files from the server via SFTP: `GAMENAME.game.php`, `GAMENAME.action.php`, `GAMENAME.view.php`, `GAMENAME_GAMENAME.tpl`, `states.inc.php`, `stats.inc.php`, `gameoptions.inc.php`, root `material.inc.php`
3. Deploy the new-framework structure (SKILL Phases 1–2) — the table starts on the new framework from the first deploy

## Reactivated archived project: unstable first hours

Reactivating an archived Studio project may not hold: the SFTP directory — and the project's entry in the games list — can disappear within the hour following reactivation, even after a successful first deploy, while the studiogame page and running tables keep working. Symptom: `make deploy` fails with `scp: remote mkdir "GAMENAME/": Permission denied`. The **"Fix SFTP permissions"** button answers "Update file permissions ✅" but does not recreate the directory.

Recovery sequence (verified):

1. studiogame page → **"Overwrite working copy with this revision"** (head) — recreates the server-side code ("HAL says: done") but NOT the SFTP mount nor the list entry
2. `/studio` (Manage games) → **"Create a new game project"** form with the **same name** → "Project X has been reactivated!" — SFTP and the list entry come back immediately
3. The restored copy is a **mix** (old legacy files + recently deployed ones) → re-purge the legacy files and redeploy everything
4. Click **"Commit my modifications now"** (BGA's VCS) so head reflects the new structure and the next overwrite/restore no longer brings the legacy files back

## File-level changes

| Legacy | New | Note |
|---|---|---|
| `gamename.game.php` (root) | `modules/php/Game.php` | + namespace, see below |
| `gamename.action.php` | **delete** | replaced by `#[PossibleAction]` autowiring (SKILL 2.2) |
| `gamename.view.php`, `gamename_gamename.tpl` | **delete** | DOM is built in JS `setup()` |
| `states.inc.php` | one class per state in `modules/php/States/` | SKILL 2.2 pattern |
| `stats.inc.php` | `stats.json` | see `references/config-files.md` |
| `gameoptions.inc.php` | `gameoptions.json` + `gamepreferences.json` | one legacy file becomes two |
| `gameinfos.inc.php` | `gameinfos.jsonc` | see `references/gameinfos-jsonc.md` |
| `material.inc.php` (root) | `modules/php/material.inc.php` (optional) | auto-included there — never `include()` it manually |
| `gamename.js` (Dojo class) | `modules/js/Game.js` (ES6, class named `Game`) | SKILL 2.6 pattern |

## PHP — Game.php

- Head of file: `declare(strict_types=1);` + `namespace Bga\Games\GameName;` (PascalCase). Remove `require_once APP_GAMEMODULE_PATH . "module/table/table.game.php";`
- `class GameName extends Table` → `class Game extends \Bga\GameFramework\Table`
- `self::DbQuery(...)` and friends → `$this->DbQuery(...)` (PHP 8.4 — see TECHNICAL_NOTES.md)
- `define('X', …)` in material → `public const X` on `Game` (see TECHNICAL_NOTES.md)

| Legacy call | New call |
|---|---|
| `initGameStateLabels` / `get/setGameStateValue` | `$this->bga->globals->get/set/inc` (any type) — migrate values, never mix the two systems |
| `$this->notifyAllPlayers(...)` / `notifyPlayer(...)` | `$this->bga->notify->all(...)` / `->player(...)` |
| `initStat / setStat / incStat` | `$this->bga->tableStats` / `$this->bga->playerStats` (⚠ `playerStats->init` has no `$playerId` — see config-files.md) |
| `DbQuery("UPDATE player SET player_score…")` | `$this->bga->playerScore->set()` / `playerScoreAux` |
| `$this->getNew('module.common.deck')` + `init()` | Keep the table and move to **ItemManager** with `dbField:` mappings (`references/item-manager.md`, "Migrating from Deck") — or, minimal change, `$this->deckFactory->createDeck('table')` (Deck is deprecated) |
| `throw new BgaUserException(...)` | `\Bga\GameFramework\UserException` |
| `throw new BgaSystemException / BgaVisibleSystemException / feException` | `\Bga\GameFramework\SystemException` / `VisibleSystemException` (drop `FEX_NOCODE` constants) |
| `$this->isAsync()` / `isRealtime()` | `$this->bga->tableOptions->isTurnBased()` / `isRealTime()` |
| `getGameLanguage()` | `$this->bga->tableOptions->getGameLanguage()` |
| `$this->getTableOptions()` / option globals read with `getGameStateValue` | `$this->bga->tableOptions->get($id)` — `?int`, null on tables older than the option |
| `$this->default_viewport = "width=740"` (JS) | `"game_interface_width": { "min": …, "autoscale": true }` in `gameinfos.jsonc` |
| `getActivePlayerName()` / `getCurrentPlayerName()` / `getCurrentPlayerColor()` | `getPlayerNameById(...)` / `getPlayerColorById(...)` |
| `$this->gamestate->state()` / `getState()` | `getCurrentMainState()` / `getCurrentMainStateId()` |
| `totranslate()` / `$this->_()` | `clienttranslate()` |
| `getGameName()` | delete |

## PHP — states

- Each entry of the legacy `$machinestates` array becomes a State class: constructor takes **only** `id` and `type` (name/description/transitions disappear)
- `$this->gamestate->nextState('transitionName')` → act methods **return the next state class** (`return NextState::class;`)
- `setPlayerNonMultiactive($id, 'transition')` → return the state class from the act method (SKILL 2.2)
- `zombieTurn($state, $active_player)` in Game.php → `public function zombie(int $playerId): string` on each state class, returning the next state (SKILL "Zombie handling")
- Action methods move from `.action.php` + Game.php pairs to a single `#[PossibleAction]` method on the state class, with typed/attributed parameters (`IntParam`, `IntArrayParam`, `StringParam(enum:)` — see `references/php-framework.md`)

## JS — Game.js

- Dojo `declare("bgagame.gamename", ebg.core.gamegui, {...})` → `export class Game { ... }` (must be named exactly `Game`)
- `this.checkAction()` + `this.ajaxcall()` → `this.bga.actions.performAction('actName', args)`
- Manual `dojo.subscribe` per notification → `this.bga.notifications.setupPromiseNotifications()` + `notif_*` methods (see `references/notifications.md`)
- `this.*` gameui helpers move under `this.bga.*` namespaces — the Migration Guide has the full old→new table; the main moves: `addActionButton`/`updatePageTitle` → `this.bga.statusBar.*`, `setClientState`/`restoreServerGameState` → `this.bga.states.*`, `showMessage`/`confirmationDialog` → `this.bga.dialogs.*` (now Promise-based), `scoreCtrl[id]` → `this.bga.playerPanels.getScoreCounter(id)`, `this.instantaneousMode` → `!this.bga.gameui.bgaAnimationsActive()` (logic reversed)
- Dojo DOM helpers → vanilla JS (`dojo.query` → `querySelectorAll`, `dojo.addClass` → `classList.add`, `dojo.hitch(this, f)` → `f.bind(this)`, `dojo.place` → `insertAdjacentHTML`/`appendChild`, …)
- Remove `@import url(../../../css/csslayer.css);` from the CSS if present
- Classic components (`ebg.stock`, `ebg.zone`, …) still work — load via `importDojoLibs` (see their reference files); consider BgaCards for card games

## Suggested migration order

1. **Snapshot**: commit the working legacy state; download the current server files as reference
2. **Config files**: `gameinfos.jsonc`, split `gameoptions.inc.php`, `stats.json` — deploy + reload, verify the table still launches
3. **Game.php**: move + namespace + exceptions + `$this->` sweep — the game must still run on legacy states at this point
4. **States**: convert one state at a time, starting from `gameSetup`'s successor; deploy and walk the game through each converted state before the next
5. **Actions**: delete `.action.php` once every action is autowired
6. **JS**: convert the client last — the PHP side is testable through the old client during 3–5
7. Run the full Phase 3 test loop (SKILL.md) after each step

## Pitfalls

- **Never mix the two globals systems** — `initGameStateLabels` values do not appear in `$this->bga->globals` and vice versa. Migrate each global once, then delete the label.
- **Transition names die with `states.inc.php`** — any surviving `nextState('name')`/`setPlayerNonMultiactive($id,'name')` call throws `This transition (X) is impossible` once the target state is a class.
- **In-flight tables keep the old schema and state ids** — quit all tables and start fresh ones after each structural step (see SKILL "BGA Schema Changes During Development").
- **`is_beta`, `complexity/luck/strategy/diplomacy`, `designer/artist/year` have no jsonc equivalent** — re-enter them in the Studio Game Metadata Manager; don't carry them into `gameinfos.jsonc`.
- **`exception_on_warning: true`** surfaces legacy warnings (old-style interpolation, static calls) as hard errors — that's the point; fix them rather than setting it to `false`.
