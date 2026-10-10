# LOX Nitrogen Pressurant Pressure (currently disabled)

> **Status: DISABLED.** `stands.py` L35-36 has `N2Sensor = ('LOX', 5)` commented out with the note *"N2Sensor is still unplugged - FIO5 is free if it gets wired back up later"*, and `LOX.Sensors` (L48) does not include pin 5. The server therefore sends no `analog["5"]` for LOX. In addition, FIO5 is now passed as the dummy **MOSI** pin of the LOX thermocouple SPI bus (`stands.py` L62), so the U3 drives FIO5 as a digital output during every thermocouple read – the sensor **cannot simply be plugged back in** without choosing another MOSI pin.
>
> The frontend still carries the channel (`pins.json`, `formatDataPoint`, calibration panel), which produces the artefacts described under *Current behaviour* below.

Shared theory, acquisition details, calibration procedure and error analysis for all 4–20 mA pressure channels are in **[pressure-4-20ma-common.md](pressure-4-20ma-common.md)**. This file records what is specific to this channel.

## A. Overview

| Item | Value | Source |
|---|---|---|
| Role | Pressure of the nitrogen supply that pressurises the LOX tank. **Not currently acquired.** | `pins.json` `full_name`: "LOX Nitrogen Pressurant Sensor" |
| Calibration key (`abbrev`) | `lox_n2` | [`pins.json`](../../reactfront/src/pins.json) L91-95 |
| LabJack | LOX stand U3 (`SerialNumber = 2`) | [`stands.py`](../../src/stands.py) |
| Pin | `N2Sensor = ('LOX', 5)` → FIO5 / AIN5 | `stands.py` L35-36 (commented out) |
| Graph series | `'LOX N2'` (bar) and `'LOX N2 V'` (volts) | [`utils.js`](../../reactfront/src/utils.js) L229-230 |
| Diagram card shape | semicircle | `pins.json` |
| Sensor make / model | **Unknown** (not in repository) | – |
| Physical quantity | Fluid pressure, displayed in bar (gauge vs absolute not stated) | – |
| Assumed output | 4–20 mA into a 120 Ω shunt → 0.48–2.40 V **[Inferred]** | `utils.js` L63-67 comments |
| Default full scale `barMax` | **250 bar** | `utils.js` L63-67 |
| Default `minVolts` / `maxVolts` | 0.48 V / 2.40 V | `utils.js` L63-67 |
| Range, accuracy, response time, media limits | Unknown – datasheet required | – |

Operating principle: strain-gauge diaphragm transmitter with a linear 4–20 mA output; see the common document.

## B. Data acquisition

| Step | Code |
|---|---|
| Pin configured as analog at start-up | `LabJack.__init__`, `labjack.py` L131-135 |
| Read | `LabJack.get_voltage(5)` → `u3.getAIN(5)`, `labjack.py` L201-204 |
| Packaged | `get_state()["analog"][5]`, `labjack.py` L82-85 |
| Sent | `STATE.data.labjacks.LOX.analog["5"]` (volts), ≤ 20 Hz |
| Converted | `getBar(V, sensorData.lox_n2.barMax, .minVolts, .maxVolts)` in `formatDataPoint` and in `ControlCard` |
| Displayed | Pressure graph, Voltage graph, diagram card (20-sample mean, with instantaneous volts in brackets) |

Missing/invalid data handling is as described in the common document (no range checks; a USB read error stops the server).

## C. Calculation

```
P [bar] = (V − minVolts) / (maxVolts − minVolts) × barMax          (getBar, utils.js L1-5)
```

With the defaults for this channel:

```
P = (V − 0.48) / 1.92 × 250  =  130.208 × (V − 0.48)   bar
```

| Derived quantity | Value |
|---|---|
| Sensitivity | 130.208 bar/V |
| One 12-bit LSB (≈ 0.596 mV, assuming the 0–2.44 V range) **[Theory]** | ≈ 0.078 bar |
| Reading at 0 V (open loop) | -62.5 bar |
| Reading at ADC clip (≈ 2.44 V) | 255.2 bar |

### Worked example – default constants

