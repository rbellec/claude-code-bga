# JS Framework — BGA Reference

**Type:** JS Client (ES6 Module)
**Doc:** https://en.doc.boardgamearena.com/Game_interface_logic:_yourgamename.js
**When to use:** Full API reference for building the game client beyond the basics in SKILL.md.

## Lifecycle

```javascript
constructor(bga)                       // receives the `bga` facade: this.bga = bga; globals, register state classes
setup(gamedatas)                       // build UI, connect events, setup notifications
onEnteringState(stateName, args)       // state-specific UI (NOT called after setup)
onLeavingState(stateName)              // cleanup
onUpdateActionButtons(stateName, args) // action buttons (REQUIRED for multiactive)
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
this.bga.gameui.attachToNewParent('mobile', 'newParent');    // reparent (clones!)

// Templates
this.bga.gameui.format_string(_('${player_name} plays ${card}'), args);
```

## Animations

```javascript
// Slide (returns animation — must call .play())
this.bga.gameui.slideToObject('mobile', 'target', duration, delay).play();
this.bga.gameui.slideToObjectPos('mobile', 'target', x, y, duration, delay).play();

// Auto-playing (return .promise for await)
this.bga.gameui.slideTemporaryObject(html, parent, from, to, duration).promise;
this.bga.gameui.slideToObjectAndDestroy('mobile', 'target', duration).promise;
this.bga.gameui.fadeOutAndDestroy('element', duration).promise;

// Rotation
this.bga.gameui.rotateTo('element', degrees).promise;    // animated
this.bga.gameui.rotateInstantTo('element', degrees);     // instant

// Wait (respects animation settings)
await this.bga.gameui.wait(500);

// Check before animating
if (this.bga.gameui.bgaAnimationsActive()) { /* animate */ }
```

## Action Buttons

```javascript
// In onUpdateActionButtons:
this.bga.statusBar.addActionButton(
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

// Status bar title
this.bga.statusBar.setTitle(_('${you} must choose a card'), {});
```

## Server Actions

```javascript
// ONLY call from user events (onclick, onchange) — NEVER from notifications/loops/callbacks
this.bga.actions.performAction('actPlayCard', { id: cardId })
    .then(() => { /* success */ })
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
this.bga.gameui.addTooltip('element_id', _('Help text'), _('Action text'));
this.bga.gameui.addTooltipHtml('element_id', '<div>Rich HTML</div>');
this.bga.gameui.addTooltipToClass('card', _('A card'), '');
this.bga.gameui.addTooltipHtmlToClass('card', htmlContent);
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

// Sounds (no file extension — BGA serves .ogg or .mp3)
this.bga.sounds.play('click');
this.bga.sounds.dontPreloadSounds(['rare_sound']);
this.bga.gameui.disableNextMoveSound(); // suppress default move sound
```

## User Preferences

```javascript
const pref = this.bga.userPreferences.get(100); // preference ID from gamepreferences.json
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
// Connect events
dojo.connect($('element'), 'onclick', this, 'onMethodName');
this.bga.gameui.connect($('element'), 'onclick', 'onMethodName');
this.bga.gameui.connect($('element'), 'onclick', (e) => { /* inline */ });
this.bga.gameui.connectClass('css-class', 'onclick', 'onMethodName');

// Disconnect
this.bga.gameui.disconnect($('element'), 'onclick');
this.bga.gameui.disconnectAll();
```

## Pitfalls

- **`performAction` only from user events** — NEVER from loops, callbacks, notifications, or state methods
- **`attachToNewParent` clones** — destroys original element and dojo.connect handlers
- **`slideToObject` needs `.play()`** — `slideTemporaryObject` auto-plays
- **Slide methods incompatible with CSS transforms** (scale, zoom, rotate)
- **Notification args are strings** — always `parseInt(value, 10)` before arithmetic
- **Hotseat shares `gamedatas`** — no per-player client data
- **Prefix CSS classes** with game name to avoid conflicts (e.g., `mygame_selected`)
- **`#overall-content` gets state CSS class** — `.gamestate_playerTurn` for conditional CSS
- **Tooltips need unique IDs** on all target elements
- **Avoid custom zoom** — provide +/- if essential, but default view must be fully playable
- **Deprecated: dojo.query, dojo.style** — use vanilla JS (document.querySelector, classList, etc.)
- **Trust `bga-framework.d.ts` over the wiki** — the scaffold's declaration file is generated from the framework; when the wiki omits an option the d.ts lists (e.g. `addActionButton` `tooltip`/`confirm`/`autoclick`), the d.ts is right
- **Tutorial mode detection** — `typeof g_tutorialwritten !== 'undefined'` (object when a tutorial is being played)
