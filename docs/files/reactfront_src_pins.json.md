# `reactfront/src/pins.json`

Source: [`reactfront/src/pins.json`](../../reactfront/src/pins.json) (151 lines)

## 1. Purpose

Declarative description of every widget on the P&ID control view: 14 valve switches (`buttons`) and 10 sensor cards (`sensors`), each with its LabJack pin and its position on the diagram. It is the frontend's (hand-maintained) copy of the pin map in `src/stands.py` – the file's own `"notice"` field warns about the duplication.

## 2. Structure

```json
{
  "notice": "The labjack pin values are duplicated in the Python code, beware!",
  "buttons": [ { "pin": {...}, "position": {...} }, ... ],
  "sensors": [ { "pin": {...}, "position": {...} }, ... ]
}
```

### `pin` fields

| Field | Used by | Meaning |
|---|---|---|
| `test_stand` | switches, cards, `pinFromID` | `"ETH"` or `"LOX"` – key into `state.data.labjacks` |
| `labjack_pin` | switches, cards, `pinFromID` | DIO number – key into `digital` / `analog` |
| `full_name` | (not displayed) | Descriptive name |
| `abbrev` | buttons: event labels (`"Opened LOX main"`), igniter detection; sensors: **key into `sensorData`** (calibration) and the card's `sensorName` | |
| `name` | buttons: React key and execution-list label; sensors: React key (mostly absent) | |
| `sensorNumber` | nothing | Legacy index |
| `type` | nothing (the type comes from `sensorData[abbrev].type`) | Present on the two temperature entries |

### `position` fields

`x`, `y`, `width`, `height` in grid units; `control-panel.js` multiplies by 26 px (`normalisePosition`) and positions the widget absolutely inside the 790 × 569 px `.control-panel` box whose background is `labjack.svg`. Optional `shape`: `"semicircle"` (N₂ bottles) or `"pill"` (tanks) changes the border radius.

## 3. Buttons (valves)

| Stand | Name | Pin | abbrev | (x, y) |
|---|---|---|---|---|
| ETH | ETH Main | 15 | main | (10.1, 15.5) |
| ETH | ETH Fill | 16 | fill | (6.1, 12.4) |
| ETH | ETH Drain | 17 | drain | (6.1, 16) |
| ETH | ETH Pressurisation | 8 | pres | (6.4, 5.3) |
| ETH | ETH Vent | 9 | vent | (10.2, 3.2) |
| ETH | ETH Purge | 14 | purge | (2.5, 7.2) |
| ETH | ETH Igniter | 10 | igniter | (1, 18) |
| LOX | LOX Main | 14 | main | (16.7, 15.4) |
| LOX | LOX Fill | 16 | fill | (20.6, 12.3) |
| LOX | LOX Drain | 17 | drain | (20.6, 15.7) |
| LOX | LOX Pressurisation | 9 | pres | (19.7, 5.3) |
| LOX | LOX Vent | 10 | vent | (16.8, 3.2) |
| LOX | LOX Purge | 8 | purge | (25.8, 6.8) |
| LOX | LOX Chill | 15 | chill | (24, 14.1) |

All match `stands.py`.

## 4. Sensors

| abbrev | Stand | `labjack_pin` | Is the pin actually used to read the value? | Matches `stands.py`? |
|---|---|---|---|---|
| `lox_n2` | LOX | 5 | yes (`analog[5]`) | ❌ sensor disabled server-side → `NaN` |
| `lox_tank` | LOX | 4 | yes | ✅ |
| `lox_cryo` | LOX | 5 | no (reads `cryo_flow_lps`) | n/a – but collides with `lox_n2` as an averaging key |
| `lox_temp` | LOX | 0 | no (reads `temperature`) | n/a |
| `lox_inlet` | LOX | 2 | yes | ✅ |
| `eth_inlet` | ETH | 6 | yes | ✅ |
| `eth_n2` | ETH | 5 | yes | ✅ |
| `eth_tank` | ETH | 4 | yes | ✅ |
| `eth_temp` | ETH | 2 | no | n/a |
| `eth_load_cell` | ETH | `"0"` (string) | yes | ✅ |

For flow and temperature entries the pin is only part of the card's batch key `${test_stand}_${labjack_pin}`. Two cards sharing a key share an average: see [known-issues.md](../known-issues.md) A4.

## 5. Consumers

| File | Use |
|---|---|
| `control-panel.js` | `pins.buttons.map` → `ControlSwitch`; `pins.sensors.map` → `ControlCard` |
| `graph-panel.js` `pinFromID(labjack_pin, test_stand)` | Looks up a **button** by pin (+ stand) for event labels and command descriptions |

`formatDataPoint` and the graphs do **not** read this file.

## 6. How to change things

- **Move a widget:** edit `position`.
- **Add a valve:** (1) add it to the stand class and `Valves` list in `src/stands.py`; (2) add a `buttons` entry with a unique `name`; (3) add its safe state to `src/sequences/abort.py`; (4) add it to `VALVE_ATTR_BY_PIN` and the valve dropdown in `sequence-panel.js`.
- **Add an analog sensor:** (1) `stands.py` attribute + `Sensors`; (2) a `sensors` entry whose `abbrev` is a new key; (3) that key in `defaultSensorCalibration` (`utils.js`); (4) series in `formatDataPoint`/`emptyDataPoint`; (5) a series in a `Datalogger` in `graph-panel.js`; (6) an entry in the calibration panel's `sensors` list.

## 7. Edge cases

- `pinFromID` compares with `===`, so a string pin in a `buttons` entry would never match the numeric pin the server sends.
- If `sensorData[abbrev]` is undefined the card renders empty.
- Because the same pin number is used on both stands (e.g. 14, 16, 17), always pass the stand to `pinFromID`; the fallback returns the first match by pin only.
