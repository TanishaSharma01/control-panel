# `reactfront/src/index.css`

Source: [`reactfront/src/index.css`](../../reactfront/src/index.css) (396 lines)

## 1. Purpose

The single global stylesheet (imported by `index.js`). Layout, the header bar, the P&ID background, panel sizes, switch colours and the debug log.

## 2. Structure by logical block

| Lines | Selectors | Role |
|---|---|---|
| 1-6 | `:root` variables | `--top-bar-height: 120px`, `--top-bar-color: rgba(18,43,81)`, page background tint |
| 8-26 | `:root, body`, `code`, `html, body, #root…` | Fonts, full height, **`zoom: 0.94`** on the whole page |
| 28-54 | `.panels-root`, `.panel`, `.main-layout`, `.left-panel`, `.control-with-progress` | Page layout below the fixed header: flex row of left view + graphs |
| 56-84 | headings, `.flex`, `.active/.inactive`, `.frame` | Typography and helpers |
| 86-103 | `.control-panel-scale-wrapper`, `.control-panel` | **P&ID canvas**: 790 × 569 px, `background-image: url(labjack.svg)`, `background-size: 112%`, `background-position: -75px 20px`, `transform: scale(0.89)` from the top-left, clipped to 800 × 500 |
| 105-120 | `.control-panel-label(.eth/.lox)` | "ETH" / "LOX" captions |
| 122-149 | `.panel.safety/.control/.sequence-execution/.sequences/.calibration/.graphs` | Fixed pixel sizes for each view |
| 151-207 | Material-UI overrides | Switch placement inside the diagram, legend switch colours, table cells, `.control-label` |
| 209-219 | `.jump-to-present` | Unused |
| 221-327 | `.top-bar…` | Two-row fixed header, toggles, ABORT button, view buttons |
| 329-357 | `.sensor-…` | Card text sizes (`.sensor-value-display`, `.sensor-voltage-display` are used; the others are not) |
| 359-396 | `.debug-log-…` | Fixed bottom console (dark, monospace, max height 180 px, `z-index: 10000`) |

## 3. Coupling with code

- Widget coordinates in `pins.json` are multiplied by 26 px in `control-panel.js` and are relative to `.control-panel`. They are tuned to this exact background size/position and scale; changing `background-size`, `background-position` or the `transform` shifts every switch and card relative to the drawing.
- The left views and the graph panel have fixed widths (710 + 240 px control view, 650 px graphs), so the layout is designed for a wide desktop screen; `zoom: 0.94` is a non-standard property (supported by Chromium/WebKit, and by recent Firefox).
- The debug log is fixed to the bottom and overlays page content while expanded.
- `Panel` in `components/index.js` passes its `className` through, so the size rules are selected with `className='panel control'` etc.

## 4. Unused rules

`.panel.safety`, `.panel-row-3`, `.alert`, `.jump-to-present`, `.top-bar-status-text`, `.sensor-batch-info`, `.sensor-collection-info`, `.sensor-flow-active`, `.sensor-pressure-high` are not referenced by any component (card colours are set inline in `control-panel.js`).

## 5. Inputs / outputs / errors

Static CSS; no runtime behaviour. The only asset reference is `labjack.svg`, resolved by webpack at build time (the build contains `static/media/labjack.<hash>.svg`).
