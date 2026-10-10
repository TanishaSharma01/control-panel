# `reactfront/src/components/graph-utils.js`

Source: [`reactfront/src/components/graph-utils.js`](../../reactfront/src/components/graph-utils.js) (132 lines)

## 1. Purpose

Pure helper functions for the graphs: axis tick generation, unit formatting, binary search, interval arithmetic, interpolation, and a safe-dereference helper. No imports, no state.

## 2. Functions

| Function | Lines | Exported | Behaviour |
|---|---|---|---|
| `roundToNiceDecimalIncrement(num)` | 2-18 | no | Rounds a positive step to the nearest of 1, 2, 5, 10 × 10ⁿ, choosing the closest in log space. `0.37 → 0.5`, `14 → 10`, `3 → 2` |
| `roundToSigFigs(x, sigFigs)` | 20-22 | no | `parseFloat(x.toPrecision(sigFigs))` |
| `formatUnit(val, unit, sigFigs = 3)` | 25-35 | yes | Value with an SI prefix: T, G, M, K, (none), m, μ, n, falling back to p. `0` → `"0 unit"`. `formatUnit(0.5, 'bar')` → `"500 mbar"`; `formatUnit(-3, 's')` → `"-3 s"`; `formatUnit(1500, 'bar')` → `"1.5 Kbar"` |
| `generateAxisTicks(min, max, suggested, unit)` | 39-52 | yes | Step = nice increment of `(max − min)/suggested`; ticks at multiples of the step within the range, each `{tick, label}`; values within 1e-10 of zero are snapped to 0 |
| `binarySearch(f, target, length)` | 56-72 | yes | Over indices `0…length−1` of a monotonic function `f`, returns the highest index with `f(i) <= target` (exact match returns immediately); `−1` if even `f(0) > target` |
| `intervalUnion(a, b)` | 74-78 | yes | Smallest interval covering both; tolerates a null operand |
| `shrinkInterval([a,b],[sa,sb])` | 81-83 | yes (unused outside) | `shrinkInterval([0,10],[0.1,0.8]) → [1, 8]` |
| `expandInterval([a,b], fraction)` | 86-88 | yes | Pads both ends: `expandInterval([0,10], 0.2) → [-2, 12]` |
| `interpolateInterval(a, b, fraction)` | 90-94 | yes | Moves `a` towards `b` by `fraction` (used for smooth y-axis rescaling at 10 % per frame) |
| `lerp(x1, y1, x2, y2, x)` | 96-98 | yes | Linear interpolation (tooltip value between two samples) |
| `clamp(x, min, max)` | 100-102 | yes | |
| `boundIntervalKeepingLength([a,b],[min,max])` | 104-110 | yes | Slides an interval back inside bounds without changing its length where possible (panning) |
| `undefOnBadRef(callback)` | 121-133 | yes | Runs the callback; returns `undefined` if it throws a `TypeError` (e.g. reading a property of `null`), re-throws anything else. A substitute for optional chaining |

## 3. Callers

`graph-panel.js` (all exported functions except `shrinkInterval`), `series.js` (`binarySearch`, `undefOnBadRef`), `index.js` (`undefOnBadRef`).

## 4. Edge cases

- `generateAxisTicks` with `min === max` gives an increment of 0, the tick positions become `NaN`, and no ticks are returned. With `NaN`/`Infinity` bounds (no data yet) it returns no ticks.
- `formatUnit` applies SI prefixes to every unit, including °C (`"25 °C"`, but `0.5` → `"500 m°C"`) and kg (`1500` → `"1.5 Kkg"`).
- `lerp` divides by `x2 − x1`; identical timestamps yield `NaN`/`Infinity`.
- `undefOnBadRef` also hides genuine `TypeError` bugs inside the callback.
- `binarySearch` assumes `f` is non-decreasing.

## 5. Example

```js
generateAxisTicks(-3, 0, 4, 's')
// increment = nice(0.75) = 1 → [{tick:-3,label:'-3 s'}, {tick:-2,…}, {tick:-1,…}, {tick:0,label:'0 s'}]
binarySearch(i => [1, 2, 4, 8][i], 5, 4)   // → 2
```
