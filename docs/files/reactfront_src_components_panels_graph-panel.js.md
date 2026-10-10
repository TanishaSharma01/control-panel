# `reactfront/src/components/panels/graph-panel.js`

Source: [`reactfront/src/components/panels/graph-panel.js`](../../reactfront/src/components/panels/graph-panel.js) (647 lines)

## 1. Purpose

The live graphing system: a `Datalogger` factory that builds an interactive, hand-drawn SVG time-series graph around a private data store; five instances of it (pressure, flow, load cell, raw voltage, temperature); the `GraphPanel` that lays them out with shared SVG/CSV download controls; plus the `pinFromID` lookup and the `newData`/`newEvent` ingestion functions used by the rest of the app.

## 2. Imports (L1-13)

`React`; `Panel`; `disablePageScroll`, `enablePageScroll` from `scroll-lock` (also attached to `window`); from `graph-utils`: `generateAxisTicks`, `binarySearch`, `formatUnit`, `lerp`, `clamp`, `intervalUnion`, `expandInterval`, `interpolateInterval`, `boundIntervalKeepingLength`, `undefOnBadRef`; `SeriesCollection` from `series`; `pins.json`.

## 3. Module-level definitions

### `pinFromID(labjack_pin, test_stand = null)` (L16-28, exported)
Returns the `pins.buttons` entry whose pin (and stand, if given) match; falls back to the first entry with that pin. Only valves are searched.

### Y-range state (L31-40)
`pressureYBounds`, `flowYBounds`, … start as `null`; minimum ranges: pressure ±0.7 bar, flow ±0.2 LPS, force ±0.5 kg, voltage ±0.1 V, temperature ±5 °C. A graph never zooms in tighter than its minimum range.

### `newData(detail)` / `newEvent(detail)` (L45-48, exported)
Dispatch DOM `CustomEvent`s `datalogger-new-data` (`{time, <series>: value, …}`) and `datalogger-new-event` (`{time, label}`) on `document`.

## 4. `Datalogger({ series, hiddenSeries, unit, label, yBoundsRef, minYBoundsRef })` (L50-498)

A **factory** executed once per graph at module load. It returns a `React.forwardRef` component.

### Closure (per graph, outside React)

- `store = new SeriesCollection({...series, ...hiddenSeries})` – hidden series are stored and exported but not drawn.
- `events = []`.
- Listener for `datalogger-new-data`: for each of this graph's keys present in the event, `store.arrays[key].push(detail.time, detail[key])`; then schedule a redraw with `requestAnimationFrame`.
- Listener for `datalogger-new-event`: append; redraw.

### Component render – logical blocks

| Block | Lines | What it does |
|---|---|---|
| Size | 88 | 290 × 230 px: main plot 180 px high + 50 px overview strip |
| Time base | 93-94 | `currentSeconds` prop (server time) or local clock; `startSeconds = store.minTime()` |
| Window model | 99-119 | `window` is `[tMinus]` (sliding – follows "now", default last 3 s) or `[start, end]` (fixed). `setWindow` clamps and converts a window touching "now" back to sliding form |
| Sampling | 121-122 | `points = store.sample(window, 290)` (decimated to roughly ≤ 290 points per series); `preview = store.samplePreview(290)` |
| Y auto-scale | 125-126 | Bounds move 10 % per render towards the union of data min/max and the minimum range, then are padded by 20 % |
| Coordinate transforms | 130-145 | `v2x`, `v2y`, inverses, and the overview-strip equivalents |
| Ticks | 148-149 | `generateAxisTicks` for time (≈ 4) and value (≈ 6) |
| Tooltip | 153-175 | Binary search for the sample left of the cursor per series; linear interpolation (`lerp`); text via `formatUnit` |
| Wheel zoom | 180-193 | Zoom factor `1.001^delta` about the cursor (or the right edge in sliding mode); page scroll is locked while hovering |
| Drag pan | 195-223 | Mouse drag shifts the window, bounded to the data extent |
| `downloadSVG` | 225-234 | Serialises the main `<svg>` to `graph-<label>.svg` |
| `downloadCSV` | 236-324 | See §5 |
| `useImperativeHandle` | 328 | Exposes `{downloadSVG, downloadCSV}` to the parent via `ref` |
| Event markers | 330-333 | Visible events → x positions; label text only when the window is < 15 s wide |
| Overview strip handlers | 348-380 | Drag the selected region or its resize handles |
| JSX | 382-494 | Gridlines, axes, event lines, min/max shaded polygons (only for decimated segments), polylines, tick labels, tooltip, clickable legend (toggles a series), overview strip, "jump to present" button |

