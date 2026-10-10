# ETH Load Cell (Thrust / Weight)

## A. Overview

| Item | Value | Source |
|---|---|---|
| Role | Force measurement, displayed as kilograms. The graph is labelled "Load Cell (kg)" and a code comment calls it "load cell thrust" (`graph-panel.js` L524); the diagram card is "Load Cell (Weight)" | `pins.json` L145-149 |
| Calibration key | `eth_load_cell` (`type: 'force'`) | `utils.js` L99-104 |
| Pin | `LoadCell = ('ETH', 0)` → **FIO0 / AIN0** on the ETH U3; "DC voltage from load cell amplifier" | `stands.py` L85 |
| Load cell model, capacity, excitation | **Unknown** | – |
| Amplifier model, gain, output range | **Unknown** (only "load cell amplifier" is stated) | – |
| Graph series | `'ETH Load Cell'` (kg) and `'ETH Load Cell V'` (volts) | `utils.js` L250-253; `graph-panel.js` L524-536 |

It is wired to the ETH LabJack for I/O-availability reasons; it measures the engine/stand, not anything ethanol-specific **[Inferred]**.

### Operating principle **[Theory]**

A load cell is an elastic metal element carrying four strain gauges in a Wheatstone bridge. Force strains the element, changing gauge resistance by ΔR/R = GF × ε, and the bridge outputs a voltage proportional to both force and excitation voltage:

```
V_bridge = V_excitation × S [mV/V] × (F / F_rated)
```

Typical sensitivity `S` is 1–3 mV/V, so the full-scale bridge signal is only a few millivolts. An instrumentation amplifier multiplies it by a gain `G` and (often) adds an offset, yielding the DC voltage that the LabJack measures:

```
V_out = V_offset + G × V_bridge
```

Because `G`, `S`, `V_excitation` and `V_offset` are all unknown here, the software does not use them; it relies on an empirical two-point fit (comment at `utils.js` L16-19: "No dependency on the amplifier's internal gain").

### Electrical interface

- Single-ended analog voltage on FIO0, included in the ETH analog mask (bit 0 of `0b01110001`, `labjack.py` L131-135).
- **Input-range caution [Theory/Inferred]:** a low-voltage U3 input measures ≈ 0–2.44 V. The default `maxVoltage` of 5.0 V is outside that range. Either the amplifier output must be kept below ≈ 2.4 V at maximum load, or the ETH device is a U3-HV (where FIO0 is a ±10 V input – but then FIO1–3 could not serve as the thermocouple SPI pins). See [known-issues.md](../known-issues.md) A12.

## B. Data acquisition

Same path as the pressure channels: `get_state()` → `get_voltage(0)` → `u3.getAIN(0)` → `STATE.data.labjacks.ETH.analog["0"]` (volts), one un-averaged 12-bit conversion per state tick (≤ 20 Hz). USB read errors terminate the server (`labjack.py` L205-209).

In the browser: if `analog["0"]` is `undefined` the graph series get `NaN` (`utils.js` L250-253) and the card shows nothing. `pins.json` stores this pin as the *string* `"0"`, which works because it is only used as an object key.

**Bandwidth note [Inferred]:** ≤ 20 samples/s with no anti-alias filter is too slow to resolve a thrust transient (ignition spike, shutdown); it gives an average-thrust picture only. Combustion oscillations will alias.

## C. Calculation

### Implemented **[Existing]** – `getLoadCellKg`, `reactfront/src/utils.js` L20-23

```js
const kg = (volts - zeroVoltage) / (maxVoltage - zeroVoltage) * fullRange;
```

```
m [kg] = (V − V_zero) / (V_max − V_zero) × m_ref
```

| Symbol | Code name | Unit | Meaning |
|---|---|---|---|
| V | `volts` | V | FIO0 voltage |
| V_zero | `zeroVoltage` | V | FIO0 voltage with no load (default 0.0) |
| V_max | `maxVoltage` | V | FIO0 voltage with the reference weight applied (default 5.0) |
| m_ref | `fullRange` | kg | The reference weight (default 100) |

Sensitivity = `m_ref / (V_max − V_zero)` kg/V (20 kg/V with defaults). The calibration panel's hint text calls this "kg per mV", which is a labelling slip – it is per volt.

Called from `formatDataPoint` (`utils.js` L250-252) and `ControlCard` (`control-panel.js` L159-166). Card: 20-sample block mean, `x.xx kg (x.xxxV)`.

