# `reactfront/src/components/series.js`

Source: [`reactfront/src/components/series.js`](../../reactfront/src/components/series.js) (314 lines)

## 1. Purpose

The in-browser time-series store behind every graph. It keeps append-only `(time, value)` data in packed `Float64Array`s, maintains a pyramid of 2×-decimated mean/min/max summaries as data arrives, and can return about one point per pixel for any time window – so zooming out over hours of 20 Hz data stays fast.

## 2. Imports

`binarySearch`, `undefOnBadRef` from `graph-utils`.

## 3. Classes

### `DoubleArray(capacity)` (L4-37)
Fixed-capacity `Float64Array` with `size` (doubles used) and `chunks` (records pushed). `push(...values)` copies values at `size`; a `RangeError` when full is caught and logged (data silently dropped). `shrink()` reallocates to exact size. `filterMapChunks(filter, map)` iterates record indices.

### `FullResolutionMinMaxArray(points)` (L85-93)
Raw level: 2 doubles per record `[time, value]`; `mean(i)`, `min(i)`, `max(i)` all return the value.

### `MinMaxArray(points)` (L39-83)
Decimated level: 4 doubles per record `[time, mean, min, max]`. `updateDecimation(source)` is called after every push to the level below and recomputes only the **last** record from the last one or two source records:

- source count odd → the new record starts as a copy of the latest source record (an "in-flight" summary);
- source count even → finalised: `time` = time of the later source record, `mean` = average of the two means, `min`/`max` = min/max of the two.

Using the later timestamp (not the midpoint) keeps the newest data flush with the right edge of the graph.

### `DecimatedMinMaxSeries` (L101-208)
One contiguous segment. `capacity = 524288` raw points; `n_bound = 12` → levels 0…12 (level *n* summarises 2ⁿ raw points, up to 4096×).

| Method | Behaviour |
|---|---|
| `push(time, value)` | Push to level 0, then `updateDecimation` up the pyramid (13 cheap updates per sample) |
| `sample(startTime, endTime, k = 200)` | Binary-search the raw index window; choose level `n = clamp(1 + floor(log2(x/k + 0.01)), 0, 12)` so about `k/2…k` points are returned; binary-search that level; return an array of `[time, mean, min, max]` with `.min`, `.max`, `.decimated`, and string builders `getPoints` (polyline of means) and `getMinMaxPoints` (closed polygon: mins forward, maxes backward) |
| `samplePreview(k)` | Naive every-Nth sampling of raw data for the overview strip |
| `shrink()` | Trim all levels |

### `Series` (L212-275, exported)
A list of segments plus an `enabled` flag. `push(time, value)`: if either is `NaN`, close the current segment (shrink it) and start a new one – unless the current one is still empty; otherwise push to the last segment. This is how gaps (disconnections, missing sensors) appear as breaks in the line. `sample` returns per-segment results with helpers `totalLength`, `splitPointIndex`, `fromPointIndex` (used by the tooltip) and overall `min`/`max`. `minTime()`, `maxTime()`.

### `SeriesCollection(series)` (default export, L281-314)
Maps series name → `Series`. `sample(timeWindow, k)` / `samplePreview(k)` return per-key results (or `false` for disabled series) with combined `min`/`max`; `minTime()`; `toggleSeries(key)`.

## 4. Execution flow

Created inside each `Datalogger` factory call in `graph-panel.js`; `push` is invoked from the `datalogger-new-data` listener (~20 Hz per series); `sample`/`samplePreview` on every graph render.

## 5. Worked example of decimation

Raw pushes `(t, v)`: (0.00, 10), (0.05, 14), (0.10, 12), (0.15, 20).

| After push | Level 1 records `[time, mean, min, max]` | Level 2 |
|---|---|---|
| 1 | `[0.00, 10, 10, 10]` (in flight) | `[0.00, 10, 10, 10]` |
| 2 | `[0.05, 12, 10, 14]` | `[0.05, 12, 10, 14]` |
| 3 | `[0.05, 12, 10, 14]`, `[0.10, 12, 12, 12]` | `[0.10, 12, 10, 14]` |
| 4 | `[0.05, 12, 10, 14]`, `[0.15, 16, 12, 20]` | `[0.15, 14, 10, 20]` |

When a graph shows a decimated level, the line follows the means and the shaded band spans min…max, so short spikes remain visible when zoomed out.

## 6. Capacity, memory and limits **[Inferred from the code]**

- 524 288 points per segment ≈ 7.3 h at 20 Hz. Beyond that, pushes fail (console error) until a NaN starts a new segment.
- Each new segment allocates 2 × 524 288 doubles (raw) + 4 × Σ(524 288 >> n, n = 1…12) ≈ 3.1 million doubles ≈ 24 MiB of (initially untouched, zero-filled) typed arrays; `shrink()` releases the excess when the segment is closed. The source comment estimates ~12 MiB.
- With 24 series across the five graphs, that is on the order of 0.5 GiB of reserved address space at start-up, and every disconnect (`emptyDataPoint`) opens a fresh segment for every series that had data. Browsers generally commit zero-filled memory lazily, but long sessions with many reconnects on low-memory devices deserve attention.
- Times must be non-decreasing for the binary searches to be valid.
- Mean of means at higher levels is exact only for complete pairs; the newest record of each level is provisional.

## 7. Relationships

Used only by `graph-panel.js`. Depends on `graph-utils.js`.
