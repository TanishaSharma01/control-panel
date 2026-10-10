# `reactfront/src/components/panels/control-panel.js`

Source: [`reactfront/src/components/panels/control-panel.js`](../../reactfront/src/components/panels/control-panel.js) (420 lines)

## 1. Purpose

The main operating view: a P&ID background with a toggle switch for every valve and a numeric card for every sensor, the igniter confirmation dialog, and the "Sequence Execution" list shown beside it.

## 2. Imports (L1-7)

Material-UI `Switch`; React hooks `useState`, `useEffect`, `useCallback`; `getBar`, `getLoadCellKg`, `getTemperatureC`, `sensorData`, `SENSOR_BATCH_SIZE` from `utils`; `Panel` from `components/index`; `pins.json`; `pinFromID` from `graph-panel`.

## 3. Components and functions

### `describeCommand(command)` (L11-22)
Server command dict → text: `SLEEP` → `"Sleep 5.0s"` (from `ms`); `OPEN`/`CLOSE` → `"Open LOX Main"` using the button `name` from `pins.json`, or `"<stand> pin <n>"` if unknown.

### `SequenceExecutionList({ state, fullSequence, fullSequenceName })` (L31-70, exported)

Shows every command of the loaded sequence and highlights the one executing. The server only sends *remaining* commands, so the full list is the snapshot held by `App`.

```
completedCount = max(0, fullSequence.length − remaining.length − (inFlight ? 1 : 0))
currentIndex   = inFlight ? completedCount : −1
```

The row at `currentIndex` gets a green background and a ▶ marker. If an abort replaces the sequence, the name changes to `abort`, `App` re-snapshots, and the list switches to the abort commands.

### `normalisePosition(num)` (L72-74)
`num * 26` – grid units to pixels.

### `controlWidgetStyle({x, y, width, height, enabled, shape})` (L76-109)
Absolute-position style; `cursor: help` when disabled; `semicircle` (shifted up 30 px, rounded bottom) and `pill` (radius = width/2) variants.

### `ControlSwitch({ state, emit, ...props })` (L111-142)

- `value` = `state.data.labjacks[test_stand].digital[labjack_pin]` (server truth; `false` before data arrives).
- `setValue` ignores its argument and re-reads the current server value: open → `emit('CLOSE', {name, pin})`, closed → `emit('OPEN', …)`.
- `<Switch disabled={!props.enabled}>`; a tooltip explains that arming and manual control are required. An optional `label` is drawn beneath (used for "Igniter").

### `ControlCard({ state, emit, sensorBatches, sensorAverages, updateSensorHistory, ...props })` (L144-244)

1. `sensorKey = "<test_stand>_<labjack_pin>"`; `sensor = sensorData[props.sensorName]`.
2. Instantaneous value by `sensor.type`:

| `type` | Source | Conversion | Unit |
|---|---|---|---|
| `'temperature'` | `labjacks[stand].temperature` | `getTemperatureC` | °C |
| `'force'` | `analog[pin]` | `getLoadCellKg` | kg |
| `'flow'` | `labjacks[stand].cryo_flow_lps` | none | LPS |
| *(none)* | `analog[pin]` | `getBar` | Bar |

3. `useEffect` keyed on the state timestamp: when the value is a valid number, `updateSensorHistory(sensorKey, value)`.
4. Displayed value = stored 20-sample average if one exists, else the instantaneous value.
5. Background: flow → light green above `0.1 × 0.06309` L/s; everything else → translucent red when the value exceeds 2 (intended as "pressurised > 2 bar", but also applied to temperature and load).
6. Renders `value.toFixed(2) unit` and, for analog sensors, `(x.xxxV)`.

### `ControlPanel({ state, emit })` (default export, L246-421)

State: `sensorBatches`, `sensorAverages`, `showIgniterConfirm`.

`updateSensorHistory(sensorKey, newValue)` (L253-288): append to the batch for that key; when it reaches `SENSOR_BATCH_SIZE` (20), store the mean in `sensorAverages[key]` and reset the batch. This is a **block average**: the number on a card changes about once per second and is the mean of the previous 20 samples.

Render:

- "ETH" / "LOX" labels.
- `pins.buttons.map` → `ControlSwitch` with `enabled = arming_switch && manual_switch`. For `abbrev === 'igniter'` the `emit` prop is wrapped: an `OPEN` request opens the confirmation modal instead of being sent; `CLOSE` goes straight through.
- `pins.sensors.map` → `ControlCard` with `sensorName = abbrev`.
- A legend (Open / Closed switch samples).
- Igniter modal (L360-418): "Are you sure you want to start the igniter?" – **Yes** sends `emit('OPEN', { name: 'ETH', pin: 10 })` (pin hard-coded) and closes; **No** closes.

Nothing renders until `state.data` exists.

## 4. Execution flow

Re-rendered on every `STATE` (~20 Hz) while the Control Panel view is active. Unmounted when another view is selected, which discards `sensorBatches`/`sensorAverages` (cards restart from instantaneous values).

## 5. Inputs / outputs

Props: `state` (App state), `emit`. Output: `OPEN`/`CLOSE` commands. Reads `pins.json` and `sensorData`.

## 6. Relationships

Rendered by `index.js`. Uses `Panel` (components/index), `pinFromID` (graph-panel), utils conversions. Server counterpart: `handle_command` `OPEN`/`CLOSE`.

## 7. Edge cases and known behaviours

- Switches are not optimistic; with a dead connection they do nothing.
- The server silently ignores `OPEN`/`CLOSE` unless armed + manual; the UI disables the switches in that case.
- Batch-key collision `LOX_5` (N₂ vs flow) – [known-issues.md](../known-issues.md) A4.
- Missing `key` props for sensor cards (most entries lack `name`) – C7.
- Red highlight on temperature/load cards – A6.
- If a value stops arriving (e.g. thermocouple `null`), the card keeps showing its last average indefinitely.

## 8. Example

`eth_tank` card with `analog["4"] = 1.25 V` and default calibration: instantaneous `getBar(1.25, 100, 0.48, 2.4) = 40.10`; after 20 samples averaging 40.07 the card reads `40.07 Bar (1.250V)` on a red background.
