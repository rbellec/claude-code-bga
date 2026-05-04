# BGG Submission Helper — BGA Reference

**Type:** Process / external system
**Doc:** https://boardgamegeek.com/wiki/page/BGG_Guide_to_Game_Submissions
**When to use:** Producing a BoardGameGeek entry to obtain the `bgg_id`
referenced in `gameinfos.inc.php`. Optional — typically triggered post-beta
when the game is ready to be public on BGG.

> **Source page is Cloudflare-protected.** WebFetch and curl return 403.
> Use Claude in Chrome, or paste the wiki text into the chat, when the
> guide needs to be re-checked.

## Inputs already available in a BGA project

- `gameinfos.inc.php` → `players` (min/max), `estimated_duration`, current
  `bgg_id` (likely `0` until this submission completes), `game_name`
- `doc/RULES.md` → setting, goal, gameplay
- `doc/BGA_DESCRIPTION.md` (if produced earlier) → long description draft;
  needs adaptation (see section below)
- Designers / artists names — gather from credits or ask the author

## Workflow

Produce a single file `doc/BGG_SUBMISSION.md` listing every BGG field below
with the proposed value, ready to copy-paste field-by-field into the BGG
submission form. Mark intentionally empty fields as `(leave blank)` so
nothing is forgotten.

Fields are numbered as in the BGG guide.

1. **Primary name** — game title. Use `Title: Subtitle` (with colon) when
   applicable. For non-Latin scripts, add Latin transliteration in parens.
2. **Description** — 1–4 paragraphs, neutral voice, English, covering
   *setting + goal + gameplay*. **Forbidden:** marketing copy, component
   lists, links to external sites, stubs of 1–2 lines, copy-pasted press
   release without trimming, non-English text without an English version
   above it. If reusing publisher copy, end with
   `''—description from the publisher''` (double apostrophes = wiki
   italics). If a non-English version is included, separate it from the
   English one with `•••`.
3. **Short description** — **max 85 characters**, single sentence.
   - Omit the game name (it's already in the title field).
   - Normal capitalization, normal punctuation, no emoji, English.
   - Don't list mechanisms ("A card-drafting deck-building game!").
   - Don't sell only the theme ("Monsters, monsters, more monsters!").
   - Ideal pattern: one concrete hook (the twist, the choice, the
     setting) expressed as something a player **does**.
4. **Embargo Date/Time** — only if unpublished; blocks image upload until
   it passes.
5. **Year released** — first general retail availability. Crowdfunding
   advance copies don't count unless there's no later retail release. For
   BGA-only digital releases, use the BGA Studio public-release year —
   confirm with the author.
6. **Min/Max players** — printed on the box, or `gameinfos.inc.php['players']`
   (the array gives both bounds).
7. **Min age** — printed on the box. Leave blank if unspecified — don't guess.
8. **Min/Max playing time** — minutes. From `estimated_duration` (note: may
   be in seconds depending on framework version — convert).
9. **Category and mechanism** — pick 2–4 of each from BGG's controlled
   vocabulary. Common patterns for BGA games:
   - **Categories:** Abstract Strategy, Card Game, Real-time, Party Game,
     Bluffing, Deduction.
   - **Mechanisms:** Hand Management, Pattern Building, Pattern
     Recognition, Tile Placement, Grid Movement, Square Grid, Action
     Drafting, Action Points, Take That, Push Your Luck, Set Collection,
     Auction, Worker Placement, Variable Phase Order.
10. **Family** — leave blank unless an obvious thematic/series family fits.
11–14. **Expands / Integrates / Contains / Reimplements** — usually blank
    for a new standalone game.
15. **People** — designer(s), artist(s), graphic designer(s). If a person
    isn't yet in BGG's database, submit them first or in parallel and link
    to the pending submission. Use `(Uncredited)` if needed; don't list a
    designer in a non-design role unless explicitly credited.
16–24. **Version info** — version nickname (`(language) edition`, or
    `ENG/FRE/GER edition` for multilingual; ISO 639-2/B codes), publisher,
    artist, year, product code, dimensions (largest = length, smallest =
    depth), weight, languages. For self-published or web-released BGA-only
    games, choose **Self-published** or **Web published** — do not list
    the BGA platform as publisher.
25–30. **Release date / comment / status / pre-order** — fill if applicable.
31. **Note to admin** — alternate-language titles, link to publisher
    announcement, mention if you are the designer/publisher (so admins
    know who to contact), explain how the entry differs from a similar
    one, link to the BGA Studio page if relevant.
32. **Save**.

## Adapting `BGA_DESCRIPTION.md` → BGG description

A long description written for the BGA Studio "long description" field is
*close* to a BGG description but needs three changes:

1. **Strip marketing closers** ("Fast and tactical!", "Every card
   matters!"). BGA tolerates them; BGG declines them.
2. **Strip component lists.** BGA tutorials list card counts; BGG asks you
   to put components in the Community Wiki section *after* the listing is
   approved, never in the description.
3. **Bullet lists → prose.** BGG descriptions are paragraph-form. Fold
   bullet lists of card/action types into a single paragraph that names
   the actions inline.

Keep both artifacts — they are not interchangeable.

## Pitfalls

- **Wiki source can't be re-fetched** with WebFetch (Cloudflare 403).
  Use Claude in Chrome or paste the text when verifying the latest guide.
- **`bgg_id` stays `0`** until BGG admins approve the entry (can take
  days). The "This game doesn't have a valid BGG_ID" Studio warning is
  expected during this window.
- **Don't list BGA as publisher.** Use Self-published or Web published.
