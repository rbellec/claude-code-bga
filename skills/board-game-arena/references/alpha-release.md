# Requesting Alpha Status — BGA Reference

**Type:** Process / Studio UI
**Doc:** https://en.doc.boardgamearena.com/BGA_game_Lifecycle ·
https://en.doc.boardgamearena.com/Pre-release_checklist ·
https://en.doc.boardgamearena.com/Game_metadata_manager#Media
**When to use:** The project plays end-to-end and you're ready to leave
"in development" — the Pre-alpha → Alpha transition of the BGA lifecycle
(the one transition the developer triggers themselves).

> None of the three doc pages above are linked from the Studio UI — you
> have to know they exist. `BGA_game_Lifecycle` describes the 5 stages
> (Initial → Assigned → Pre-alpha → **Alpha** → Public Beta → Gold) and who
> triggers each transition; `Pre-release_checklist` is the authoritative
> checklist (license, media, server/client code, UI, testing, cleanup).

## Where the button actually is

Not obvious from the Studio UI:

1. `studiogame?game=GAMENAME` → section **"Build a new release version"**
   → **"Commit & build a new version now"** (requires a prior `make deploy`).
2. Once a version is built, a **"Request ALPHA status"** button appears
   next to it, under **"Versions available for production"**.
3. Clicking "Deploy to production" before being in alpha — the natural
   first instinct — gives an explicit redirect message: *"This game is not
   in production yet: it requires a BGA admin for the first deployment...
   Please check the 'BGA studio guidelines' and the 'Pre-release
   checklist', then click the 'Request Alpha' button."*

## Licensing is a hard gate — even for alpha

BGA must have a license for the game before any move to production, alpha
included. For games with no commercial publisher (public-domain rulesets,
academic sources, traditional/folk games) there is **no self-service way**
to declare this in the Studio UI. It's resolved by replying to the email
ticket generated when "Request ALPHA status" is first clicked:

- cite the source of the rules (academic publication, traditional origin)
  and why they aren't copyrightable subject matter;
- state that no publisher exists;
- request classification as **"(public domain)"** — a real classification
  value visible in the License column of the Studio `/projects` listing.

## Images — only two are mandatory for alpha

Don't produce the full asset set up front:

| Image | Dimensions | Format | Transparent | Required for |
|---|---|---|---|---|
| Box | 280×280 | PNG | Yes, **no shadow** | **Alpha** |
| Icon | 50×50 | PNG | not required | **Alpha** |
| Title | 2000×2000 | PNG | Yes | Beta |
| Banner | 1386×400 | JPG | No — and **must contain no text** | Beta |
| Publisher | 280×280 | PNG | Yes | Optional (skip if no publisher) |
| Display | height 400–760, width ≤ 1.5×height | JPG | No | Release |

Upload via Game Metadata Manager → Media. **Each slot's upload modal
restates its own spec, sometimes more precisely than the doc page** (e.g.
the modal adds "without any shadow" for the Box image) — trust the modal
text over the doc page when they seem to disagree.

Box art guidance from the modal itself: *"It should be a 3D image of a
physical copy of the game box as it appears in an online shop... If you
don't have a 3D version of the game box, you can use
http://www.3d-pack.com/ to create one."* For abstract/CSS-only games with
no illustrated art, a simple SVG mock — front face + a darker left "spine"
strip for a 3D-book illusion, no ground shadow — rendered to PNG with
`rsvg-convert -w <W> -h <H> in.svg -o out.png` satisfies the spec fine; no
real 3D renderer needed.

Generate the asset locally and let the user upload it through the normal
browser UI — automated file upload from the browser extension is
unreliable for spec-strict assets (see SKILL.md §3.8).

## Pre-flight warnings on "Request ALPHA status"

Clicking the button opens a dialog listing several items, most dismissible
via a per-item "Ignore" link — but at least one is a hard blocker:

- Project technical name ≠ reduced display name — informational, ignorable
  if intentional
- Publisher BGG id unset — ignorable when there's genuinely no publisher
- "N tags set, sync from production if added there" — ignorable when the
  project has never been deployed to production (nothing to sync)
