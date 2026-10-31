# JS Framework — BGA Reference

**Type:** JS Client (ES6 Module)
**Doc:** https://en.doc.boardgamearena.com/Game_interface_logic:_yourgamename.js
**When to use:** Full API reference for building the game client beyond the basics in SKILL.md.

## Lifecycle

```javascript
constructor(bga)                       // receives the `bga` facade: this.bga = bga; globals, register state classes
setup(gamedatas)                       // build UI, connect events, setup notifications
onEnteringState(stateName, args)       // switch-style alternative to JS state classes; state args at args.args (may be null);
                                       // also called for the current state after setup() on load / F5
onLeavingState(stateName)              // cleanup
onUpdateActionButtons(stateName, args) // args IS the getArgs() result (null for GAME states); called before onEnteringState;
                                       // in multiactive states called on every activation change (REQUIRED there)
onScreenWidthChange()                  // resize hook, also fired at load — pair with getBoundingClientRectIgnoreZoom(el) under autoscale
```

## State Management

```javascript
// JS State Classes — the scaffold's default pattern (alternative: a switch in
// Game.onEnteringState). One class per PHP state, registered in the constructor.
class PlayerTurn {
    constructor(game, bga) { this.game = game; this.bga = bga; }
    onEnteringState(args, isCurrentPlayerActive) { }          // args = PHP getArgs()
    onLeavingState(args, isCurrentPlayerActive) { }
    onPlayerActivationChange(args, isCurrentPlayerActive) { } // multiactive: call it yourself at the end of onEnteringState
    // this.args is filled automatically with the current state args
}
this.bga.states.register('PlayerTurn', new PlayerTurn(this, this.bga)); // name or numeric id
this.bga.states.logger = console.log; // debug — remove before production

// Accessors
this.bga.states.getStateClass('PlayerTurn');
this.bga.states.getCurrentMainStateName();    // ignores private states
this.bga.states.getCurrentPlayerStateName();  // private state name if any
this.bga.states.getCurrentMainStateClass();   // instance
this.bga.states.getCurrentPlayerStateClass(); // instance

// Client states (multi-step interactions without server)
this.bga.states.setClientState('client_selectTarget', {
    descriptionmyturn: _('${you} must select a target'),
});
this.bga.states.restoreServerGameState(); // cancel client state
this.bga.states.isOnClientState();        // check
```

## Player Information

```javascript
this.bga.players.getCurrentPlayerId()        // viewing player's ID
this.bga.players.isCurrentPlayerActive()      // is it this player's turn?
this.bga.players.isCurrentPlayerSpectator()   // read-only viewer
this.bga.players.getFormattedPlayerName(pid)  // HTML with color
this.bga.players.getPlayerById(pid)           // player data object (also getCurrentPlayer, getActivePlayer)
this.bga.players.getActivePlayerId()          // single active player
this.bga.players.getActivePlayerIds()         // array (multiactive)
this.bga.players.getPlayerAvatarUrl(pid, 50)  // sizes: 32, 50, 92, 184
this.bga.players.isPlayerActive(pid)

// gamedatas.players[pid] typed fields: id (string), name, color, score (string), is_ai (string),
// beginner (boolean), eliminated, zombie (numbers); gamedatas.playerorder: number[]; gamedatas.gamestate.args

// Player "no" (seat number, 1-based turn order) ↔ id — pairs with the PHP useNo APIs
this.bga.players.getCurrentPlayerNo()         // null for spectators
this.bga.players.getActivePlayerNo()
this.bga.players.getPlayerByNo(no)
this.bga.players.getPlayerNoById(pid)
this.bga.players.getPlayerIdByNo(no)

// Read-only detection (spectator, replay, archive)
const isReadOnly = this.bga.players.isCurrentPlayerSpectator()
    || typeof g_replayFrom !== 'undefined' || g_archive_mode;
```

## DOM Manipulation

