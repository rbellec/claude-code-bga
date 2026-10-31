# Notifications — BGA Reference

**Type:** PHP + JS
**Doc:** https://en.doc.boardgamearena.com/Main_game_logic:_yourgamename.game.php (PHP) + https://en.doc.boardgamearena.com/Game_interface_logic:_yourgamename.js (JS)
**When to use:** Sending game events from server to client. Every user-visible change must go through a notification.

## PHP — Sending Notifications

```php
// Public (all players + spectators)
$this->bga->notify->all('cardPlayed', clienttranslate('${player_name} plays ${card_name}'), [
    'player_id' => $pid,
    'player_name' => $this->getPlayerNameById($pid),
    'card_name' => $cardName,
    'card' => $cardData,           // custom data for JS
    'i18n' => ['card_name'],       // mark args for translation
]);

// Private (one player only)
$this->bga->notify->player($pid, 'cardsDrawn', clienttranslate('You draw ${count} card(s)'), [
    'count' => $count,
    'cards' => $cards,
]);
```

### Decorator — fill recurring args once

Registered once (constructor), applied to the args of every `notify->all/player` call. The scaffold ships this example commented out:

```php
$this->bga->notify->addDecorator(function (string $message, array $args): array {
    if (isset($args['player_id']) && !isset($args['player_name']) && str_contains($message, '${player_name}')) {
        $args['player_name'] = $this->getPlayerNameById($args['player_id']);
    }
    return $args;   // must return the (possibly extended) args
});
```

### Addressing players by "no"

`$this->bga->notify->setUseNo(true)` makes `notify->player($no, …)` take the seat number instead of the player id (same switch exists on `playerStats` and `PlayerCounter`). Keep the whole game on one convention.

### Player Name Highlighting

Use `${player_name}` in the message with a matching `player_id` in args — BGA auto-colors it.

```php
// Multiple players:
'${player_name} attacks ${player_name2}'
// Args: player_id, player_name, player_id2, player_name2
```

### Private Data in Public Notifications

Each `_private` entry is a `NotificationMessage` (replacement log + private args) — not a plain array:

```php
use Bga\GameFramework\NotificationMessage;

$this->bga->notify->all('cardDealt', clienttranslate('A card is dealt to ${player_name}'), [
    'player_id' => $pid,
    'player_name' => $this->getPlayerNameById($pid),
    '_private' => [
        // this player sees the private log AND receives the private args
        $pid => new NotificationMessage(
            clienttranslate('You receive ${card_name}'),
            ['card' => $cardData, 'card_name' => $cardName, 'i18n' => ['card_name']]
        ),
    ],
    '_merge_private' => true,  // merge private args into args for that player
    // (or call $this->bga->notify->alwaysMergePrivate() once in setup)
]);
```

JS side: without `_merge_private`, the private args arrive under `args._private`; with it, they are merged into `args` for the targeted player.

### Preserve Args for Replay

```php
$this->bga->notify->all('moveToken', clienttranslate('...'), [
    'token_id' => $id,
    'position' => $pos,
    'preserve' => ['token_id', 'position'], // kept in game logs for replay
]);
```

### Built-in Notification Types

```php
// Scoring dialog (table window)
$this->bga->notify->all('tableWindow', '', [
    'id' => 'finalScoring',
    'title' => clienttranslate('Final Scoring'),
    'table' => $scoreTable,
    'closing' => clienttranslate('Close'),
]);

// Simple log message (no handler needed)
$this->bga->notify->all('message', clienttranslate('Round ${round} begins'), [
    'round' => $round,
]);

// Pause
$this->bga->notify->all('simplePause', '', ['time' => 1000]);
```

## JS — Receiving Notifications

### Modern Setup (Promise-based — recommended)

```javascript
// In setup():
this.bga.notifications.setupPromiseNotifications({
    // prefix: 'notif_',         // default
    // minDuration: 500,          // ms, notifications with a log text (only when animations are active)
    // minDurationNoText: 1,      // ms, notifications without text
    // handlers: [],              // extra objects whose notif_* methods are registered too
    // logger: console.log,       // debug
    // ignoreNotifications: [],   // skip these types
    // onStart: (notifName, msg, args) => {}, onEnd: (notifName, msg, args) => {},
});

// Handler methods — auto-detected by prefix
async notif_cardPlayed(args) {
    // args = the PHP notification args
    await this.animateCard(args.card);   // must resolve instantly when !bgaAnimationsActive() (replay, archive)
    // minDuration is enforced automatically after completion
}

notif_scoreUpdate(args) {
    // Sync handlers work too (no await needed)
    this.scoreCounter[args.player_id].toValue(parseInt(args.score, 10));
}
```

### Legacy Setup (manual)

```javascript
// In setupNotifications():
dojo.subscribe('cardPlayed', this, 'notif_cardPlayed');
this.bga.gameui.notifqueue.setSynchronous('cardPlayed', 500); // wait 500ms between notifs
// (`notifqueue` is the legacy runtime property — not part of the typed this.bga.gameui surface)
```

### Ignore Notifications

```javascript
// Skip notifications client-side (e.g., dealt cards you already animated)
this.bga.gameui.notifqueue.setIgnoreNotificationCheck('dealCard',
    (notif) => notif.args.player_id === this.bga.players.getCurrentPlayerId()
);
// NOTE: client-side only — not for hiding private data (use _private in PHP);
// cannot be used on synchronous notifications without a duration
```

### Dynamic Synchronous Duration

```javascript
// In legacy setSynchronous handler:
notif_complexAnimation(notif) {
    const duration = notif.args.count * 200;
    this.bga.gameui.notifqueue.setSynchronousDuration(duration);
    // CRITICAL: must call this or UI locks forever
}
```

### Notification Object

```javascript
{
    type: 'cardPlayed',        // notification name
    log: '${player_name} plays ${card_name}', // log message
    args: { ... },             // PHP args
    bIsTableMsg: true,         // from notify->all()
    move_id: 42,               // associated move
    time: 1700000000,          // UNIX timestamp
    uid: 123456,               // number
}
```

## Pitfalls

- **Notifications queue until action completes** — exceptions prevent ALL notifications from sending
- **Notification payload limit: 128KB** total across bundled requests
- **Never call setState in notifications** — causes race conditions, breaks replays
- **`setupPromiseNotifications` auto-enforces minDuration** — even if handler completes instantly
- **`setSynchronousDuration` must be called** in legacy sync handlers — or UI locks forever
- **`setIgnoreNotificationCheck` is client-side only** — not a security measure for private data
- **Notification args are strings** — always `parseInt()` before arithmetic in JS
- **At least one notification per action** is required (state transitions count as notifications)
- **`i18n` array** marks args for translation — e.g., `'i18n' => ['card_name']`
- **Use `clienttranslate()`** for the message — literal text only, no variables inside it
