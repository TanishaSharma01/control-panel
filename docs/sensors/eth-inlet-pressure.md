# ETH Motor Inlet Pressure

Shared theory, acquisition details, calibration procedure and error analysis for all 4–20 mA pressure channels are in **[pressure-4-20ma-common.md](pressure-4-20ma-common.md)**. This file records what is specific to this channel.

## A. Overview

| Item | Value | Source |
|---|---|---|
| Role | Ethanol pressure at the engine (motor) inlet, downstream of the main fuel valve – i.e. the injector feed pressure during a firing. | `pins.json` `full_name`: "ETH Motor Inlet Pressure" |
| Calibration key (`abbrev`) | `eth_inlet` | [`pins.json`](../../reactfront/src/pins.json) L121-125 |
| LabJack | ETH stand U3 (`SerialNumber = 1`) | [`stands.py`](../../src/stands.py) |
| Pin | `InletPressureSensor = ('ETH', 6)` → FIO6 / AIN6 | `stands.py` L84 |
| Graph series | `'ETH Inlet'` (bar) and `'ETH Inlet V'` (volts) | [`utils.js`](../../reactfront/src/utils.js) L237-238 |
| Diagram card shape | rectangle (default) | `pins.json` |
| Sensor make / model | **Unknown** (not in repository) | – |
| Physical quantity | Fluid pressure, displayed in bar (gauge vs absolute not stated) | – |
| Assumed output | 4–20 mA into a 120 Ω shunt → 0.48–2.40 V **[Inferred]** | `utils.js` L73-77 comments |
| Default full scale `barMax` | **150 bar** | `utils.js` L73-77 |
| Default `minVolts` / `maxVolts` | 0.48 V / 2.40 V | `utils.js` L73-77 |
| Range, accuracy, response time, media limits | Unknown – datasheet required | – |

Operating principle: strain-gauge diaphragm transmitter with a linear 4–20 mA output; see the common document.

## B. Data acquisition

| Step | Code |
|---|---|
| Pin configured as analog at start-up | `LabJack.__init__`, `labjack.py` L131-135 |
| Read | `LabJack.get_voltage(6)` → `u3.getAIN(6)`, `labjack.py` L201-204 |
| Packaged | `get_state()["analog"][6]`, `labjack.py` L82-85 |
| Sent | `STATE.data.labjacks.ETH.analog["6"]` (volts), ≤ 20 Hz |
| Converted | `getBar(V, sensorData.eth_inlet.barMax, .minVolts, .maxVolts)` in `formatDataPoint` and in `ControlCard` |
| Displayed | Pressure graph, Voltage graph, diagram card (20-sample mean, with instantaneous volts in brackets) |

Missing/invalid data handling is as described in the common document (no range checks; a USB read error stops the server).

## C. Calculation

```
P [bar] = (V − minVolts) / (maxVolts − minVolts) × barMax          (getBar, utils.js L1-5)
```

With the defaults for this channel:

```
P = (V − 0.48) / 1.92 × 150  =  78.125 × (V − 0.48)   bar
```

| Derived quantity | Value |
|---|---|
| Sensitivity | 78.125 bar/V |
| One 12-bit LSB (≈ 0.596 mV, assuming the 0–2.44 V range) **[Theory]** | ≈ 0.047 bar |
| Reading at 0 V (open loop) | -37.5 bar |
| Reading at ADC clip (≈ 2.44 V) | 153.1 bar |

### Worked example – default constants

Measured voltage `V = 0.864 V` (equivalent loop current 0.864 V / 120 Ω = 7.20 mA):

```
P = (0.864 − 0.480) / (2.400 − 0.480) × 150
  = 0.384 / 1.920 × 150
  = 0.2000 × 150
  = 30.00 bar          (= 435.1 psi, for reference; psi is not computed by the code)
```

## D. Calibration

**Status: no project calibration is recorded for this channel.** The defaults are the ideal 4/20 mA × 120 Ω voltages, i.e. an uncalibrated software conversion. An operator can enter measured constants in *Sensor Calibration → ETH Motor Inlet Pressure* (`calibration-panel.js` `renderPressureSensor`), which are stored only in that browser's `localStorage` under `sensorCalibration.eth_inlet`.

### Numerical example of a two-point calibration (illustrative values, not measured data)

Suppose the zero reading with the line vented is `0.478 V`, and a reference gauge shows `100.0 bar` while the channel reads `1.770 V`.

Option 1 – enter the reference point directly: `barMax = 100.0`, `minVolts = 0.478`, `maxVolts = 1.770`.

Option 2 – keep the nameplate full scale: `barMax = 150`, `minVolts = 0.478`,

```
maxVolts = 0.478 + (1.770 − 0.478) × 150 / 100.0 = 2.416 V
```

A later raw reading of `1.000 V` then converts to:

```
calibrated (option 1):  (1.000 − 0.478) / (1.770 − 0.478) × 100.0 = 40.40 bar
calibrated (option 2):  (1.000 − 0.478) / (2.416 − 0.478) × 150 = 40.40 bar   (identical line)
uncalibrated default:   (1.000 − 0.480) / 1.920 × 150 = 40.63 bar
difference:             0.22 bar (0.15 % of 150 bar full scale)
```

Procedure, validation and uncertainty assessment: see [pressure-4-20ma-common.md §D](pressure-4-20ma-common.md#d-calibration).

## E. Error analysis

All generic sources (quantisation ≈ 0.047 bar/LSB here, noise, shunt tolerance, temperature, wiring faults, per-browser constants) are covered in the common document. No filtering or validation exists for this channel beyond the 20-sample block average on its diagram card.

### Channel-specific notes

- Added in commit `d099eee` ("lox and eth motor inlet pressure sensors added").

## Code references

- `src/stands.py` L84 – pin definition
- `src/labjack.py` L131-135, L201-209, L82-85 – configuration and read
- `reactfront/src/utils.js` L1-5 (`getBar`), L73-77 (defaults), L237-238 (`formatDataPoint`)
- `reactfront/src/pins.json` L121-125 – card placement
- `reactfront/src/components/panels/control-panel.js` L174-178 – card conversion
- `reactfront/src/components/panels/graph-panel.js` L501-522, L550-565 – Pressure and Voltage graphs
- `reactfront/src/components/panels/calibration-panel.js` L121-168 – calibration form