- **`zombieModeLevel` missing** — enforced as *mandatory* metadata, not
  just a warning. Fill Game Metadata Manager → **Metadata** tab → "Zombie
  Mode level" (free-text numeric, `0`–`3`) — see the Zombie handling
  section of SKILL.md for the levels and the honesty rule.

## Undocumented hard requirement: at least 2 tags

After clearing the warnings, the request can still fail at submit time
with `ERROR: only one game tag found — the game tags are not set.` — an
enforced minimum of (at least) 2 tags on the project, mentioned neither in
the Pre-release checklist nor in the warnings dialog. Fix: Game Metadata
Manager → **Tags** tab → "+" → use the **"Find a tag"** search box and
pick a genuinely-fitting tag (e.g. "Abstract games" for a rules-only
abstract game).

**Tags UI trap:** in the "+" tag picker, the category rows (Complexity /
Specific / Theme / Mechanism / Admin) expand into a flat tag list on click
— but clicking directly on a category's own colored pill/label (not the
caret) can immediately **add an unrelated tag** with no confirmation step.
Always use the search box instead of browsing categories. To undo a wrong
tag: click the tag chip in the main Tags row → a popover shows tag
ID/order → **"Remove tag"**.

## Server-side checklist items easy to miss

From the Pre-release checklist, the requirements most often absent from a
project that only ever ran in dev:

- **`giveExtraTime($playerId)` after each player action** — no-op in
  turn-based mode but required for real-time tables
- **Player statistics** — a project often has table stats only; add
  meaningful per-player stats
- **Remove debug `console.log()`** from the JS client
- **Minified build test** — click "Use minified JS" / "Use minified CSS" on
  the manage page (after a Studio commit) and play; a game that only runs
  unminified gets stuck at "Connecting to game" in alpha
- **Dark mode** — readable with `html[data-theme="dark"]` (see
  `references/guidelines.md`)
- **Spectator test** with all test accounts 0–9 in the players group;
  in-game replay from the log; full "Replay game" after a finished table
- **Translations** — `&dummyTranslations` on a table URL, "Scan
  translations" in a dry-run build (`references/translations.md`)
- **TypeScript/SCSS sources** live in `src/`, not `modules/` (only built
  files are deployed)

## Before you click: the tutorial-readiness audit

Three further conditions — valid replay archive, every action re-triggerable
by hand, every interface element carrying an `ID` and not covered — live on a
**separate doc page that the Pre-release checklist never mentions**
(https://en.doc.boardgamearena.com/Tutorials_checklist). They don't block
alpha, which is exactly why they get discovered a year later, when a volunteer
tutorial author gives up and the fix requires a new game version. Run
`references/tutorial-ready.md` now — it is minutes of static analysis plus one
replay.

## Workflow summary

1. Work through the Pre-release checklist (code requirements, stats, cleanup)
2. Produce Box (280×280) + Icon (50×50) PNGs, upload via Game Metadata
   Manager → Media
3. Fill "Zombie Mode level" (Metadata tab); ensure ≥ 2 tags (Tags tab)
4. `make deploy` → "Commit & build a new version now"
5. "Request ALPHA status" → handle the warnings dialog → submit. The
   Studio walkthrough page (revised Sept 2026) describes two buttons, **"Request
   PRIVATE ALPHA status"** (admins check the game starts, you invite
   testers yourself) and **"Request PUBLIC ALPHA status"**; the Pre-release
   checklist still names a single "Request ALPHA status" — use whichever
   the manage page shows
6. If the game has no publisher: reply to the generated ticket with the
   licensing rationale (public domain template above). The ticket reply
   also carries your **production BGA user name** (to be made game admin
   in prod) and, for a private alpha, the other admins' user names

## Pitfalls

- **The 2-tag minimum only surfaces at submit time**, after a version
  rebuild — check tags before building to avoid a wasted cycle.
- **`zombieModeLevel` declares one number for the whole game** — make
  every state's `zombie()` honor it (SKILL.md, Zombie handling).
- **Trust the upload modal's spec over the doc page** when they disagree.