```javascript
// Element access
$(elementId)                              // get by ID (BGA shortcut)
document.querySelector('.selector')       // first match
document.querySelectorAll('.selector')    // all matches

// Creation & placement
dojo.place('<div id="token_1" class="token"></div>', 'parent_div');
dojo.place(htmlString, refNode, 'before'|'after'|'first'|'last'|'only');
dojo.create('div', { id: 'foo', class: 'bar' }, parentNode);
dojo.empty('container_id');               // remove all children
dojo.destroy('element_id');               // remove element

// Positioning
this.bga.gameui.placeOnObject('mobile', 'target');          // instant
this.bga.gameui.placeOnObjectPos('mobile', 'target', x, y); // with offset
const el = this.bga.gameui.attachToNewParent('mobile', 'newParent'); // reparent — returns the NEW node, old ref invalid

// Templates
this.bga.gameui.format_string(_('${player_name} plays ${card}'), args);
// Hook to rewrite every log line before display (icons for ${card} etc.) — must return { log, args }
bgaFormatText(log, args) { /* edit log/args */ return { log, args }; }
```

## Animations (legacy dojo — prefer BgaAnimations for new code)

```javascript
// All of these return a DojoAnimation ({ play() }). Await it with bgaPlayDojoAnimation —
// there is no `.promise` property.
const anim = this.bga.gameui.slideToObject('mobile', 'target', duration, delay);
await this.bga.gameui.bgaPlayDojoAnimation(anim);        // plays it and resolves at the end
this.bga.gameui.slideToObjectPos('mobile', 'target', x, y, duration, delay);
this.bga.gameui.slideTemporaryObject(html, parent, from, to, duration, delay);
this.bga.gameui.slideToObjectAndDestroy('mobile', 'target', duration, delay);
this.bga.gameui.fadeOutAndDestroy('element', duration, delay);

// Rotation
this.bga.gameui.rotateTo('element', degrees);            // starts instantly, additive
this.bga.gameui.rotateInstantTo('element', degrees);     // instant

// Wait (respects animation settings)
await this.bga.gameui.wait(500);

// Check before animating
if (this.bga.gameui.bgaAnimationsActive()) { /* animate */ }
```

## Action Buttons

```javascript
// In onUpdateActionButtons:
const btn = this.bga.statusBar.addActionButton(      // returns the HTMLButtonElement
    _('Play'),                                           // label
    () => this.bga.actions.performAction('actPlay'),     // callback (function, not string)
    {
        color: 'primary',    // 'primary' (blue), 'secondary' (gray), 'alert' (red)
        id: 'playBtn',       // DOM id
        classes: 'mygame-btn', // extra CSS classes
        disabled: false,     // grayed out
        title: _('Play your selected card'), // plain-text label description (a11y, for icon labels)
        tooltip: _('Plays the card and ends your turn'), // tooltip on the button
        confirm: _('Play this card?'),  // confirm dialog before the callback (string, or () => string|null to decide at click time)
        autoclick: false,    // true | { abortSignal, pausable } — auto-click after a short delay (confirmation buttons)
        // destination: element — place the button elsewhere than the status bar
    }
);  // returns the HTMLButtonElement
// (The options above are those typed in bga-framework.d.ts — the wiki page lags behind it.)

// Status bar title / reset
this.bga.statusBar.setTitle(_('${you} must choose a card'), {});
this.bga.statusBar.removeActionButtons();
```

## Server Actions

```javascript
// ONLY call from user events (onclick, onchange) — NEVER from notifications/loops/callbacks
// Returns the ajax Promise, or UNDEFINED when checkAction/checkPossibleActions blocked the call
// (so `await` is safer than chaining .then on it). Rejection value: { message, args }.
this.bga.actions.performAction('actPlayCard', { id: cardId })
    ?.then(() => { /* success */ })
    .catch(() => { /* error */ });

// Options
this.bga.actions.performAction('actPlay', args, {
    lock: true,                    // lock UI during call (default true)
    checkAction: true,             // verify action possible (default true)
    checkPossibleActions: false,   // ignore lock/active (for multiactive "change mind")
    ignoreDefaultErrorHandler: false, // true = handle the rejection yourself (no red banner)
});

// Don't name args after the PHP magic parameters (args, activePlayerId,
// currentPlayerId + No/snake_case variants — see php-framework.md), nor
// after the reserved query params: action, module, class
// Arrays supported: { ids: [1, 2, 3] }
// JSON strings: { data: JSON.stringify(obj) }
```

## Tooltips

