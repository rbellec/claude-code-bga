# BGA Studio Guidelines — Reference

**Type:** Design & Review Standards
**Doc:** https://en.doc.boardgamearena.com/BGA_Studio_Guidelines
**When to use:** Polishing UI before review, checking accessibility, ensuring BGA conventions are followed.

## Layout

- **Center play area** on all devices; leave margins to prevent mobile misclicks
- **Min 1024px width** support; play area max ~750px on smallest displays
- **Stack vertically on mobile** — secondary elements below main board
- **White background divs with h3 titles** for content organization
- **Player panels:** resources, scores, first player token; max 1/4 screen height for 4+ players on mobile
- **Status bar:** max 4 buttons + context

## Interactive Elements

- **Whole game playable with left-click only** — no context menus required
- **Drag-and-drop only with click alternative**
- **Pointer cursor** for all clickable elements
- **Min tap target: 32x32px**, ideal 40-44px, with sufficient spacing
- **Highlight active areas** — use outline/shadow/filter, not border
- **Colors:** white/yellow/blue for selection (NOT red — red = error)
- **Grey buttons** for unavailable but relevant actions
- **Every clickable element must respond** — action or explicit error message

## Animations

- **Standard duration: 0.5s**, max 0.8s
- **Animate moving elements** — show point/resource animations from source
- **Don't over-animate** — keep it functional
- **End-game scoring:** use synchronous notifications, animate point sources
- **Check `bgaAnimationsActive()`** — skip animations during instant replay

## Sounds

- **Lower volume than BGA default**
- **Short, sharp, purposeful**
- **Pair with visual cues** — game must be playable in silence

## Accessibility

- **Don't rely on color alone** — use icons/textures/shapes
- **Min contrast 4.5:1** (WCAG AA)
- **Colorblind mode:** add unique shapes/symbols for colors; offer as preference
- **No flashing/rapid visuals**
- **Label all buttons** for screen readers

## CSS Conventions

- **Prefix CSS classes** with game name: `mygame_selected`, `mygame_token`
- **Use CSS sprites** for game elements (single image, background-position)
- **CSS class selectors** for animated elements (not ID — animations clone elements)
- **`#overall-content` gets state class**: `.gamestate_playerTurn` for conditional CSS without JS
- **Thematic background** — subtle playing mat texture, not distracting

## Game Log

- **Every important move** must be in the log
- **Players should understand the game story** from logs alone
- **Use icons, colors, formatting** for scanning
- **Specify who acted and what changed**

## Faithful Representation

- **Minimize modifications** from original board game
- **Reduce size rather than omit** components
- **Use tooltips** for smaller components
- **Don't indicate good/bad moves** — showing available moves is OK
- **All player-visible info must be accessible** — opponent hand sizes, deck counts

## Technical Quality

- **F5 refresh must restore exact state** — server is truth, refresh invisible to players
- **Hidden elements visible to owner only** — `getAllDatas()` must not return hidden info
- **Calculate game progression accurately** (0-100%) — accounts for multiple end conditions
- **Handle disconnects smoothly** — short, friendly messages; never expose stack traces
- **Namespace everything:** PHP namespace, CSS prefix, no global JS vars

## Code Conventions

- **Prefix state functions: `st`**, arg functions: `arg`, action functions: `act`
- **PSR-4 namespacing** with auto-loading
- **Don't use `.action.php`** — use autowiring
- **Validate all action inputs**
- **Never suppress errors** with `@` operator
- **Never use undocumented BGA functions**

## Button Colors

| Color | Usage |
|-------|-------|
| Blue (primary) | Advance, confirm, positive action |
| Red (alert) | Cancel, undo, stop, destructive |
| White/Gray (secondary) | Optional, pass, skip |
| Gray (disabled) | Unavailable but relevant |

## Publication Checklist

- **BETA:** >= 4.2 avg rating, >= 10 ratings
- **LIVE:** >= 4.2 avg rating, >= 100 ratings
- **Tutorial:** valid replay archive, replayable actions, interface elements accessible for tutorial attachment
