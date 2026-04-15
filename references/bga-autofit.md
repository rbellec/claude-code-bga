# BgaAutofit — BGA Library Reference

**Type:** JS Client (Modern ESM)
**Doc:** https://en.doc.boardgamearena.com/BgaAutofit
**When to use:** Auto-scale text to fit fixed-size containers — useful for translated text on game cards or tiles.

## Setup

```javascript
const BgaAutofit = await importEsmLib('bga-autofit', '1.x');
BgaAutofit.init(); // call once — works for all future elements too
```

## Usage

Just add the `bga-autofit` CSS class to any container with a fixed size:

```html
<div class="my-card-title bga-autofit">${translatedText}</div>
```

```css
.my-card-title {
    width: 100px;
    height: 24px;
    overflow: hidden;
}
```

## Pitfalls

- **Call `init()` only once** — it uses MutationObserver internally, so new elements added later are detected automatically
- **No reinit needed** — elements added after `setup()` will also be auto-fitted
- **Semantic versioning** — use `'1.x'` for stability
