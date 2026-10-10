# `reactfront/src/utils.js`

Source: [`reactfront/src/utils.js`](../../reactfront/src/utils.js) (316 lines)

## 1. Purpose

The frontend's measurement layer: conversion formulas from server values to engineering units, the calibration store (defaults + `localStorage`), averaging helpers, and `formatDataPoint`, which turns one `STATE` message into a flat record for the graphs.

## 2. Imports

None. Uses the browser `localStorage` API at module load.

## 3. Conversion functions

| Function | Lines | Formula | Status |
|---|---|---|---|
| `getBar(volts, barMax, minVolts, maxVolts)` | 1-5 | `(V − Vmin)/(Vmax − Vmin) × barMax` | Used (pressure) |
| `getLoadCellKg(volts, fullRange, zeroVoltage, maxVoltage)` | 20-23 | `(V − Vzero)/(Vmax − Vzero) × fullRange` | Used (load cell) |
| `getTemperatureC(raw, fullRange, readingAt25, readingAtFullRange)` | 31-34 | `25 + (raw − R25)/(RFR − R25) × (fullRange − 25)` | Used (thermocouples) |
| `getLPS(volts, minFlow, maxFlow, minVolts=0, maxVolts=5)` | 37-44 | `max(0, (V − Vmin)/(Vmax − Vmin) × (Qmax − Qmin) + Qmin)` | **Unused** (old analog flow path) |
| `voltsToBar(volts, barMax)` | 199-206 | `barMax/(120·0.02) × (V − 120·0.004)` | **Unused**, and scaled by 2.4 V instead of the 1.92 V span |
| `barToVolts(bar, barMax)` | 208-215 | `bar/barMax × 2.4 + 0.48` | **Unused** (inverse of the above) |

None of them guards against division by zero (`maxVolts === minVolts` → `±Infinity`/`NaN`) or non-numeric input (`undefined` → `NaN`). A commented-out `getGPM` (L7-14) is the US-gallon predecessor of `getLPS`.

Derivations, variable definitions and worked examples: [sensors/pressure-4-20ma-common.md](../sensors/pressure-4-20ma-common.md), [sensors/eth-load-cell.md](../sensors/eth-load-cell.md), [sensors/max6675-common.md](../sensors/max6675-common.md), [sensors/lox-cryo-flow.md](../sensors/lox-cryo-flow.md).

## 4. Calibration store

### `defaultSensorCalibration` (L47-105)

| Key | Fields |
|---|---|
| `eth_tank`, `lox_tank` | `barMax: 100, minVolts: 0.48, maxVolts: 2.4` |
| `eth_n2`, `lox_n2` | `barMax: 250, minVolts: 0.48, maxVolts: 2.4` |
| `lox_inlet`, `eth_inlet` | `barMax: 150, minVolts: 0.48, maxVolts: 2.4` |
| `lox_cryo` | `type: 'flow'` only |
| `eth_temp`, `lox_temp` | `type: 'temperature', fullRange: 100, readingAt25: 25.0, readingAtFullRange: 100.0` |
| `eth_load_cell` | `type: 'force', fullRange: 100, zeroVoltage: 0.0, maxVoltage: 5.0` |

Keys equal the `abbrev` values in `pins.json`. Pressure entries have no `type`; `ControlCard` treats "no type" as pressure.

### Loading (L110-116)

```js
const _stored = JSON.parse(localStorage.getItem('sensorCalibration')) || {};
export let sensorData = Object.fromEntries(
    Object.keys(defaultSensorCalibration).map(key => [key, { ...defaultSensorCalibration[key], ...(_stored[key] || {}) }])
);
```

Per-sensor shallow merge: stored fields override defaults; new default fields appear automatically; stored sensors that no longer exist are dropped. Old stored fields (e.g. `zero`/`span` from the previous mA-based scheme) are carried along harmlessly. Corrupt JSON in `localStorage` would throw at import and blank the app.

### `updateSensorCalibration(sensorKey, newCalibration)` (L119-122)
Merges into `sensorData[sensorKey]` and writes the whole `sensorData` to `localStorage`.

### `resetSensorCalibration(sensorKey = null)` (L125-132)
Resets one sensor or all to defaults and saves. (`sensorData` is an exported `let`; ES-module live bindings make reassignment visible to importers.)

