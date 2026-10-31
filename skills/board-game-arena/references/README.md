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
| Gameinfos JSONC | [gameinfos-jsonc.md](gameinfos-jsonc.md) | `gameinfos.jsonc` format (2026+ scaffolds), correspondence with legacy `.inc.php` |
| Legacy Migration | [legacy-migration.md](legacy-migration.md) | Converting a pre-2025 project to the new framework (files, PHP, JS, order) |
| Translations | [translations.md](translations.md) | i18n patterns: clienttranslate, _(), i18n array, bga_format |
| Guidelines | [guidelines.md](guidelines.md) | BGA Studio review standards: layout, accessibility, UX, design |
| Debug Helpers | [debug-helpers.md](debug-helpers.md) | `debug_<name>` auto-buttons + toolbar input, premium-gate workaround for end-game stats |
| Makefile Template | [makefile-template.md](makefile-template.md) | Project setup: check + deploy targets (used once at init) |
| Rules Clarification | [rules-clarification.md](rules-clarification.md) | Rules analysis process, author Q&A, assumptions tracking (loaded at project start) |
| PHP Code Quality | [php-code-quality.md](php-code-quality.md) | Optional `make audit` loop: PHP-CS-Fixer, Rector, PHPStan, PHPMD (opt-in) |
| Alpha Release | [alpha-release.md](alpha-release.md) | Requesting ALPHA status: checklist, images, licensing, zombie level, tags |
| Tutorial-Ready Audit | [tutorial-ready.md](tutorial-ready.md) | Making the game tutorial-able: valid replay, manually re-triggerable actions, anchorable DOM (audit to run before alpha) |
| BGG Submission | [bgg-submission.md](bgg-submission.md) | Producing the BGG entry that unlocks `bgg_id` (used once at release, optional) |

## PHP (Server-side)

| Library | File | Purpose |
|---------|------|---------|
| ItemManager + Collection | [item-manager.md](item-manager.md) | Typed game items in DB (cards, tiles, tokens): locations, pick, shuffle, query — successor of Deck |
| Deck (deprecated) | [deck.md](deck.md) | Legacy card/tile management — pre-2026 projects only |
| PlayerCounter / TableCounter | [player-table-counter.md](player-table-counter.md) | Tracked numeric counters with auto-notification |

## JavaScript (Client-side) — Modern ESM

| Library | File | Purpose |
|---------|------|---------|
| BgaCards | [bga-cards.md](bga-cards.md) | Card display, animation, selection |
| BgaDice | [bga-dice.md](bga-dice.md) | Dice rendering and roll animation |
| BgaAnimations | [bga-animations.md](bga-animations.md) | Slide, scale, rotate, scoring animations |
| BgaScoreSheet | [bga-score-sheet.md](bga-score-sheet.md) | Animated end-game score summary |
| BgaAutofit | [bga-autofit.md](bga-autofit.md) | Auto-size text in fixed containers |
| BgaZoom | [bga-zoom.md](bga-zoom.md) | Zoom controls + auto-zoom of a wide play area |
| BgaJumpTo | [bga-jump-to.md](bga-jump-to.md) | Shortcut panel to many boards |

## JavaScript (Client-side) — Classic Dojo

| Library | File | Purpose |
|---------|------|---------|
| Stock (deprecated) | [stock.md](stock.md) | Legacy item display (hand, market) — use BgaCards on new projects |
| Zone | [zone.md](zone.md) | Spatial arrangement of tokens |
| Counter | [counter.md](counter.md) | Animated numeric display |
| Scrollmap | [scrollmap.md](scrollmap.md) | Scrollable/pannable game board |
| Draggable | [draggable.md](draggable.md) | Drag-and-drop (legacy) |
| ExpandableSection | [expandable-section.md](expandable-section.md) | Collapsible UI sections |
