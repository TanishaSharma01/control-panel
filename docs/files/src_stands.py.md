# `src/stands.py`

Source: [`src/stands.py`](../../src/stands.py) (95 lines)

## 1. Purpose

The backend's hardware configuration: which LabJack pin each valve, sensor, lamp and serial bus uses on each stand. It contains only class-level constants – no logic. It is the server-side source of truth; the frontend duplicates the numbers by hand (comment L6).

## 2. Imports

None.

## 3. Definitions

### `Stand` (L2-4)
String constants `LOX = 'LOX'`, `ETH = 'ETH'`. Used only as a type annotation in `commands.py`.

### `LightStand(Stand)` (L8-15)
Instance attributes `green = (stand, 13)`, `yellow = (stand, 12)`, `red = (stand, 18)`, `buzzer = (stand, 19)` and `LightStand = [13, 12, 18, 19]` (the list `labjack.py` iterates). Both stands instantiate it with `'LOX'` (L33, L80); only the pin numbers (`[1]`) are ever used.

### Convention
Every device is a tuple `(stand_name, dio_number)`, so sequences can write `Open(LOX.Main)` and `commands.Open` unpacks the tuple. DIO numbers: 0–7 FIO, 8–15 EIO, 16–19 CIO.

### `LOX` (L17-62)

| Attribute | Value | Meaning |
|---|---|---|
| `name` | `"LOX"` | Used for logging and relay-inversion lookup |
| `SerialNumber` | `2` | Passed to `u3.U3(serial=…)`; also a phase offset in the fake signals |
| `Main` / `Fill` / `Drain` / `Pressure` / `Vent` / `Purge` / `Chill` | 14 / 16 / 17 / 9 / 10 / 8 / 15 | Valve relay pins |
| `Valves` | `[14, 16, 17, 9, 10, 8, 15]` | Pins reported under `digital` |
| `LightStand` | `LightStand('LOX')` | Lamp pins |
| `N2Sensor` | *commented out* (was 5) | "still unplugged" |
| `LOXSensor` | `('LOX', 4)` | Tank pressure, FIO4 |
| `InletPressureSensor` | `('LOX', 2)` | Moved from FIO6 to FIO2 to free the UART pins |
| `Sensors` | `[4, 2]` | Pins configured as analog and reported under `analog` |
| `CryoFlowUART` | `{'rx': 7, 'tx': 6, 'baud': 114286}` | Hardware UART to the flow-meter microcontroller |
| `Thermocouple` | `{'sck': 1, 'cs': 0, 'so': 3, 'mosi': 5}` | MAX6675 SPI pins (`mosi` is a required dummy) |

### `ETH` (L64-93)

| Attribute | Value | Meaning |
|---|---|---|
| `SerialNumber` | `1` | |
| `name` | `"ETH"` | |
| `Main` / `Fill` / `Drain` / `Pressure` / `Vent` / `Purge` / `Igniter` | 15 / 16 / 17 / 8 / 9 / 14 / 10 | Valve relay pins (the igniter is handled as a valve) |
| `Valves` | `[15, 16, 17, 8, 9, 14, 10]` | |
| `LightStand` | `LightStand('LOX')` | Same pin numbers as LOX |
| `N2Sensor` / `ETHSensor` / `InletPressureSensor` / `LoadCell` | 5 / 4 / 6 / 0 | Analog inputs |
| `Sensors` | `[5, 4, 6, 0]` | |
| `Thermocouple` | `{'sck': 3, 'cs': 2, 'so': 11, 'mosi': 1}` | |

ETH has no `CryoFlowUART`, so `get_state()` omits `cryo_flow_lps` for it.

## 4. Execution flow

Evaluated once at import. Imported by `server.py`, `labjack.py`, `commands.py`. `LOX` and `ETH` are also injected into the namespace used to evaluate sequence files.

## 5. How other code consumes it

| Consumer | Attributes used |
|---|---|
| `LabJack.__init__` / `LabJackFake.__init__` | `Valves`, `Sensors`, `LightStand.LightStand`, `SerialNumber`, `name`, `CryoFlowUART`, `Thermocouple` (the last two via `hasattr`) |
| `LabJackBase._is_inverted_relay` | `ETH.Vent[1]`, `LOX.Vent[1]` |
| `server.init_state_handler` | `LOX.LightStand.{green,yellow,red,buzzer}`, `LOX.LOXSensor` |
| Sequence files | `LOX.<Valve>`, `ETH.<Valve>` |

Feature detection is by attribute presence: deleting/commenting `Thermocouple` or `CryoFlowUART` disables that driver cleanly.

## 6. Edge cases and constraints

- `Sensors` entries must be FIO pins (0–7): the analog mask is only applied to `FIOAnalog`.
- `CryoFlowUART['rx']` must equal `tx + 1` (asserted in `labjack.py`).
- A pin must not appear in more than one role; nothing checks this.
- Any pin used in a sequence but missing from `Valves` would still be driven, but never reported.

## 7. Keeping the frontend in sync

When changing a pin here, also update: `reactfront/src/pins.json`, the hard-coded pin strings in `reactfront/src/utils.js` `formatDataPoint`, `VALVE_ATTR_BY_PIN` in `sequence-panel.js`, the igniter pin in `control-panel.js` L385, and `sequences/abort.py` if a valve was added. Current mismatches: [known-issues.md](../known-issues.md) A4, A5.

## 8. Example

```python
>>> from stands import LOX, ETH
>>> LOX.Main
('LOX', 14)
>>> ETH.Sensors
[5, 4, 6, 0]
>>> hasattr(ETH, 'CryoFlowUART')
False
```
