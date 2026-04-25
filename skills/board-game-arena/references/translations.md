# Translations — BGA Reference

**Type:** PHP + JS
**Doc:** https://en.doc.boardgamearena.com/Translations
**When to use:** Making game text translatable for BGA's community translation system.

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
