# gameinfos.jsonc — BGA Reference

**Type:** Config (game metadata)
**Doc:** https://en.doc.boardgamearena.com/Game_meta-information:_gameinfos.inc.php (redirects to the current "gameinfos.jsonc" page)
**When to use:** Editing game metadata on a recent project, or converting an older project's `gameinfos.inc.php` (see `references/legacy-migration.md`).

## Which format does your project use?

- **Recent scaffolds (2026+) generate `gameinfos.jsonc`** — JSON with comments.
- **Older projects use `gameinfos.inc.php`** (PHP array) — still works, no forced migration. Check which file exists at the project root before editing.

Unlike the strict-JSON config files (`stats.json`, `gameoptions.json`, `gamepreferences.json`), **JSONC allows `//` comments** — the scaffold itself ships heavily commented.

## Template (from a real 2026 scaffold)

```jsonc
{
    // Name of the game in English (basis for translation)
    "game_name": "My Great Game",

    "publisher": "My Publishing Company",   // empty string if none
    "publisher_website": "http://www.mypublishingcompany.com/",
    "publisher_bgg_id": 1234,
    "bgg_id": 0,                            // 0 until the BGG entry exists

    "players": [2, 3, 4],
    "suggest_player_number": null,          // null if no advice; REQUIRED if the lowest
                                            // player count doesn't fit default options
    "not_recommend_player_number": null,    // e.g. [2, 3]

    "estimated_duration": 30,               // minutes, used for launch only

    // Seconds added by giveExtraTime() per speed profile
    "fast_additional_time": 30,
    "medium_additional_time": 40,
    "slow_additional_time": 50,

    "tie_breaker_description": "",          // tooltip explaining player_score_aux; NO newlines
    "losers_not_ranked": false,             // true = "Winner"/"Loser" only (see config-files.md warning)
    "solo_mode_ranked": false,
    "is_coop": 0,

    "language_dependency": false,           // or { "1": "en", "2": "fr" } by popularity

    "player_colors": ["ff0000", "008000", "0000ff", "ffa500"],
    "favorite_colors_support": true,        // flag only — needs reattributeColorsBasedOnPreferences() in PHP

    "disable_player_order_swap_on_rematch": false,

    // Game interface width range (left area, without the right column)
    "game_interface_width": {
        "min": 740                          // 320–740; lower = better mobile display
    },

    // Turn PHP warnings into exceptions — keep true: warnings otherwise corrupt
    // the JSON responses (see TECHNICAL_NOTES.md)
    "exception_on_warning": true
}
```

## Correspondence with the legacy `gameinfos.inc.php`

| Legacy (`.inc.php`) | jsonc | Note |
|---|---|---|
| `'disable_player_order_swap_table'` | `"disable_player_order_swap_on_rematch"` | renamed |
| `'complexity'`, `'luck'`, `'strategy'`, `'diplomacy'` | — | removed; set in the **Game Metadata Manager** (Studio UI) |
| `'is_beta'` | — | removed; managed by the release process |
| `'designer'`, `'artist'`, `'year'`, `'presentation'`, `'tags'` | — | removed; Game Metadata Manager |
| — | `"solo_mode_ranked"` | new |
| — | `"game_interface_width"` | new |
| — | `"exception_on_warning"` | new — keep `true` |
| everything else (`players`, `player_colors`, `bgg_id`, …) | same key | PHP array syntax → JSON syntax |

## Pitfalls

- **Check which file the project has** — don't create a `gameinfos.jsonc` next to an existing `gameinfos.inc.php`; convert (see `references/legacy-migration.md`) or keep editing the `.inc.php`.
- **Update the Makefile** — `DEPLOY_ROOT` must list `gameinfos.jsonc` instead of `gameinfos.inc.php` after a conversion.
- **`tie_breaker_description` must have no newlines** — breaks the JS, game won't load.
- **After any change: deploy AND reload** "game informations" via the BGA Studio manage page (same as the other config files).
- **`suggest_player_number` is not optional in practice** when the lowest player count is incompatible with default options — the automatic lobby tries the lowest count first.