A `mouseup` listener is added to `document.body` once per mounted graph to end drags.

## 5. CSV export format (`downloadCSV`)

1. Header: `Timestamp (s),<all series incl. hidden>,Valve Events`.
2. Collects every stored full-resolution sample (`segment.arrays[0]`) of every series into a map keyed by timestamp.
3. Attaches each valve event to the nearest sample within 50 ms (de-duplicated by label).
4. Rows sorted by time; `Timestamp` is relative to the first sample, 3 decimals; values 3 decimals; **missing values written as `0.000`**; events as a quoted `;`-joined string.
5. File name `sensor_data_<label>_<ISO timestamp>.csv`.

The export covers the whole browser session for that graph (not just the visible window) and contains **calibrated** values (plus volts for graphs with hidden `… V` series). Example:

```
Timestamp (s),LOX Tank,LOX N2,LOX Inlet,ETH Tank,ETH N2,ETH Inlet,LOX Tank V,LOX N2 V,LOX Inlet V,ETH Tank V,ETH N2 V,ETH Inlet V,Valve Events
0.000,30.000,0.000,30.000,40.104,150.000,30.000,1.056,0.000,0.864,1.250,1.632,0.864,
0.051,30.010,0.000,29.990,40.110,150.020,30.005,1.056,0.000,0.864,1.250,1.632,0.864,"Opened LOX main"
```

(`LOX N2` is `0.000` here because the channel has no data.)

## 6. Graph instances (L500-577)

| Constant | Unit | Visible series | Hidden series (CSV only) |
|---|---|---|---|
| `PressureDatalogger` | bar | LOX Tank, LOX N2, LOX Inlet, ETH Tank, ETH N2, ETH Inlet | the six `… V` series |
| `ForceDatalogger` | kg | ETH Load Cell | ETH Load Cell V |
| `FlowDatalogger` | LPS | LOX Flow | – |
| `VoltageDatalogger` | V | the six pressure `… V` series + ETH Load Cell V | – |
| `TemperatureDatalogger` | °C | ETH Temp, LOX Temp | – |

Series names must equal the keys produced by `formatDataPoint`.

## 7. `GraphPanel({ state })` (default export, L579-647)

Creates a ref per graph, renders all five side by side (wrapping) with subtitles, passing `currentSeconds = state.data.time`, and a footer with a `<select>` choosing which graph the **Download SVG** / **Download CSV** buttons act on.

## 8. Execution flow

Module load → five stores and ten document listeners exist for the life of the page (data accumulates even if the panel were unmounted) → `index.js` pushes ~20 points/s → each graph redraws at most once per animation frame, and also whenever `App` re-renders.

## 9. Edge cases and performance

- Each series/segment pre-allocates large typed arrays (see [reactfront_src_components_series.js.md](reactfront_src_components_series.js.md)); 24 series are created in total.
- Before any data exists, `store.minTime()` is `Infinity`/`NaN`-like and the plot is empty; code relies on NaN-tolerant arithmetic.
- `formatUnit` adds SI prefixes, so 0.5 bar is labelled `500 mbar` and 1500 would be `1.5 Kbar`.
- Tooltip values are interpolated between decimated points when zoomed out.
- The y-range state is per page load; toggling a series off removes it from auto-scaling.
- `handleMouseOut` dereferences `svgRef.current` without a guard.

## 10. Relationships

Used by `index.js` (`GraphPanel`, `newData`, `newEvent`, `pinFromID`), `control-panel.js` and `sequence-panel.js` (`pinFromID`). Depends on `series.js` and `graph-utils.js`.
