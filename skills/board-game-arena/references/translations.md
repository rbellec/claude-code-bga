# Translations — BGA Reference

**Type:** PHP + JS
**Doc:** https://en.doc.boardgamearena.com/Translations
**When to use:** Making game text translatable for BGA's community translation system.

## Workflow — where and when translations actually happen

**TL;DR:** During alpha, your Studio Control Panel does **not** have a
per-game "Translation" page. The actual translations live elsewhere and only
become available once your game enters beta.

### Two different places

| Place | URL | Purpose |
|---|---|---|
| **Studio Control Panel** (per game) | `studio.boardgamearena.com` → your game | Code, deploy, debug, options, **"Display dummy translation" button** (see below) |
| **Translation Center** (global) | https://en.boardgamearena.com/translationhq | The actual place where translators (and you) type translations per language |

### Lifecycle

1. **Alpha** — wrap your strings with `_()` / `clienttranslate()` and deploy.
   - Strings are extracted by BGA but are **not** exposed to the public Translation Center.
   - You cannot, by default, translate them yourself via the UI.
   - **Workaround:** ask BGA admins on the BGA Developers Discord to enable
     translations on your alpha. They generally accept. With a pre-filled
     EN→target-language document on hand, filling things in takes ~20 min.

2. **Beta** — once your game enters beta, strings automatically appear in the
   Translation Center and the community begins translating. From the official
   doc: *"Once your game enters Beta, the BGA player community will descend
   upon your game like a plague of locusts and translate it into dozens of
   languages."*

3. **Production** — validated translations are served at runtime based on the
   player's language.

### What "Display dummy translation" actually does

It is a **debug tool**, not an editor. It scans your code and replaces every
`_()` / `clienttranslate()` site with a dummy marker at runtime. Use it to:

- Verify visually that **all** user-facing strings are wrapped
- Detect strings that escape and stay in raw English (= i18n bug)
- It is the runtime equivalent of the static `Check project` analysis

Run it systematically before pushing to beta.

## Core Functions

### PHP — Marking for Translation

```php
// clienttranslate() — marks string for client-side translation
// MUST contain literal text only — no variables, no sprintf()
clienttranslate('${player_name} plays a card')    // OK
clienttranslate("${player_name} plays $card")     // WRONG — variable inside

// Use in:
// - Notification messages
// - Exception messages
// - State descriptions (in states.inc.php)
// - Material definitions (card names, etc.)
```

### JS — Displaying Translations

```javascript
_('Play a card')                    // translate and display
_('${player_name} plays ${card}')   // with substitution
```

### Config Files

In `gameoptions.json`, `gamepreferences.json`, `stats.json` — names are **auto-translated**, no wrapper needed.

## Notification i18n

Mark args that need translation with the `i18n` array:

```php
$this->bga->notify->all('cardPlayed', clienttranslate('${player_name} plays ${card_name}'), [
    'player_name' => $this->getPlayerNameById($pid),
    'card_name' => $this->CARD_NAMES[$type],  // defined with clienttranslate() in material
    'i18n' => ['card_name'],                   // this arg will be translated client-side
]);
```

## Pattern: raw value (CSS) + translated label (display)

Common case: one notification field is needed both as a CSS class on the JS
side (must stay English: `mygame-token-red`) **and** as a label shown in the
game log (must be translated).

**Don't** — pass the raw value in the message and rely on `i18n` to translate it:

```php
// ❌ WRONG — 'red' shows up in the log in English regardless of locale
clienttranslate('${player_name} places a ${color} token');
// args: ['color' => 'red', 'i18n' => ['color']]
// → 'red' was never wrapped in clienttranslate(), so it isn't in any dictionary
```

**Do** — two distinct fields, one raw and one translated:

```php
// ✅ GOOD
clienttranslate('${player_name} places a ${color_label} token');
// args: [
//   'color'       => $tokenColor,                                          // raw — for CSS
//   'color_label' => $tokenColor === 'red'
//                      ? clienttranslate('red')
//                      : clienttranslate('blue'),                          // translated
//   'i18n'        => ['color_label'],
// ]
```

On the JS side, the notif handler reads `args.color` (raw) for the CSS class,
and the BGA engine has already substituted the translated `args.color_label`
into the visible message.

## Material Definitions

```php
// In material.inc.php or as class constants:
$this->card_types = [
    1 => ['name' => clienttranslate('Knight'), 'tooltip' => clienttranslate('Moves in L-shape')],
    2 => ['name' => clienttranslate('Bishop'), 'tooltip' => clienttranslate('Moves diagonally')],
];
```

## Parameterized Strings

```php
// PHP — parameters in notification args, not in the string itself
clienttranslate('${player_name} scores ${points} point(s)')
// Args: ['player_name' => ..., 'points' => $n]

// JS — substitute after translation
dojo.string.substitute(_('Pick ${n} cards'), { n: 2 })
```

## Formatting with bga_format

```javascript
// Markdown-style formatting
bga_format(_('This is *bold* and _highlighted_'), {
    '*': text => `<b>${text}</b>`,
    '_': 'highlight-class',  // wraps in <span class="highlight-class">
});
```

## Best Practices

- **Reuse identical strings** — reduces translator workload
- **Use placeholders** not concatenation — `'${n} cards'` not `n + ' cards'`
- **English as development language** — all strings in English first
- **Present tense** — `'plays'` not `'played'`
- **Avoid gender-specific pronouns** — use `'${player_name}'` not `'he/she'`
- **Use `(s)` for plurals** — `'${n} card(s)'` rather than separate plural forms
- **No period** for buttons, titles, menu items
- **Period** for complete sentences in logs

## Pitfalls

- **`clienttranslate()` is statically scanned** — variables inside it break detection
- **Concatenation defeats detection** — `clienttranslate('Part 1') . clienttranslate('Part 2')` loses context
- **Can't call `_()` in JS object constructors** — translation not available yet
- **`totranslate()`** is only for legacy `gameoptions.inc.php` — not needed in JSON config files
- **Avoid escape sequences** (`\n`, `\t`) in strings — use HTML instead
- **Locale detection:** `_('$locale')` returns the current language code
- **JS `${var}` interpolation defeats translation** —
  `_('place a ${color} token').replace('${color}', cardColor)` is **broken**:
  `cardColor` is the raw English value (e.g. `'red'`/`'blue'`) and ends up
  untranslated inside the localized string. Split per value instead:
  `cardColor === 'red' ? _('place a red token') : _('place a blue token')`.
  In some target languages adjectives may agree the same way and you might
  get away with it; for strongly gendered languages (German, Russian, …),
  only per-string splits are correct.
- **`UserException` must be wrapped** — its messages are shown to the player
  (red popup). The framework doc explicitly says *"Should be translated"*.
  ```php
  // ❌
  throw new UserException('Invalid position');
  // ✅
  throw new UserException(clienttranslate('Invalid position'));
  ```
  `SystemException` and `VisibleSystemException` are the opposite — they go
  to server logs, devs only, and **must not** be translated.

## Useful URLs

- [Translations doc](https://en.doc.boardgamearena.com/Translations) — official API reference
- [Translation Center](https://en.boardgamearena.com/translationhq) — where translations are entered (beta+)
- [Translation guidelines](https://en.doc.boardgamearena.com/Translation_guidelines) — style conventions
- [Translations recap (bga-devs blog)](https://bga-devs.github.io/blog/posts/translations-summary/) — dev-side technical recap
