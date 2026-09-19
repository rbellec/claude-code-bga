# Tutorial-Ready Audit — BGA Reference

**Type:** Process / audit
**Doc:** https://en.doc.boardgamearena.com/Tutorials_checklist ·
https://boardgamearena.com/tutorialfaq ·
https://en.doc.boardgamearena.com/Pre-release_checklist
**When to use:** Before requesting ALPHA status, and on any game already in
alpha that has never been audited for it. The three conditions below are **not
in the Pre-release checklist** and are not surfaced anywhere in the Studio UI —
a game can go through alpha, beta and gold without anyone checking them, and
the bill arrives much later.

> **Why it matters, in one line:** on BGA a tutorial is not code you write —
> it is a *replay of a real game, annotated by a player*. If your client can't
> be driven from a replay, nobody can ever write a tutorial for your game, and
> you will only find out when a volunteer gives up halfway through.

## 1. Four facts that change the plan

| Fact | Consequence for the developer |
|---|---|
| Tutorials are built from the **"Replay game"** feature + *advanced replay features* (comment bubbles, highlights, forced moves) | there is **no tutorial API and nothing to implement** |
| *"Most tutorials are created by the community"* (BGA tutorial FAQ) — any player can write one, and earns up to 600 points for a published one | the tutorial is **not your deliverable**; making the game *tutorial-able* is |
| A tutorial needs the **replay of a game actually played** | a tutorial **cannot precede** production. Order is: alpha → real game played → tutorial. Never the reverse |
| Tutorials are published on player ratings (avg ≥ 4.2; ≥ 10 ratings for BETA, ≥ 100 for LIVE; below that an admin must intervene) | a tutorial on a freshly-alpha game **cannot be published yet** — don't put one on a release plan |

So the developer's job is exactly this file: make sure that **when** someone
tries, they can.

## 2. The three conditions (verbatim) and what breaks each

From the BGA *Tutorials checklist*, a game is tutorial-able iff:

1. **the game replay archive must be valid** — the replay runs from start to
   finish without errors via *"Replay game"* on a finished table;
2. **each action of the replay must be replayable by triggering it manually** —
   the tutorial author reproduces the original sequence by clicking, so every
   recorded action must be reachable from the UI;
3. **each element of the interface must be available for attaching a comment
   or a highlight** — the author anchors bubbles on your DOM nodes.

The doc also names the causes, which is what makes this auditable:

| Condition | Named cause | Detectable by |
|---|---|---|
| ① / ② | `this.ajaxcall()` / `this.bga.actions.performAction()` called **programmatically** from a notification handler or a callback instead of from a user interface action | **static analysis** (§3) |
| ② | **several ways to trigger the same action** with different URL/arguments | **static analysis** (§3) |
| ③ | **overlapping elements** hiding the `div` of an important element | **DOM audit** (§4) |
| ③ | **missing `ID`** on interface elements | **DOM audit** (§4) |

> This is the deeper reason behind the SKILL.md rule *"`performAction` only
> from user events — NEVER from notifications, loops, callbacks, or state
> methods"*. That rule is usually justified by double-submits and race
> conditions; tutorial-ability is the second, later, more expensive reason.

## 3. Audit A — static (minutes, no browser)

### A1. Action calls fired from a non-user context

List every call site with its enclosing function (heuristic, good enough to
triage — read each hit before judging):