```javascript
this.bga.gameui.addTooltip('element_id', _('Help text'), _('Action text'), delay = 400); // '' for an unused part
this.bga.gameui.addTooltipHtml('element_id', '<div>Rich HTML</div>', delay = 400);
this.bga.gameui.addTooltipToClass('card', _('A card'), '', delay = 400);
this.bga.gameui.addTooltipHtmlToClass('card', htmlContent, delay = 400);
this.bga.gameui.removeTooltip('element_id');
// All tooltip targets MUST have unique IDs
```

## Dialogs

```javascript
// Confirmation
this.bga.dialogs.confirmation(_('Are you sure?')).then(result => {
    if (result) { /* proceed */ }
});

// Multiple choice — choices is a map value => label; resolves with the KEY (string) or null
this.bga.dialogs.multipleChoice(_('How many?'), { 0: '0', 1: '1', 5: '5' }).then(choice => {
    if (choice !== null) { /* handle parseInt(choice, 10) */ }
});

// Message
this.bga.dialogs.showMessage(_('Important announcement'), 'info'); // 'info' | 'error' (also logged) | 'only_to_log'
this.bga.dialogs.showMoveUnauthorized(); // standard "not allowed now" error

// Info popin (Ok button) — resolves when closed
await this.bga.dialogs.info(_('The deck is empty: last round'), _('Last round'));

// Generic HTML dialog — resolves when hidden/destroyed
await this.bga.dialogs.generic(_('Scoring details'), '<div id="scoring-table"></div>', {
    id: 'mygame-scoring',        // auto-generated if omitted
    beforeShow: (dialog) => { /* populate #scoring-table */ },
    maxWidth: 800,
    hideCloseIcon: false,
    closeOnOverlayClick: true,   // forced to false when hideCloseIcon is true
});

// Scoring popup
this.bga.gameui.displayScoring('anchor_id', 'ff0000', '+10', 1000);

// Speech bubble
this.bga.gameui.showBubble('meeple_id', _('Hello!'), 0, 1000);
```

## Banners

```javascript
this.bga.gameArea.addLastTurnBanner(_('Last turn! (deck empty)'));
this.bga.gameArea.removeLastTurnBanner();

this.bga.gameArea.addWinConditionBanner(_('${player_name} wins!'), {
    player_name: this.bga.players.getFormattedPlayerName(pid),
});
```

## Player Panels

```javascript
// Add content
const panel = this.bga.playerPanels.getElement(playerId);
panel.insertAdjacentHTML('beforeend', '<div>Custom content</div>');

// Score counter
const counter = this.bga.playerPanels.getScoreCounter(playerId);

// Same, addressed by player no (seat number) instead of id
this.bga.playerPanels.getElementByNo(playerNo);
this.bga.playerPanels.getScoreCounterByNo(playerNo);

// Automata (AI) player
this.bga.playerPanels.addAutomataPlayerPanel(0, 'Bot', { score: 0 });

// Enable/disable
this.bga.gameui.disablePlayerPanel(pid);
this.bga.gameui.enablePlayerPanel(pid);
this.bga.gameui.enableAllPlayerPanels();
```

## Images & Sounds

```javascript
// Images
this.bga.images.dontPreloadImage('unused.png');         // skip root images
this.bga.images.preloadImages(['expansion/cards.png']); // load non-root
const url = this.bga.images.getImgUrl('board.jpg');     // get URL
// g_gamethemeurl + 'img/file.svg' still works but is @deprecated in favor of getImgUrl()

// Sounds: drop name.ogg / name.mp3 in the game's `sounds/` folder (autoloaded), then
this.bga.sounds.play('click');                          // name without extension
this.bga.sounds.dontPreloadSounds(['rare_sound']); this.bga.sounds.preloadSounds(['rare_sound']);
this.bga.gameui.disableNextMoveSound(); // suppress default move sound
```

## User Preferences

```javascript
const pref = this.bga.userPreferences.get(100); // preference ID from gamepreferences.json
this.bga.userPreferences.set(100, 2);
this.bga.userPreferences.toggleVisibility(100, false); // hide a pref from the menu (e.g. irrelevant with this option)
this.bga.userPreferences.onChange = (prefId, value) => { /* react live */ };
```

