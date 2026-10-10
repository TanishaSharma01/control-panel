# Sensor Reference

Central inventory of every measurement input in the system. Each sensor has its own document; the two shared-chain documents hold the theory common to several channels.

## Inventory

| # | Sensor | Stand / LabJack | Interface & pins (`src/stands.py`) | Server output | Browser conversion | Display unit | Model known? | Project-calibrated? | Doc |
|---|---|---|---|---|---|---|---|---|---|
| 1 | ETH tank pressure | ETH | AIN, FIO4 | volts | `getBar`, 100 bar | bar | No | No (nominal defaults) | [eth-tank-pressure.md](eth-tank-pressure.md) |
| 2 | ETH N₂ pressurant pressure | ETH | AIN, FIO5 | volts | `getBar`, 250 bar | bar | No | No | [eth-n2-pressure.md](eth-n2-pressure.md) |
| 3 | ETH motor inlet pressure | ETH | AIN, FIO6 | volts | `getBar`, 150 bar | bar | No | No | [eth-inlet-pressure.md](eth-inlet-pressure.md) |
| 4 | LOX tank pressure | LOX | AIN, FIO4 | volts | `getBar`, 100 bar | bar | No | No | [lox-tank-pressure.md](lox-tank-pressure.md) |
| 5 | LOX motor inlet pressure | LOX | AIN, FIO2 | volts | `getBar`, 150 bar | bar | No | No | [lox-inlet-pressure.md](lox-inlet-pressure.md) |
| 6 | LOX N₂ pressurant pressure | LOX | **disabled** (was AIN FIO5) | – | `getBar`, 250 bar → `NaN` | bar | No | No | [lox-n2-pressure.md](lox-n2-pressure.md) |
| 7 | LOX cryogenic flow | LOX | UART: TX FIO6, RX FIO7, 114 286 baud | L/s (parsed text) | pass-through + 10-sample mean | LPS | "PT420" only | No (conversion is in external firmware) | [lox-cryo-flow.md](lox-cryo-flow.md) |
| 8 | ETH thermocouple | ETH | SPI: SCK FIO3, CS FIO2, SO EIO3 (dummy MOSI FIO1) | °C (MAX6675) | `getTemperatureC` | °C | MAX6675 (probe unknown) | No (identity defaults) | [eth-thermocouple.md](eth-thermocouple.md) |
| 9 | LOX thermocouple | LOX | SPI: SCK FIO1, CS FIO0, SO FIO3 (dummy MOSI FIO5) | °C (MAX6675) | `getTemperatureC` | °C | MAX6675 (probe unknown) | No (identity defaults) | [lox-thermocouple.md](lox-thermocouple.md) |
| 10 | ETH load cell | ETH | AIN, FIO0 | volts | `getLoadCellKg` | kg | No | No (placeholder defaults) | [eth-load-cell.md](eth-load-cell.md) |

Shared-chain documents: [pressure-4-20ma-common.md](pressure-4-20ma-common.md) · [max6675-common.md](max6675-common.md)

### Not sensors, but also "inputs"

| Input | How it is read | Notes |
|---|---|---|
| Valve state (`digital`) | `u3.getDIOState(pin)` | Read-back of the U3 output line, inverted for vent valves. **Not** a valve-position sensor |
| Light-stand state (`light_stand`) | `u3.getDIOState(pin)` | Read-back of lamp outputs |
| UPS status | `apcaccess status` subprocess, 1 Hz | `ONLINE` / `ONBATT` substring match (`server.py` L181-204) |

## Pin map per LabJack (from `src/stands.py`)

DIO index: 0–7 = FIO0–7, 8–15 = EIO0–7, 16–19 = CIO0–3.