Measured voltage `V = 1.632 V` (equivalent loop current 1.632 V / 120 Ω = 13.60 mA):

```
P = (1.632 − 0.480) / (2.400 − 0.480) × 250
  = 1.152 / 1.920 × 250
  = 0.6000 × 250
  = 150.00 bar          (= 2175.6 psi, for reference; psi is not computed by the code)
```

## D. Calibration

**Status: no project calibration is recorded for this channel.** The defaults are the ideal 4/20 mA × 120 Ω voltages, i.e. an uncalibrated software conversion. An operator can enter measured constants in *Sensor Calibration → LOX Nitrogen Pressurant Pressure* (`calibration-panel.js` `renderPressureSensor`), which are stored only in that browser's `localStorage` under `sensorCalibration.lox_n2`.

### Numerical example of a two-point calibration (illustrative values, not measured data)

Suppose the zero reading with the line vented is `0.485 V`, and a reference gauge shows `200.0 bar` while the channel reads `2.021 V`.

Option 1 – enter the reference point directly: `barMax = 200.0`, `minVolts = 0.485`, `maxVolts = 2.021`.

Option 2 – keep the nameplate full scale: `barMax = 250`, `minVolts = 0.485`,

```
maxVolts = 0.485 + (2.021 − 0.485) × 250 / 200.0 = 2.405 V
```

A later raw reading of `1.100 V` then converts to:

```
calibrated (option 1):  (1.100 − 0.485) / (2.021 − 0.485) × 200.0 = 80.08 bar
calibrated (option 2):  (1.100 − 0.485) / (2.405 − 0.485) × 250 = 80.08 bar   (identical line)
uncalibrated default:   (1.100 − 0.480) / 1.920 × 250 = 80.73 bar
difference:             0.65 bar (0.26 % of 250 bar full scale)
```

Procedure, validation and uncertainty assessment: see [pressure-4-20ma-common.md §D](pressure-4-20ma-common.md#d-calibration).

## E. Error analysis

All generic sources (quantisation ≈ 0.078 bar/LSB here, noise, shunt tolerance, temperature, wiring faults, per-browser constants) are covered in the common document. No filtering or validation exists for this channel beyond the 20-sample block average on its diagram card.

### Current behaviour while disabled **[Existing]**

- `formatDataPoint` (`utils.js` L229-230): `dict.labjacks.LOX.analog?.["5"]` is `undefined` → `getBar(undefined, …)` = `NaN` → the `'LOX N2'` and `'LOX N2 V'` graph series never receive a point.
- `ControlCard` for `lox_n2`: `currentValue` is `NaN`, so the card initially reads **`NaN Bar`**. Its batch key is `LOX_5`, which is *also* the key of the cryo-flow card (`pins.json` L104 gives `lox_cryo` `labjack_pin: 5`). Once the flow card has stored its first 20-sample average under `LOX_5`, the N2 card displays **the cryo-flow average with the unit "Bar"** ([known-issues.md](../known-issues.md) A4).
- In `--dev` mode the same happens, because `LabJackFake` only fabricates voltages for pins listed in `Sensors`.

### To re-enable **[Proposed]**

1. Re-wire the transmitter to a free analog-capable pin and uncomment/adjust `N2Sensor` in `stands.py`, adding it to `LOX.Sensors`.
2. If FIO5 is used, move the thermocouple `mosi` to another unused pin.
3. Update `pins.json` (`lox_n2.labjack_pin`, and give `lox_cryo` a non-colliding pin value) and the `analog?.["5"]` index in `formatDataPoint`.

## Code references

- `src/stands.py` L35-36 (commented out) – pin definition
- `src/labjack.py` L131-135, L201-209, L82-85 – configuration and read
- `reactfront/src/utils.js` L1-5 (`getBar`), L63-67 (defaults), L229-230 (`formatDataPoint`)
- `reactfront/src/pins.json` L91-95 – card placement
- `reactfront/src/components/panels/control-panel.js` L174-178 – card conversion
- `reactfront/src/components/panels/graph-panel.js` L501-522, L550-565 – Pressure and Voltage graphs
- `reactfront/src/components/panels/calibration-panel.js` L121-168 – calibration form
