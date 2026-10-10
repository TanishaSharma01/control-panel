# `reactfront/src/components/datalogger.js`

Source: [`reactfront/src/components/datalogger.js`](../../reactfront/src/components/datalogger.js) (142 lines)

## 1. Status: dead code

This file is an **early prototype** of the graphing component. It is **not imported by any other file**, so webpack never bundles it and it has no effect on the application. The working implementation lives in [`panels/graph-panel.js`](reactfront_src_components_panels_graph-panel.js.md) and [`series.js`](reactfront_src_components_series.js.md).

It could not run if it were imported:

| Line | Problem |
|---|---|
| 2 | `import binarySearch from 'graph-utils.js'` – bare module specifier (not `./graph-utils`) and a default import, whereas `graph-utils.js` only has named exports |
| 66 | `pushArray` calls `array.set(array, …)` on its own argument instead of `this.array.set(…)` |
| 119, 122 | `const forceUpdate` is reassigned |
| 122 | `useReducer` is not imported |
| 130, 133 | `w`, `h`, `v2x`, `v2y`, `currentSeconds` are undefined |

## 2. What it documents (design intent)

The header comment (L4-35) records the goals of the datalogger design, which the real implementation fulfils:

- an event-based, append-only API – `newData({time, 'LOX Tank': value, …})`, `newEvent({time, label})`;
- min/max shading and decimation for large data sets;
- multiple series, zoom and drag gestures;
- NaN values introduce gaps; data must arrive in chronological order.

The intended usage example still uses `psi` units, reflecting the project's earlier unit choice.

## 3. Contents

- `newData`, `newEvent` (L41-44) – identical to those now exported from `graph-panel.js`.
- `DecimatedMinMaxSeries` (L52-87) – a single flat `Float64Array` of `(time, value)` pairs with a binary-search `sample(startTime, endTime)` and no decimation yet.
- `zip` (L89-91).
- `Datalogger({ series })` (L93-142) – registers the two document listeners and returns a component drawing one polyline per series.

## 4. Recommendation **[Proposed]**

Delete the file, or move its header comment into `graph-panel.js`, to avoid confusing new developers. Do not import it: its top-level `Datalogger` would register additional document listeners.