```bash
# from PROJECT_DIR
rg --line-number 'performAction\(|ajaxcall\(' -g '*.js' -g '!**/node_modules/**' \
  | tee /tmp/bga-action-callsites.txt

# same, annotated with the enclosing method name (control blocks such as
# `if (...) {` / `for (...) {` match the same shape — they are skipped so the
# last real method header wins)
awk '
  /^[[:space:]]*(async[[:space:]]+)?(function[[:space:]]+)?[A-Za-z_$][A-Za-z0-9_$]*[[:space:]]*\(.*\)[[:space:]]*\{/ {
      fn=$0; sub(/^[[:space:]]*/,"",fn); sub(/^async[[:space:]]+/,"",fn)
      sub(/^function[[:space:]]+/,"",fn); sub(/[[:space:]]*\(.*$/,"",fn)
      if (fn !~ /^(if|for|while|switch|catch)$/) enclosing=fn
  }
  /performAction\(|ajaxcall\(/ { printf "%s:%d  [%s]\n    %s\n", FILENAME, NR, enclosing, $0 }
' $(rg -l 'performAction\(|ajaxcall\(' -g '*.js' -g '!**/node_modules/**')
```

**Flag as a violation** when the enclosing function is any of:

- `notif_*`, or anything registered in `setupNotifications` /
  `bgaSetupPromiseNotifications` / `subscribeNotif`
- `onEnteringState`, `onLeavingState`, `onUpdateActionButtons`, `setup`,
  `constructor`
- a `setTimeout` / `requestAnimationFrame` / `.then()` / `await` continuation
  that is not rooted in a DOM event handler
- a `forEach` / `for` loop body (batch-firing actions)

**Not a violation — read the hit before judging:** the enclosing method only
tells you where the code *lives*, not what *fires* it. A call that sits inside
`onEnteringState` / `onUpdateActionButtons` / a `for` loop but is the callback
argument of `addActionButton(...)`, `addEventListener('click', ...)` or an
equivalent user-event registration is user-rooted and fine — the closure is
*defined* there, it is *fired* by the click. The violation is the call that
runs synchronously when the method runs, or from a timer / promise
continuation it starts.

**The classic offender:** auto-play convenience — *"only one legal move, so
let's play it for the player"* from `onUpdateActionButtons`. It is invisible in
normal play, it is a hard blocker for a tutorial, and it is also a race
condition waiting to happen.

### A2. One game action, one call shape

```bash
# how many distinct call sites per action name
rg -o "performAction\(\s*'([^']+)'" -r '$1' -g '*.js' | sort | uniq -c | sort -rn
```

Any action appearing more than once is **suspect, not guilty**: two call sites
are fine if and only if they send the *same* action name with the *same*
argument keys. Two UX paths to the same move (click the card **and** click the
"Play" button) must converge on **one** helper:

```js
// BAD — two shapes for one move: the replay can't be re-triggered manually
onCardClick(id)   { this.bga.actions.performAction('actPlay', { id }); }
onPlayButton()    { this.bga.actions.performAction('actPlay', { card: this.selected, from: 'hand' }); }

// GOOD — one funnel, one shape
playCard(id)      { this.bga.actions.performAction('actPlay', { id }); }
onCardClick(id)   { this.playCard(id); }
onPlayButton()    { this.playCard(this.selected); }
```

### A3. Debug paths must not be the only path

```bash
rg -n '#\[PossibleAction\]' -g '*.php' | rg -i 'debug'
rg -n 'debug_' -g '*.js'
```

Temporary `debug_*` actions are fine during development but must be removed
before alpha (SKILL.md, *Debug tips*). An action that can **only** be reached
through a debug button does not exist for a tutorial author.

### A4. Nothing the client invents

A replay replays the **notification stream**. Anything the client makes up
locally is not in the archive and will differ between the original game and the
replay:

```bash
rg -n 'Math\.random|Date\.now|new Date\(' -g '*.js' -g '!**/node_modules/**'
```

Cosmetic jitter is tolerable; anything that changes *which element exists* or
*what its id is* is not (see §5).

## 4. Audit B — DOM (one live table)

Run against a table in the normal test loop (SKILL.md §3.2–3.5), in the
**player view**, mid-game — not on the lobby, not on `/tableview`.

```javascript
(() => {
  const root = document.getElementById('game_play_area') || document.body;
  const noId = [], covered = [];
  for (const el of root.querySelectorAll('*')) {
    const r = el.getBoundingClientRect();
    if (r.width < 8 || r.height < 8) continue;              // skip decorations
    if (r.bottom < 0 || r.top > innerHeight) continue;      // must be on screen
    if (!el.id) { noId.push((el.tagName + '.' + el.className).slice(0, 60)); continue; }
    const top = document.elementFromPoint(r.left + r.width / 2, r.top + r.height / 2);
    if (top && top !== el && !el.contains(top) && !top.contains(el)) {
      covered.push(el.id + '  <-- covered by  ' + (top.id || top.className));
    }
  }
  console.log('[tutorial-audit] sizeable elements without id:', noId.length, noId.slice(0, 30));
  console.log('[tutorial-audit] elements covered by another node:', covered.length, covered.slice(0, 30));
})();
```

Then read it back with
`mcp__claude-in-chrome__read_console_messages(tabId, pattern: "tutorial-audit")`.

**What must have a stable `id`** — the things a tutorial will point at:

- each board cell / slot / space
- each card, tile, token, meeple currently visible
- each player panel and each counter inside it
- every action button (BGA generates ids for framework buttons — check yours)
- the hand / market / stock containers, the log, the status bar

**Caveats of this snippet** (it triages, it does not certify): elements are
only checked at the current scroll position and viewport width; a `.zone` that
legitimately hosts children will report its children as "covering" it — read
the list, don't count it.

### B1. Id *stability* (the one the snippet can't see)

Run the snippet on **two different tables**, dump the id sets, and diff:

```javascript
console.log('[tutorial-audit-ids]', JSON.stringify(
  [...document.querySelectorAll('#game_play_area [id]')].map(e => e.id).sort()));
```

Ids must be derived from game data (`card_17`, `cell_3_4`, `player_panel_82`),
**never** from a render counter or a random suffix (`elem_7`, `tok_a9f3`).
An id that changes between two games of the same position means every tutorial
anchor breaks on the next replay.

## 5. Audit C — replay (the real proof)

1. Play a hotseat game to a real end (SKILL.md §3.4; use two tabs, one per
   player, or the game will self-abandon — §3.8).
2. From the **finished table page**, take the *"Replay game"* link (grab the
   href from the page rather than typing a URL by hand; the Studio archive path
   has changed in the past).
3. Watch the replay from step 0 to the end, then:
   `mcp__claude-in-chrome__read_console_messages(tabId, pattern: "error|Error|fatal|undefined")`.
4. Step through it **one action at a time** with the replay controls and check
   that each step advances the display. A step that renders nothing is a
   notification your client ignores — the tutorial author has nothing to
   comment on there.

Green = the archive is valid **and** the client can be driven from it. This is
condition ① proved, and most of ② observed.

## 6. What a failure costs later

If the game is already in production and a tutorial author hits any of this,
the fix is a **new version of the game** — and *"you will have to rewrite the
entire tutorial using a replay based on a game played with this new version"*.
There is an escape hatch, **Tutorial game release override** (Game Metadata
Manager → *Tutorials*, or a BGA admin), but it only works if the new version
stays **compatible with the move notifications sent by the previous one**: add
a parameter the client now expects, and the override silently breaks the
replay. Which is the real argument for running this audit *before* alpha.

## 7. Report template

Have the audit produce this, not prose:

```markdown
## Tutorial-readiness — <game>, <date>, version <n>
① replay valid ........... PASS / FAIL  (console: <n> errors)
② manual re-trigger ...... PASS / FAIL  (<n> non-user call sites, <n> multi-shape actions)
③ anchorable interface ... PASS / FAIL  (<n> sizeable nodes without id, <n> covered, ids stable: yes/no)

### Blockers
- <file:line> — performAction fired from notif_xxx → move to the click handler
### Watch list
- <selector> — covered by <overlay> at width < 900px
```

## Pitfalls

- **Passing for the player.** Any "there is only one legal move, play it
  automatically" shortcut kills condition ②. Offer a pre-selected button
  instead; let the player click it.
- **The full-size transparent overlay** (drag layer, highlight layer, modal
  backdrop left at `opacity: 0`) — invisible in play, and it hides the entire
  board from the tutorial editor. `pointer-events: none` does **not** help: the
  author still can't select what's underneath in the editor.
- **Ids assigned by a counter** — see §B1. The single most common reason a
  tutorial breaks on the *second* replay rather than the first.
- **Elements that exist only at some widths** — a responsive layout that swaps
  DOM (rather than restyling it) gives the author anchors that vanish for half
  the readers.
- **Auditing on `/tableview`** — the spectator view has no `game_play_area`
  and no action buttons (SKILL.md §3.5). The audit must run in the player view.
- **Treating the tutorial as a release blocker.** It is not, it cannot be
  written before the game is playable in production, and it cannot be published
  before it has ratings. Ship the game; make it tutorial-able on the way out.

## Status of this reference

First draft, written from the BGA tutorial FAQ and the *Tutorials checklist*
doc page. §§1–2 and §6 are quoted from those sources. §§3–5 are **derived
procedures, not documented by BGA** — they have not yet been run end-to-end on
a real project. Refine the greps and the DOM snippet against the first real
audit, then delete this notice.
