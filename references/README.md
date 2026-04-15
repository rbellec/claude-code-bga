# BGA References

Detailed reference guides for BGA framework APIs and libraries. Each file is self-contained: setup, API, example, and pitfalls.

These files are designed to be loaded on demand by Claude Code when implementing a game. The main skill file (`SKILL.md`) contains decision tables pointing here.

## Framework Guides

| Guide | File | Purpose |
|-------|------|---------|
| JS Framework | [js-framework.md](js-framework.md) | Full JS client API (DOM, animations, tooltips, dialogs, panels, sounds) |
| PHP Framework | [php-framework.md](php-framework.md) | Full PHP server API (DB, players, states, scoring, undo, random) |
| Notifications | [notifications.md](notifications.md) | Complete notification system (PHP send + JS receive + sync) |
| Config Files | [config-files.md](config-files.md) | gameoptions.json, gamepreferences.json, stats.json, gameinfos.inc.php |
| Translations | [translations.md](translations.md) | i18n patterns: clienttranslate, _(), i18n array, bga_format |
| Guidelines | [guidelines.md](guidelines.md) | BGA Studio review standards: layout, accessibility, UX, design |

## PHP (Server-side)

| Library | File | Purpose |
|---------|------|---------|
| Deck | [deck.md](deck.md) | Card/tile management (shuffle, draw, move) |
| PlayerCounter / TableCounter | [player-table-counter.md](player-table-counter.md) | Tracked numeric counters with auto-notification |

## JavaScript (Client-side) — Modern ESM

| Library | File | Purpose |
|---------|------|---------|
| BgaCards | [bga-cards.md](bga-cards.md) | Card display, animation, selection |
| BgaDice | [bga-dice.md](bga-dice.md) | Dice rendering and roll animation |
| BgaAnimations | [bga-animations.md](bga-animations.md) | Slide, scale, rotate, scoring animations |
| BgaScoreSheet | [bga-score-sheet.md](bga-score-sheet.md) | Animated end-game score summary |
| BgaAutofit | [bga-autofit.md](bga-autofit.md) | Auto-size text in fixed containers |

## JavaScript (Client-side) — Classic Dojo

| Library | File | Purpose |
|---------|------|---------|
| Stock | [stock.md](stock.md) | Display sets of items (hand, market) |
| Zone | [zone.md](zone.md) | Spatial arrangement of tokens |
| Counter | [counter.md](counter.md) | Animated numeric display |
| Scrollmap | [scrollmap.md](scrollmap.md) | Scrollable/pannable game board |
| Draggable | [draggable.md](draggable.md) | Drag-and-drop (legacy) |
| ExpandableSection | [expandable-section.md](expandable-section.md) | Collapsible UI sections |
