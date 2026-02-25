# SpacerCalculator — Cloud & PWA Guide

## Live URL
`https://fredaum666.github.io/SpacerCalculator/`

---

## Deploying Updates

When you push changes to GitHub, installed users **do NOT need to reinstall**.

**Flow:**
1. Edit files locally (see below for what each file does)
2. **Bump the cache version** in `sw.js` (e.g. `v1` → `v2`) — required so users get the new files
3. Push to GitHub:
   ```bash
   git add .
   git commit -m "Describe your change"
   git push origin main
   ```
4. GitHub Pages serves the new files automatically
5. Next time the user opens the app, the service worker downloads the update silently — it activates after they **close and reopen** the app once

> No reinstall needed. The PWA updates itself automatically.

---

## File Reference

| File | Purpose |
|------|---------|
| `index.html` | The entire app — all HTML, CSS, and JS in one file |
| `manifest.json` | PWA metadata (name, icons, theme color, display mode) |
| `sw.js` | Service worker — caches files for offline use |
| `icon-192.png` | App icon for Android home screen / PWA install |
| `icon-512.png` | App icon for splash screens and high-res displays |
| `SpacerCalculatorIcon.png` | Original source icon (512×512) |

### `sw.js` — bump this on every update
```js
const CACHE = 'spacer-calc-v1'; // ← change to v2, v3, etc. on each push
```

### `manifest.json` — PWA identity
```json
{
  "name": "Spacer Calculator",
  "short_name": "Spacer Calc",
  "theme_color": "#0f0e0c",
  "background_color": "#0f0e0c",
  "display": "standalone",
  "start_url": "./index.html"
}
```

---

## App Feature Reference (`index.html`)

### Design System (CSS variables, `:root`)
| Variable | Value | Used for |
|----------|-------|---------|
| `--bg` | `#0f0e0c` | Page background |
| `--surface` | `#1a1815` | Input backgrounds |
| `--card` | `#221f1b` | Result tiles |
| `--border` | `#3a342a` | All borders |
| `--amber` | `#e8a022` | Primary accent, labels, buttons |
| `--amber-dim` | `#a06a10` | Toggle switch active |
| `--amber-glow` | `#e8a02230` | Focus ring glow |
| `--cream` | `#f0e6d0` | Main text color |
| `--muted` | `#7a6e5e` | Secondary labels |
| `--green` | `#5a9e5a` | (Reserved) |
| `--red` | `#b04040` | Error border |
| `--mono` | Space Mono | All UI text (Google Fonts) |
| `--display` | Bebas Neue | Titles and result values (Google Fonts) |

---

### Inputs & Controls

#### Total Length (`#totalLength`)
- Type: `text` with `inputmode="none"` (custom keyboard)
- Accepts: `48 1/2` (inches), `4' 8 1/2` (feet + inches)
- Parsed by `parseInches(str)`

#### Spacer Width (`#spacerWidth`)
- Same format as Total Length

#### Number of Spacers (`#numSpacers`)
- Type: `number`, min `1`
- Controlled by `+` / `−` buttons (`#incrementSpacers`, `#decrementSpacers`)
- Both buttons call `calculateAndShow()` on click

#### Spacers at Ends toggle (`#endsToggle`)
- Checkbox styled as a custom switch
- When ON: adds 2 extra spacers (one at each end)
- Triggers `calculateAndShow()` on change

#### Rounding Precision (`#rounding`)
- `<select>` with values `16` (1/16") or `8` (1/8")
- Passed as denominator to `formatInches(val, denom)`
- Triggers `calculateAndShow()` on change

---

### Core Logic (JavaScript)

#### `parseInches(str) → number`
Converts a string like `"4' 8 1/2"` to decimal inches.
- Handles feet (`'`), whole inches, and fractions (`n/d`)
- Returns `0` for empty/invalid input

#### `gcd(a, b) → number`
Euclidean GCD — used to reduce fractions to lowest terms.

#### `formatInches(val, denom) → string`
Converts decimal inches back to a display string like `"8 3/8"`.
- `denom`: `16` or `8` (from rounding select)
- Outputs: `whole"`, `n/d"`, or `whole n/d"`

#### `calculateAndShow()`
Main calculation function triggered by all inputs.

**Formula:**
```
actualSpacers = numSpacers + (ends ? 2 : 0)
numSpaces     = numSpacers + 1
space         = (totalLength − actualSpacers × spacerWidth) / numSpaces
```

**Outputs:**
- `#mainResult` — each space size (highlighted, big)
- `#r-total`, `#r-width`, `#r-count`, `#r-ends` — summary tiles
- `#vizTrack` — visual bar showing spacer/space layout
- `#spacerTable` — position table (Start, Center, End per spacer)

#### `buildSequence(actualSpacers, numSpaces, ends) → Array`
Returns an array of `{type: 'spacer'|'space'}` objects representing the layout order.
- Ends ON → starts with spacer: `spacer, space, spacer, space, ...`
- Ends OFF → starts with space: `space, spacer, space, spacer, ...`

---

### Visualization (`#vizTrack`)
A flex bar where each segment's width is proportional to the total length:
- **Amber block** (`.viz-spacer`) = spacer
- **Dark block with dashed outline** (`.viz-space`) = gap between spacers

---

### Position Table (`#spacerTable`)
A `<table>` built dynamically showing:
| # | Start | Center | End |
|---|-------|--------|-----|
| 1 | 0" | ... | ... |
| 2 | ... | ... | ... |

All values formatted with `formatInches()`.

---

### Custom Imperial Keyboard

Triggered when any `.kb-target` input is focused/clicked.

**Components:**
- `.kb-overlay` — transparent fullscreen layer; click to dismiss
- `.kb-panel` — the keyboard popup, positioned directly below the input
- `#kbPreview` — live preview of the value being entered (with blinking cursor)
- `#kbDone` — closes keyboard and commits value

**State variables (inside IIFE):**
- `kbFeet` — feet portion (string, max 2 digits)
- `kbInches` — inches portion (string, max 2 digits)
- `kbFrac` — fraction string like `"3/8"` or `""` for none

**Layout:**
- Left column: Feet numpad (blue tint, `data-action="feet"`)
- Right column: Inches numpad (green tint, `data-action="inch"`)
- Bottom: Fraction grid — 16ths (purple tint) and 8ths (blue tint)

**Actions (`data-action` values):**
| Action | Effect |
|--------|--------|
| `feet` | Appends digit to `kbFeet` |
| `inch` | Appends digit to `kbInches` |
| `frac` | Sets `kbFrac` to fraction string |
| `del-feet` | Removes last character from `kbFeet` |
| `clr-feet` | Clears `kbFeet` |
| `del-inch` | Removes last char from `kbInches`, or clears `kbFrac` if inches empty |
| `clr-inch` | Clears both `kbInches` and `kbFrac` |

**`buildValue()`** assembles the current keyboard state into a string like `"4' 8 3/8"`.

On every button press, `commitToInput()` writes the value to the active input and calls `calculateAndShow()` — results update in real time as the user types.

---

### PWA Meta Tags (in `<head>`)
```html
<meta name="theme-color" content="#0f0e0c">
<meta name="mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-capable" content="yes">
<meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
<meta name="apple-mobile-web-app-title" content="Spacer Calc">
<link rel="manifest" href="./manifest.json">
<link rel="apple-touch-icon" href="./icon-192.png">
```

---

## GitHub Pages Setup (one-time)
1. Go to **Settings → Pages** in the repo
2. Source: **Deploy from a branch**
3. Branch: **`main`**, folder: **`/ (root)`**
4. Save — live in ~1 minute