Worked example with defaults, `V = 0.4957 V` (a value seen in a dev-mode log):

```
m = (0.4957 − 0.0) / (5.0 − 0.0) × 100 = 9.91 kg
```

### Not implemented (theory)

- Force in newtons: `F = m × 9.80665` (e.g. 9.91 kg → 97.2 N).
- Tare/zero button; the zero can only be changed by editing `zeroVoltage`.
- Compensation for stand geometry (lever ratio, plumbing stiffness), temperature or off-axis loads.

## D. Calibration

| Kind | Status |
|---|---|
| Factory | Load cell rated output (mV/V) – unknown, no certificate in repo |
| Software conversion | Two-point linear map above |
| Project calibration performed | **None recorded.** Defaults are explicit placeholders ("needs a real calibration weight to set accurately", `utils.js` L101) |
| Mechanism | *Sensor Calibration → ETH Load Cell* (`calibration-panel.js` `renderLoadCellSensor`, L257-313): fields Full Range (kg), Zero Voltage (V), Max Voltage (V); saved to this browser's `localStorage` as `sensorCalibration.eth_load_cell` |

Errors addressed: amplifier gain and offset, load-cell sensitivity tolerance, fixed preload/tare of the mounting. Not addressed: non-linearity, hysteresis, creep, temperature drift, changing preload.

### Procedure **[Proposed; follows the panel's instructions]**

Equipment: certified weights or a reference load cell/force gauge covering the expected thrust; the Voltage graph.

1. Assemble the stand in firing configuration (plumbing attached – feed lines add stiffness and preload). Warm up electronics ≥ 15 min.
2. No load: record mean FIO0 voltage over ≥ 20 s → `zeroVoltage`.
3. Apply a known load along the thrust axis, ideally 80–100 % of expected peak thrust → record mean voltage → `maxVoltage`; enter the load as `fullRange`.
4. Save, then verify at 3–5 intermediate loads, increasing then decreasing (hysteresis), and re-check zero.
5. Re-zero (`zeroVoltage`) before every test: preload changes with assembly and temperature.

### Numerical example (illustrative values)

Zero-load voltage 0.497 V; with a 50.0 kg reference the voltage is 2.000 V. Enter `fullRange = 50`, `zeroVoltage = 0.497`, `maxVoltage = 2.000`.

A later reading of 1.200 V:

```
m = (1.200 − 0.497) / (2.000 − 0.497) × 50
  = 0.703 / 1.503 × 50
  = 23.39 kg          (≈ 229.4 N, not computed by the code)
```

Sensitivity = 50 / 1.503 = 33.27 kg/V; one 12-bit LSB (≈ 0.596 mV on the low-voltage range) ≈ 0.020 kg.

Accuracy assessment: for each check load, `error = m_displayed − m_ref`, `% = error / fullRange × 100`; repeatability from repeated loadings; uncertainty combines reference-weight class, ADC quantisation/noise and observed non-linearity/hysteresis.

## E. Error analysis

| Source | Effect | Present handling |
|---|---|---|
| Amplifier/ADC noise, quantisation | Jitter | 20-sample card mean; none on graph |
| ADC range clipping (output > 2.44 V on a low-voltage input) | Flat-topped thrust curve | None |
| Zero drift (temperature, creep, preload) | Offset | Manual re-entry of `zeroVoltage` |
| Mechanical: off-axis load, friction in the thrust mount, plumbing stiffness | Scale error | Calibrate in situ |
| Vibration / combustion dynamics above 10 Hz | Aliasing | None |
| EMI from igniter/solenoids on mV-level bridge wiring | Spikes | None |
| Negative load / reversed polarity | Negative kg shown (no clamp) | None |
| Card colour | Card turns red above 2 kg (pressure rule) | – |

### Proposed improvements (not implemented)

Tare button; display in newtons; faster dedicated sampling (LabJack stream mode) for thrust curves; server-side calibration storage; clip detection.

## Code references

`src/stands.py` L85-86 · `src/labjack.py` L131-135, L201-209 · `reactfront/src/utils.js` L16-23, L99-104, L250-253, L275-276 · `reactfront/src/pins.json` L144-150 · `reactfront/src/components/panels/control-panel.js` L159-166 · `reactfront/src/components/panels/graph-panel.js` L524-536, L563 · `reactfront/src/components/panels/calibration-panel.js` L257-313
