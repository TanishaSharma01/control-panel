# LOX Tank Pressure

Shared theory, acquisition details, calibration procedure and error analysis for all 4–20 mA pressure channels are in **[pressure-4-20ma-common.md](pressure-4-20ma-common.md)**. This file records what is specific to this channel.

## A. Overview

| Item | Value | Source |
|---|---|---|
| Role | Pressure in the liquid-oxygen run tank ullage. | `pins.json` `full_name`: "LOX Tank Pressure Sensor" |
| Calibration key (`abbrev`) | `lox_tank` | [`pins.json`](../../reactfront/src/pins.json) L97-101 |
| LabJack | LOX stand U3 (`SerialNumber = 2`) | [`stands.py`](../../src/stands.py) |
| Pin | `LOXSensor = ('LOX', 4)` → FIO4 / AIN4 | `stands.py` L39 |
| Graph series | `'LOX Tank'` (bar) and `'LOX Tank V'` (volts) | [`utils.js`](../../reactfront/src/utils.js) L227-228 |
| Diagram card shape | pill | `pins.json` |
| Sensor make / model | **Unknown** (not in repository) | – |
| Physical quantity | Fluid pressure, displayed in bar (gauge vs absolute not stated) | – |
| Assumed output | 4–20 mA into a 120 Ω shunt → 0.48–2.40 V **[Inferred]** | `utils.js` L53-57 comments |
| Default full scale `barMax` | **100 bar** | `utils.js` L53-57 |
| Default `minVolts` / `maxVolts` | 0.48 V / 2.40 V | `utils.js` L53-57 |
| Range, accuracy, response time, media limits | Unknown – datasheet required | – |

Operating principle: strain-gauge diaphragm transmitter with a linear 4–20 mA output; see the common document.

## B. Data acquisition

| Step | Code |
|---|---|
| Pin configured as analog at start-up | `LabJack.__init__`, `labjack.py` L131-135 |
| Read | `LabJack.get_voltage(4)` → `u3.getAIN(4)`, `labjack.py` L201-204 |
| Packaged | `get_state()["analog"][4]`, `labjack.py` L82-85 |
| Sent | `STATE.data.labjacks.LOX.analog["4"]` (volts), ≤ 20 Hz |
| Converted | `getBar(V, sensorData.lox_tank.barMax, .minVolts, .maxVolts)` in `formatDataPoint` and in `ControlCard` |
| Displayed | Pressure graph, Voltage graph, diagram card (20-sample mean, with instantaneous volts in brackets) |

Missing/invalid data handling is as described in the common document (no range checks; a USB read error stops the server).

## C. Calculation

```
P [bar] = (V − minVolts) / (maxVolts − minVolts) × barMax          (getBar, utils.js L1-5)
```

With the defaults for this channel:

```
P = (V − 0.48) / 1.92 × 100  =  52.083 × (V − 0.48)   bar
```

| Derived quantity | Value |
|---|---|
| Sensitivity | 52.083 bar/V |
| One 12-bit LSB (≈ 0.596 mV, assuming the 0–2.44 V range) **[Theory]** | ≈ 0.031 bar |
| Reading at 0 V (open loop) | -25.0 bar |
| Reading at ADC clip (≈ 2.44 V) | 102.1 bar |

### Worked example – default constants

Measured voltage `V = 1.056 V` (equivalent loop current 1.056 V / 120 Ω = 8.80 mA):

```
P = (1.056 − 0.480) / (2.400 − 0.480) × 100
  = 0.576 / 1.920 × 100
  = 0.3000 × 100
  = 30.00 bar          (= 435.1 psi, for reference; psi is not computed by the code)
```

## D. Calibration

**Status: no project calibration is recorded for this channel.** The defaults are the ideal 4/20 mA × 120 Ω voltages, i.e. an uncalibrated software conversion. An operator can enter measured constants in *Sensor Calibration → LOX Tank Pressure* (`calibration-panel.js` `renderPressureSensor`), which are stored only in that browser's `localStorage` under `sensorCalibration.lox_tank`.

### Numerical example of a two-point calibration (illustrative values, not measured data)

Suppose the zero reading with the line vented is `0.490 V`, and a reference gauge shows `80.0 bar` while the channel reads `2.020 V`.

Option 1 – enter the reference point directly: `barMax = 80.0`, `minVolts = 0.490`, `maxVolts = 2.020`.

Option 2 – keep the nameplate full scale: `barMax = 100`, `minVolts = 0.490`,

```
maxVolts = 0.490 + (2.020 − 0.490) × 100 / 80.0 = 2.402 V
```

A later raw reading of `1.300 V` then converts to:

```
calibrated (option 1):  (1.300 − 0.490) / (2.020 − 0.490) × 80.0 = 42.35 bar
calibrated (option 2):  (1.300 − 0.490) / (2.402 − 0.490) × 100 = 42.35 bar   (identical line)
uncalibrated default:   (1.300 − 0.480) / 1.920 × 100 = 42.71 bar
difference:             0.36 bar (0.36 % of 100 bar full scale)
```

Procedure, validation and uncertainty assessment: see [pressure-4-20ma-common.md §D](pressure-4-20ma-common.md#d-calibration).

## E. Error analysis

All generic sources (quantisation ≈ 0.031 bar/LSB here, noise, shunt tolerance, temperature, wiring faults, per-browser constants) are covered in the common document. No filtering or validation exists for this channel beyond the 20-sample block average on its diagram card.

### Channel-specific notes

- **This is the only sensor the server itself acts on.** `init_state_handler` (`server.py` L539-560) reads it at 1 Hz, by name (`hasattr(LOX,'LOXSensor')`), to drive the red warning lamp (pin 18): off below 0.52 V (≈ 2.1 bar with defaults), solid between 0.52 V and 1.47 V, blinking above 1.47 V (≈ 51.6 bar with defaults; the code comment says "600psi", see [known-issues.md](../known-issues.md) A7). The thresholds are raw volts and ignore browser calibration.
- History: temporarily removed and then restored on FIO4 (`stands.py` L37-39; commit `ca158d6` "fixed lox sensor").

## Code references

- `src/stands.py` L39 – pin definition
- `src/labjack.py` L131-135, L201-209, L82-85 – configuration and read
- `reactfront/src/utils.js` L1-5 (`getBar`), L53-57 (defaults), L227-228 (`formatDataPoint`)
- `reactfront/src/pins.json` L97-101 – card placement
- `reactfront/src/components/panels/control-panel.js` L174-178 – card conversion
- `reactfront/src/components/panels/graph-panel.js` L501-522, L550-565 – Pressure and Voltage graphs
- `reactfront/src/components/panels/calibration-panel.js` L121-168 – calibration form