## 5. Averaging helpers

| Name | Lines | Notes |
|---|---|---|
| `SENSOR_BATCH_SIZE = 20` | 135 | Used by `control-panel.js` |
| `calculateBatchAverage(values)` | 138-141 | Arithmetic mean or `null` |
| `processSensorBatch(batch, value, size)` | 144-166 | Accumulate; on reaching `size` return the mean and an empty batch |
| `createSensorBatchUpdater(setBatches, setAverages)` | 169-196 | Hook-style wrapper around the above |

The last three are **not used**; `control-panel.js` re-implements the same logic inline.

## 6. `formatDataPoint(dict)` (L218-255)

Input: `STATE.data` (with `time` already in seconds). Output keys:

| Key | Source |
|---|---|
| `time` | `dict.time` |
| `'LOX Tank'`, `'LOX Tank V'` | `getBar(LOX.analog?.["4"], lox_tank…)`, raw volts |
| `'LOX N2'`, `'LOX N2 V'` | `LOX.analog?.["5"]` – currently always `NaN`/`undefined` |
| `'LOX Inlet'`, `'LOX Inlet V'` | `LOX.analog?.["2"]` |
| `'ETH Tank'`, `'ETH Tank V'` | `ETH.analog["4"]` |
| `'ETH N2'`, `'ETH N2 V'` | `ETH.analog["5"]` |
| `'ETH Inlet'`, `'ETH Inlet V'` | `ETH.analog["6"]` |
| `'LOX Flow'` | `LOX.cryo_flow_lps ?? NaN` |
| `'ETH Temp'`, `'LOX Temp'` | `getTemperatureC(<stand>.temperature, …)` or `NaN` when `null` |
| `'ETH Load Cell'`, `'ETH Load Cell V'` | `getLoadCellKg(ETH.analog["0"], …)` / volts, `NaN` if absent |

Pin numbers are hard-coded strings (TODO at L217). LOX lookups use optional chaining; **ETH lookups do not** (L233-238), so a missing `ETH.analog` throws.

`emptyDataPoint` (L257-277): the same keys, all `NaN`. Pushing it makes every series start a new line segment.

## 7. Dead/commented code

L279-316: the previous fixed calibration table (`zero`/`span` in mA; `eth_tank` 3.99/16.02 mA, `lox_tank` 3.99/16.04 mA; the N2 sensors marked "uncalibrated"; `lox_cryo` 0.80–29.00 GPM "PT420 calibration from sheet"). Historically interesting: it shows that the two tank sensors once had measured mA constants. The comment block is not syntactically valid JS (missing comma) but is inert.

## 8. Execution flow and relationships

- Module load: reads `localStorage`, builds `sensorData`.
- `index.js` calls `formatDataPoint` per `STATE`.
- `control-panel.js` imports `getBar`, `getLoadCellKg`, `getTemperatureC`, `sensorData`, `SENSOR_BATCH_SIZE`.
- `calibration-panel.js` imports `sensorData`, `updateSensorCalibration`, `resetSensorCalibration`, `defaultSensorCalibration`.

## 9. Example

```js
formatDataPoint({ time: 1785127770.207, labjacks: {
  LOX: { analog: {"4": 1.056, "2": 0.864}, temperature: 0.0, cryo_flow_lps: 1.0668 },
  ETH: { analog: {"5": 1.632, "4": 1.25, "6": 0.864, "0": 0.4957}, temperature: 25.75 } } })
// → { time: 1785127770.207,
//     'LOX Tank': 30.0, 'LOX Tank V': 1.056, 'LOX N2': NaN, 'LOX N2 V': undefined,
//     'LOX Inlet': 30.0, 'LOX Inlet V': 0.864,
//     'ETH Tank': 40.1, 'ETH Tank V': 1.25, 'ETH N2': 150.0, 'ETH N2 V': 1.632,
//     'ETH Inlet': 30.0, 'ETH Inlet V': 0.864,
//     'LOX Flow': 1.0668, 'ETH Temp': 25.75, 'LOX Temp': 0,
//     'ETH Load Cell': 9.914, 'ETH Load Cell V': 0.4957 }   (default calibration)
```
