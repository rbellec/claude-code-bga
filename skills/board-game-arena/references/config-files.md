# Config Files — BGA Reference

**Doc:** https://en.doc.boardgamearena.com/Game_options_and_preferences:_gameoptions.inc.php + https://en.doc.boardgamearena.com/Game_statistics:_stats.inc.php + https://en.doc.boardgamearena.com/Gameinfos.inc.php
**When to use:** Setting up or modifying game options, preferences, statistics, or metadata.

## gameoptions.json

Game-affecting options shown at table creation.

```json
{
    "100": {
        "name": "Board size",
        "values": {
            "1": { "name": "Small (3x3)" },
            "2": { "name": "Medium (5x5)", "tmdisplay": "Medium board" },
            "3": { "name": "Large (7x7)" }
        },
        "default": 2
    },
    "101": {
        "name": "Advanced rules",
        "values": {
            "0": { "name": "No" },
            "1": { "name": "Yes" }
        },
        "default": 0,
        "displaycondition": [
            { "type": "minplayers", "value": 2 }
        ]
    }
}
```

### Key Fields

| Field | Level | Description |
|-------|-------|-------------|
| `name` | option | Display name (auto-translated) |
| `values` | option | Map of numeric value → {name, description?, tmdisplay?} |
| `default` | option | Default value |
| `level` | option | `"base"` (default), `"major"` (prominent), `"additional"` (hidden) |
| `tmdisplay` | value | Text shown in lobby table listing |
| `displaycondition` | option | Hide option based on player count or other options |
| `startcondition` | option | Validate at game start |
| `nobeginner` | value | Advanced players only |
| `beta` / `alpha` | value | Development stage flag |

### Checkbox Display

Auto-renders as checkbox if exactly 2 values with names: yes/no, on/off, or enabled/disabled (case-insensitive).

### Access in PHP

```php
$value = (int)$this->bga->tableOptions->get(100);
// Or in setupNewGame: $options parameter
```

### Reserved IDs

- **200**: Clock mode (real-time vs turn-based)
- **201**: Rating mode (normal, training, arena)

## gamepreferences.json

Cosmetic preferences (per player, not game-affecting).

```json
{
    "100": {
        "name": "Card size",
        "values": {
            "1": { "name": "Small" },
            "2": { "name": "Normal" },
            "3": { "name": "Large" }
        },
        "default": 2
    },
    "101": {
        "name": "Colorblind mode",
        "values": {
            "0": { "name": "Disabled" },
            "1": { "name": "Enabled" }
        },
        "default": 0,
        "needReload": true,
        "cssPref": true
    }
}
```

### Key Fields

| Field | Description |
|-------|-------------|
| `needReload` | Auto-reload UI when changed |
| `cssPref` | Apply CSS class to `<html>` tag: `prefs_100_2` |

### Access

```javascript
// JS
const pref = this.bga.userPreferences.get(100);

// PHP
$pref = $this->userPreferences->get($playerId, 100);
```

## stats.json

Statistics displayed at game end.

```json
{
    "table": {
        "total_rounds": {
            "id": 10,
            "name": "Number of rounds",
            "type": "int"
        }
    },
    "player": {
        "cards_played": {
            "id": 10,
            "name": "Cards played",
            "type": "int"
        },
        "avg_score_per_turn": {
            "id": 11,
            "name": "Average score per turn",
            "type": "float"
        },
        "won_by_elimination": {
            "id": 12,
            "name": "Won by elimination",
            "type": "bool"
        }
    }
}
```

### Key Rules

- **IDs must be >= 10** and unique within their section (table/player)
- **Types:** `int`, `float`, `bool`
- **Never reuse or change IDs** — loses historical data
- **Total limit:** table stats + (player stats x max players) <= ~930 — exceeding crashes the game
- **Uninitialized stats display as "-"** (useful for variant-specific stats)
- **`"display": "limited"`** — admin-only visibility
- **`"value_labels"`** — human-readable labels for categorical data

### PHP Usage

```php
// In setupNewGame:
$this->bga->tableStats->init('total_rounds', 0);
$this->bga->playerStats->init('cards_played', 0, $pid);

// During game:
$this->bga->tableStats->inc('total_rounds', 1);
$this->bga->playerStats->inc('cards_played', 1, $pid);
$this->bga->playerStats->set('avg_score_per_turn', $avg, $pid);
```

## gameinfos.inc.php

Game metadata. Key fields:

```php
$gameinfos = [
    'game_name' => 'My Game',
    'publisher' => 'Publisher Name',    // empty for public domain
    'publisher_website' => '',
    'publisher_bgg_id' => 0,
    'bgg_id' => 0,                      // 0 if no BGG page

    'players' => [2, 3, 4],            // supported player counts
    'suggest_player_number' => 3,       // optimal (affects ELO K-factor)
    'not_recommend_player_number' => null,

    'player_colors' => ['ff0000', '008000', '0000ff', 'ffa500'],

    'favorite_colors_support' => true,
    'disable_player_order_swap_table' => false,

    'is_beta' => 1,                     // cannot be 0 until release
    'is_coop' => 0,                     // 1 for cooperative games

    'language_dependency' => false,

    'complexity' => 2,                  // 1-5
    'luck' => 2,                        // 1-5
    'strategy' => 3,                    // 1-5
    'diplomacy' => 1,                   // 1-5

    'losers_not_ranked' => false,       // true = "Winner" or "Loser" only
    'tie_breaker_description' => '',    // NO newlines (breaks JS)

    'db_undo_support' => false,         // true to enable undo system
];
```

### Reload After Changes

Deploy then reload via BGA Studio manage page (or use the automated reload from SKILL.md section "Reload BGA Studio configs").

## JSON Constraints

All JSON config files:
- **No comments** (use `"$comment": "..."` hack if needed)
- **No trailing commas**
- **All keys must be strings**
- **No PHP constants** — use raw numeric values
- **Auto-translated** — no `totranslate()` wrappers needed

## Pitfalls

- **Order in JSON determines display order**, not ID sorting
- **gameoptions affect gameplay**, gamepreferences are cosmetic only
- **Stats limit ~930 total** — exceeding crashes the game with no clear error
- **Stats IDs are permanent** — changing them loses historical data
- **`tie_breaker_description` must have no newlines** — causes JS errors
- **`is_beta` cannot be set to 0** before release stabilization
- **After changing any config file**: must deploy AND reload via BGA Studio manage page
