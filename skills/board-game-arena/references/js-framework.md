# JS Framework — BGA Reference

**Type:** JS Client (ES6 Module)
**Doc:** https://en.doc.boardgamearena.com/Game_interface_logic:_yourgamename.js
**When to use:** Full API reference for building the game client beyond the basics in SKILL.md.

## Lifecycle

```javascript
constructor()                          // global vars, register state classes
setup(gamedatas)                       // build UI, connect events, setup notifications
onEnteringState(stateName, args)       // state-specific UI (NOT called after setup)
onLeavingState(stateName)              // cleanup
onUpdateActionButtons(stateName, args) // action buttons (REQUIRED for multiactive)
```

## State Management

```javascript
// JS State Classes (optional — alternative to switch in onEnteringState)
class PlayerTurn {
    constructor(game, bga) { this.game = game; this.bga = bga; }
    onEnteringState(args, isCurrentPlayerActive) { }
    onLeavingState(args, isCurrentPlayerActive) { }
    onPlayerActivationChange(args, isCurrentPlayerActive) { }
}
this.bga.states.register('PlayerTurn', new PlayerTurn(this, this.bga));
this.bga.states.logger = console.log; // debug

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
this.bga.players.getPlayer(pid)               // player data object
this.bga.players.getActivePlayerId()          // single active player
this.bga.players.getActivePlayerIds()         // array (multiactive)
this.bga.gameui.getPlayerAvatarUrl(pid, 50)   // sizes: 32, 50, 92, 184

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
        disabled: false,     // grayed out
        confirm: _('Are you sure?'), // confirmation dialog
        tooltip: _('Play your selected card'),
    }
);

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
});

// Forbidden parameter names: $args, $activePlayerId, $currentPlayerId
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

// Multiple choice
this.bga.dialogs.multipleChoice(_('How many?'), ['0', '1', '5']).then(choice => {
    if (choice !== null) { /* handle */ }
});

// Message
this.bga.dialogs.showMessage(_('Important announcement'), 'info'); // 'info' or 'error'
this.bga.gameui.showMoveUnauthorized(); // standard error

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
// Also: g_gamethemeurl + 'img/file.svg'

// Sounds (no file extension — BGA serves .ogg or .mp3)
this.bga.sounds.play('click');
this.bga.sounds.dontPreloadSounds(['rare_sound']);
this.bga.gameui.disableNextMoveSound(); // suppress default move sound
```

## User Preferences

```javascript
const pref = this.bga.userPreferences.get(100); // preference ID from gamepreferences.json
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
