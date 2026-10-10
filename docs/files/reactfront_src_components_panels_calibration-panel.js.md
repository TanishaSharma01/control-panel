# `reactfront/src/components/panels/calibration-panel.js`

Source: [`reactfront/src/components/panels/calibration-panel.js`](../../reactfront/src/components/panels/calibration-panel.js) (351 lines)

## 1. Purpose

The "Sensor Calibration" view. Lets the operator edit the constants used by the browser-side conversion formulas for each sensor, save them to this browser's `localStorage`, or reset them; and, for the cryogenic flow meter, send the test-mode command to its microcontroller.

## 2. Imports

React `useState`; `Panel`; `sensorData`, `updateSensorCalibration`, `resetSensorCalibration`, `defaultSensorCalibration` from `utils`; Material-UI `Button`.

## 3. `CalibrationPanel({ state, emit })`

### State
- `calibration` – an editable shallow copy of `sensorData` (form values; may temporarily hold `''`).
- `selectedSensor` – which form is shown (default `'eth_tank'`).

### Handlers

| Handler | Lines | Behaviour |
|---|---|---|
| `handleUpdate(sensorKey, field, value)` | 10-35 | Empty string is stored as `''` (so a field can be cleared while typing); otherwise `parseFloat`; non-numeric input is ignored |
| `handleSave(sensorKey)` | 37-85 | Builds `cleanedSensor`: `type`/`serialNumber` copied as-is; every other field must parse as a number, otherwise an `alert` is shown and that field is skipped. Then checks the required fields for the sensor kind – temperature: `fullRange, readingAt25, readingAtFullRange`; load cell: `fullRange, zeroVoltage, maxVoltage`; otherwise: `barMax, minVolts, maxVolts` – and aborts with an alert if any is missing. On success: `updateSensorCalibration`, alert, **`window.location.reload()`** |
| `handleReset(sensorKey)` | 87-93 | Confirm → `resetSensorCalibration(key)` → reload |
| `handleResetAll()` | 95-101 | Confirm → `resetSensorCalibration()` → reload |

The reload guarantees that every module using `sensorData` starts from the saved values – and also clears all graph history and the WebSocket session in that tab.

### Sensor list (L103-114)
`eth_tank`, `lox_tank`, `eth_n2`, `lox_n2`, `eth_inlet`, `lox_inlet`, `lox_cryo`, `eth_temp`, `lox_temp`, `eth_load_cell` with display names. Rendered as selector buttons.

### Forms

| Renderer | Lines | Fields | Notes |
|---|---|---|---|
| `renderPressureSensor` | 121-168 | Bar Max, Min Volts, Max Volts | Shows the defaults underneath |
| `renderFlowSensor` | 170-200 | none | Explains that L/s is computed by the microcontroller; a **Test Mode: ON/OFF** button sends `emit('CRYOCOMMAND', { stand: 'LOX', command: 't' })`. The label reflects `state.data.cryo_test_mode.LOX`, the server's best guess |
| `renderTemperatureSensor` | 202-255 | Full Range (°C), Reading at 25 °C, Reading at Full Range | Includes the in-app calibration instructions |
| `renderLoadCellSensor` | 257-313 | Full Range (kg), Zero Voltage (V), Max Voltage (V) | Formula hint says "kg per mV"; the maths is per volt |

## 4. Data flow

```
form field → calibration (React state) → Save → updateSensorCalibration()
           → sensorData (module variable) + localStorage["sensorCalibration"] → page reload
           → utils.js rebuilds sensorData from localStorage → formatDataPoint / ControlCard use it
```

Nothing is sent to the server except `CRYOCOMMAND`.

## 5. Stored format example

```json
{"eth_tank":{"barMax":100,"minVolts":0.492,"maxVolts":2.012}, "lox_tank":{"barMax":100,"minVolts":0.48,"maxVolts":2.4},
 "…":"…", "lox_cryo":{"type":"flow"},
 "eth_temp":{"type":"temperature","fullRange":100,"readingAt25":26.5,"readingAtFullRange":98},
 "eth_load_cell":{"type":"force","fullRange":50,"zeroVoltage":0.497,"maxVoltage":2}}
```

## 6. Edge cases

- No sanity checks: `minVolts == maxVolts` (division by zero), negative ranges or swapped values are accepted.
- Calibration is per browser profile and origin (`http://192.168.0.5:3000` and `http://localhost:3000` have separate stores); it is not included in server logs or CSV exports.
- Not gated on arming: anyone with the page open can change how values are displayed on their own device.
- Saving the `lox_cryo` key is not reachable (no Save button for the flow card).
- Changes do not affect the server's red-lamp thresholds.

## 7. Relationships

Rendered by `index.js` when `activePanel === 'calibration'`. Writes through `utils.js`. Server counterpart for the button: `CRYOCOMMAND` in `server.py` → `LabJack.send_cryo_command`.

Calibration procedures: [sensors/pressure-4-20ma-common.md](../sensors/pressure-4-20ma-common.md#d-calibration), [sensors/max6675-common.md](../sensors/max6675-common.md#d-calibration), [sensors/eth-load-cell.md](../sensors/eth-load-cell.md#d-calibration).
