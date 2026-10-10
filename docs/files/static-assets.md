# Static assets

Non-code files that are part of the application or repository. None contain logic.

| File | Size | Used by | Notes |
|---|---|---|---|
| [`reactfront/src/labjack.svg`](../../reactfront/src/labjack.svg) | 1270 lines | `index.css` L93 (`.control-panel` background) | The P&ID drawing of the test stand (Inkscape; document name "labjack Alt NoButtons"). Valve switches and sensor cards from `pins.json` are overlaid on it. Editing the drawing's geometry requires re-tuning the `x`/`y` values in `pins.json` |
| [`reactfront/src/labjack-1.svg`](../../reactfront/src/labjack-1.svg) | 611 lines | **nothing** | An older/alternative version of the diagram; not referenced |
| `reactfront/public/logo.png` | – | `TopBar` (`<img src='./logo.png'>`) | Header logo |
| `reactfront/public/favicon.ico` | – | browser default request | Tab icon |
| `screenshot.png` | – | root `README.md` | Screenshot of the UI |
| `bin/images/on-2.png`, `off-2.png`, `raptor.png` | – | **nothing found** in the source | Legacy images |
| `reactfront/build/**` | – | served by the Pi (`startup.sh`) | Generated output of `npm run build`, committed to git. Do not edit by hand. At the time of writing the working tree contains a newer, uncommitted build than the committed one |

## Relationship to the code

```
labjack.svg ──(CSS background)──► .control-panel ◄──(absolute positions × 26 px)── pins.json
```