| DIO | LOX U3 (`SerialNumber = 2`) | ETH U3 (`SerialNumber = 1`) |
|---|---|---|
| 0 (FIO0) | Thermocouple CS | Load cell (AIN) |
| 1 (FIO1) | Thermocouple SCK | Thermocouple dummy MOSI |
| 2 (FIO2) | Inlet pressure (AIN) | Thermocouple CS |
| 3 (FIO3) | Thermocouple SO | Thermocouple SCK |
| 4 (FIO4) | Tank pressure (AIN) | Tank pressure (AIN) |
| 5 (FIO5) | Thermocouple dummy MOSI (N₂ sensor unplugged) | N₂ pressure (AIN) |
| 6 (FIO6) | UART TX → flow MCU | Inlet pressure (AIN) |
| 7 (FIO7) | UART RX ← flow MCU | – |
| 8 (EIO0) | Purge valve | Pressurisation valve |
| 9 (EIO1) | Pressurisation valve | Vent valve (inverted relay) |
| 10 (EIO2) | Vent valve (inverted relay) | Igniter |
| 11 (EIO3) | – | Thermocouple SO |
| 12 (EIO4) | Light: yellow | Light: yellow |
| 13 (EIO5) | Light: green | Light: green |
| 14 (EIO6) | Main valve | Purge valve |
| 15 (EIO7) | Chill valve | Main valve |
| 16 (CIO0) | Fill valve | Fill valve |
| 17 (CIO1) | Drain valve | Drain valve |
| 18 (CIO2) | Light: red | Light: red |
| 19 (CIO3) | Buzzer | Buzzer |

## Comparison

| Aspect | Pressure (×6) | Flow | Thermocouple (×2) | Load cell |
|---|---|---|---|---|
| Physical principle | Strain-gauge diaphragm, 4–20 mA loop | Pulse-frequency meter (probably turbine) + MCU | Seebeck effect, K-type, cold-junction compensated | Strain-gauge bridge + amplifier |
| Where conversion to units happens | Browser | External microcontroller | MAX6675 IC (then optional correction in browser) | Browser |
| Raw resolution | 12-bit ADC ≈ 0.6 mV | set by firmware | 0.25 °C | 12-bit ADC ≈ 0.6 mV |
| True update rate | ≤ 20 Hz | firmware line rate (polled ≤ 20 Hz) | ≈ 1 Hz | ≤ 20 Hz |
| Server-side filtering | none | none (latest line) | median of ≤ 3 SPI reads | none |
| Browser filtering | card: 20-sample block mean | graph: 10-sample rolling mean; card: 20-sample block mean | card: 20-sample block mean | card: 20-sample block mean |
| Calibration model | 2-point linear (`barMax`, `minVolts`, `maxVolts`) | none in repo | 2-point linear anchored at 25 °C | 2-point linear (`fullRange`, `zeroVoltage`, `maxVoltage`) |
| Calibration storage | browser `localStorage` | – | browser `localStorage` | browser `localStorage` |
| Fault detection | none | malformed-line rejection | open-thermocouple bit | none |
| Used by server logic | LOX/ETH tank → red lamp thresholds | no | no | no |

## Calibration status summary

**No sensor in this system has a documented project calibration.** The code provides *mechanisms* (editable two-point linear maps) and *nominal defaults*; whether real constants have been entered is a property of each operator's browser and is not recorded in the repository or in the server logs. Distinguish:

| Term | Applies here to |
|---|---|
| Factory calibration | U3 analog inputs (constants applied by `getAIN`); MAX6675 trim; presumably each transmitter / load cell / flow meter (certificates not available) |
| Software conversion | `getBar`, `getLoadCellKg`, `(v>>3)*0.25`, CSV parsing |
| Empirical (project) calibration | Possible through the Sensor Calibration panel; **none recorded** |

## Distinguishing raw and derived series

Every analog channel has two graph series: `'<Name>'` (engineering units) and `'<Name> V'` (volts). The volts series is what to trust when diagnosing; it is plotted on the *Raw Voltages* graph and exported in the Pressure and Load Cell CSV downloads as hidden columns.