## Table Options (client side)

```javascript
this.bga.tableOptions.get(100);          // value of a gameoptions.json option
this.bga.tableOptions.isTurnBased();
this.bga.tableOptions.isRealTime();
this.bga.tableOptions.getGameVersion();  // game version when the page was loaded
this.bga.tableOptions.getGameLanguage(); // index from gameinfos language_dependency, or null
```

## 3D Display

`this.bga.display3D` wraps BGA's legacy 3D table view (rotate / slide / zoom of a set of elements):

```javascript
this.bga.display3D.init3d({
    elements: [document.getElementById('board')],
    showControls: true,                      // legacy 3D control buttons
    view: { zoom: 1, minZoom: 0.5, maxZoom: 2, xAxis: 0, zAxis: 0 },
    draggable: true,                         // right-button rotate, middle-button slide (or { rotationMultiplier, slideMultiplier })
    zoomByWheel: true,                       // or { multiplier }
});
this.bga.display3D.setElements([...]); this.bga.display3D.addElements([...]);
this.bga.display3D.rotate(dx, dz); this.bga.display3D.slide(dx, dy); this.bga.display3D.zoom(inc);
this.bga.display3D.setZoom(1.5); this.bga.display3D.resetView();
// bounds: setMinXAxis/setMaxXAxis/setMinZAxis/setMaxZAxis/setMin|MaxXPos/setMin|MaxYPos/setMin|MaxZoom(value|null)
```

## Loading Libraries

```javascript
// ESM libs (typed names: 'bga-animations' | 'bga-autofit' | 'bga-cards' | 'bga-dice' |
//           'bga-jump-to' | 'bga-score-sheet' | 'bga-zoom'); version '1.x' = latest 1.*, or pinned '1.0.0'
const BgaCards = await importEsmLib('bga-cards', '1.x');
getLibUrl('bga-cards', '1.x');            // URL only
// Dojo/UMD legacy components (ebg.stock, ebg.zone, …): importDojoLibs — see their reference files
```

## Event Handling

```javascript
// Plain DOM — what the scaffold and real projects use
document.getElementById('board').addEventListener('click', (e) => this.onBoardClick(e));
document.querySelectorAll('.mygame_piece').forEach(el => el.addEventListener('click', (e) => this.onPieceClick(e)));
// Delegation keeps one listener alive across re-renders:
board.addEventListener('click', (e) => { const cell = e.target.closest('[data-cell]'); if (cell) this.onCellClick(cell); });

// Typed dojo form (3 args) — for legacy components that expose dojo events
const handle = dojo.connect($('element'), 'onclick', (e) => this.onMethodName(e));
dojo.disconnect(handle);
// The old gamegui helpers (this.connect / connectClass / disconnect) are NOT part of the typed
// this.bga.gameui surface — don't introduce them in new code.
```

## Pitfalls

- **`performAction` only from user events** — NEVER from loops, callbacks, notifications, or state methods
- **`attachToNewParent` returns a new node** — the old reference and its listeners are invalid afterwards
- **No `.promise` on dojo animations** — `await this.bga.gameui.bgaPlayDojoAnimation(anim)`
- **Slide methods incompatible with CSS transforms** (scale, zoom, rotate)
- **Notification args are strings** — always `parseInt(value, 10)` before arithmetic
- **Hotseat shares `gamedatas`** — no per-player client data
- **Prefix CSS classes** with game name to avoid conflicts (e.g., `mygame_selected`)
- **`#overall-content` gets state CSS class** — `.gamestate_playerTurn` for conditional CSS
- **Tooltips need unique IDs** on all target elements
- **Avoid custom zoom** — provide +/- if essential, but default view must be fully playable
- **Prefer vanilla DOM over `dojo.query` / `dojo.style`** (`querySelector`, `classList`, `style`) — the typed `dojo` surface is small and legacy
- **Trust `bga-framework.d.ts` over the wiki** — the scaffold's declaration file is generated from the framework; when the wiki omits an option the d.ts lists (e.g. `addActionButton` `tooltip`/`confirm`/`autoclick`), the d.ts is right
- **Tutorial mode detection** — `typeof g_tutorialwritten !== 'undefined'` (object when a tutorial is being played)
