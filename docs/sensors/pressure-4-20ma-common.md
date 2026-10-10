# Pressure Channels – Common Measurement Chain (4–20 mA → shunt → LabJack AIN)

All six pressure channels use the identical chain and the identical code; only the pin, range and calibration key differ. This document holds the shared theory, acquisition path, formulas, calibration procedure and error analysis. Each channel has its own file with its specific numbers and worked examples:

[ETH tank](eth-tank-pressure.md) · [ETH N2](eth-n2-pressure.md) · [ETH inlet](eth-inlet-pressure.md) · [LOX tank](lox-tank-pressure.md) · [LOX inlet](lox-inlet-pressure.md) · [LOX N2 (disabled)](lox-n2-pressure.md)

## A. Overview

### What is known and unknown

| Item | Status |
|---|---|
| Manufacturer / model | **Unknown – not recorded anywhere in the repository.** |
| Output type | 4–20 mA current loop **[Inferred]** from code comments `// 4mA × 120Ω`, `// 20mA × 120Ω` (`utils.js` L50-76) and from the old `zero: 3.99 // mA at 0 bar`, `span: 16.02` calibration left in comments (`utils.js` L281-304) |
| Shunt resistor | 120 Ω **[Inferred]** from the same comments and from `voltsToBar` (`utils.js` L200). Tolerance and actual measured values unknown |
| Full-scale range | The `barMax` defaults (100 / 150 / 250 bar). These are *software defaults*; whether they match each transmitter's nameplate is unverified |
| Accuracy, response time, media/temperature limits | Unknown (need datasheets) |

### Operating principle **[Theory]**

A pressure transmitter contains a diaphragm with a strain-gauge (piezoresistive or thin-film) Wheatstone bridge. Pressure deflects the diaphragm, unbalancing the bridge; internal electronics convert this to a loop current that is linear in pressure:

```
I = 4 mA + 16 mA × (P / P_fullscale)
```

4 mA ("live zero") represents zero pressure, so a current of 0 mA unambiguously means a broken wire or dead transmitter. The current is turned into a voltage by a precision shunt resistor `R` placed between the signal line and ground, and the DAQ measures the voltage across it:

```
V = I × R        → with R = 120 Ω:  V(4 mA) = 0.48 V,  V(20 mA) = 2.40 V
```

**Why 120 Ω [Inferred]:** the LabJack U3's low-voltage single-ended analog range is about 0–2.44 V, so 120 Ω places 20 mA (2.40 V) just inside full scale, using almost the whole ADC range.

### Electrical interface

- One LabJack flexible-IO pin per channel, configured as an analog input in `LabJack.__init__` (`labjack.py` L131-135): a bitmask with bit *n* set for every pin in `standConfig.Sensors` is passed to `configIO(FIOAnalog=mask, EIOAnalog=0)`. For LOX (`Sensors=[4,2]`) the mask is `0b00010100` = 20; for ETH (`[5,4,6,0]`) it is `0b01110001` = 113.
- Loop supply voltage, transmitter wiring (2-wire/3-wire) and shunt location are not documented in the repository.

## B. Data acquisition

1. `LabJackBase.get_state()` (`labjack.py` L82-85) loops over `self.analog_inputs` and calls `get_voltage(pin)`.
2. `LabJack.get_voltage()` (L201-209) takes `_device_lock` and returns `self.device.getAIN(pin_number)`.
3. **[Theory, LabJackPython source]** `u3.getAIN(posChannel, negChannel=31, longSettle=False, quickSample=False)` performs one single-ended conversion (negative channel 31 = ground) and converts the 12-bit result to volts with the device's stored calibration constants (`binaryToCalibratedAnalogVoltage`: `bits × slope + offset`). The returned float is in **volts**.
4. `sync_state()` (`server.py` L152-162) does this once per loop iteration (nominally 20 Hz) for both stands and broadcasts the result.

| Property | Value |
|---|---|
| Resolution **[Theory]** | 12 bit over ≈ 2.44 V → ≈ 0.6 mV/LSB |
| Sampling | One un-averaged conversion per channel per state tick (≤ 20 Hz) |
| Synchronisation | Channels are read sequentially; one timestamp for all |
| Data type on the wire | JSON number (volts), key = pin number as a string |
| Error handling | Any exception in `getAIN` → traceback + `sys.exit()` (server stops) |
| Missing data in the browser | `analog?.["4"]` → `undefined` → `getBar` → `NaN` → gap in the graph |
| Out-of-range | Not checked anywhere. V < `minVolts` gives negative bar; V > `maxVolts` gives > `barMax` |

In `--dev` mode `LabJackFake.get_voltage` (L404-408) returns a synthetic signal: `0.4·sin(t + pin + serial) + 2·sin(t/10 + pin + 7) + 1 ± 0.1` plus a rare +1 V spike (0.5 % of calls). It ranges roughly from −1.5 V to +4.5 V, i.e. it deliberately exceeds the physical range, so dev-mode pressures can be negative or above full scale.

## C. Conversion formula

### Implemented **[Existing]** – `getBar`, [`reactfront/src/utils.js`](../../reactfront/src/utils.js) L1-5

```js
export function getBar(volts, barMax, minVolts, maxVolts) {
    const bar = (volts - minVolts) / (maxVolts - minVolts) * barMax;
    return bar;
}
```

```
P [bar] = (V − V_min) / (V_max − V_min) × P_max
```

| Symbol | Code name | Unit | Meaning |
|---|---|---|---|
| V | `volts` | V | Voltage from `getAIN` |
| V_min | `minVolts` | V | Voltage at 0 bar (default 0.48 = 4 mA × 120 Ω) |
| V_max | `maxVolts` | V | Voltage at full scale (default 2.40 = 20 mA × 120 Ω) |
| P_max | `barMax` | bar | Pressure at `maxVolts` |

It is a straight line through (V_min, 0) and (V_max, P_max): slope `P_max / (V_max − V_min)` bar/V, intercept `−V_min × slope`. The result is **gauge or absolute pressure according to the transmitter type**, which the repository does not state.

Called from:

- `formatDataPoint` (`utils.js` L227-238) → graph series `'<STAND> Tank' / 'N2' / 'Inlet'` plus the raw `'… V'` series.
- `ControlCard` (`control-panel.js` L175-177) → the number on the diagram.

No rounding (except `toFixed(2)` for card text, `toFixed(3)` in CSV export), no clamping, no filtering in the conversion itself. The card applies a 20-sample block mean (`SENSOR_BATCH_SIZE`, `control-panel.js` L253-288); graphs plot every sample.

### Present but unused – `voltsToBar` / `barToVolts` (`utils.js` L198-215)

`voltsToBar` computes `barMax / (R·I₂) × (V − R·I₁)` with R = 120 Ω, I₁ = 4 mA, I₂ = 20 mA, i.e. it divides by 2.4 V rather than the 1.92 V span. It is labelled "dodgy old sensors" and is not called. Do not use it as a reference.

### Theoretical, **not implemented**

- Current from voltage: `I = V / R`.
- psi: `P_psi = P_bar × 14.5038` (a commented-out line at `utils.js` L204 shows this was once applied).
- Open-loop detection: `V < ~0.4 V` (I < 3.3 mA) ⇒ fault.

## D. Calibration

### What exists **[Existing]**

- **Software conversion with editable constants.** `defaultSensorCalibration` (`utils.js` L47-77) holds nominal values; the *Sensor Calibration* panel ([`calibration-panel.js`](../../reactfront/src/components/panels/calibration-panel.js) `renderPressureSensor`, L121-168) lets an operator edit `barMax`, `minVolts`, `maxVolts` per channel. `handleSave` validates that every field is numeric, calls `updateSensorCalibration` (`utils.js` L119-122), which merges the values into the module-level `sensorData` and writes the whole object to `localStorage["sensorCalibration"]`, then reloads the page.
- On load, stored values are deep-merged over the defaults per sensor (`utils.js` L110-116).
- "Reset to Default" / "Reset All" restore the nominal constants.

### What does *not* exist

- The repository contains **no record that any channel has been calibrated against a reference**. The defaults are the *ideal* 4 mA / 20 mA voltages across an ideal 120 Ω resistor. Treat default readings as **uncalibrated software conversion**.
- No server-side storage, no audit trail, no calibration date, no per-device sharing: constants exist only in the browser that saved them.
- No temperature compensation, no drift tracking, no multi-point or non-linear correction.
- Factory calibration: the transmitter is presumably factory-calibrated to its 4–20 mA output (unverified, no certificates in repo); the U3 analog inputs carry LabJack factory calibration constants that `getAIN` applies **[Theory]**.

### What two-point calibration corrects

| Error source | Corrected by `minVolts`/`maxVolts`? |
|---|---|
| Shunt resistor tolerance (e.g. 1 % of 120 Ω → 1 % gain error) | Yes |
| Transmitter zero and span error | Yes |
| ADC gain/offset residual | Yes |
| Non-linearity, hysteresis | No (linear, two points) |
| Temperature effects, long-term drift | No (only by recalibrating) |
| Noise | No |

### Procedure **[Proposed – consistent with how the panel is designed to be used]**

Equipment: a reference pressure gauge or dead-weight tester of better accuracy than the transmitter, a regulated inert gas supply (N₂), and the control panel's *Voltage* graph or the `(x.xxx V)` read-out on the sensor card.

1. Let the system warm up (≥ 15 min). Connect the browser that will be used for the test.
2. **Zero point:** vent the line to atmosphere. Record the mean voltage over ≥ 20 s → `minVolts`. (Expect ≈ 0.48 V; anything below ≈ 0.40 V suggests a wiring fault.)
3. **Span point:** apply a known reference pressure `P_ref`, ideally 80–100 % of the working range. Record the mean voltage `V_ref`.
4. Enter values. Either set `barMax = P_ref`, `maxVolts = V_ref`; or keep `barMax` at nameplate full scale and extrapolate `maxVolts = minVolts + (V_ref − minVolts) × barMax / P_ref`.
5. Save (page reloads).
6. **Validate** at 2–3 intermediate pressures, rising and falling. Record displayed vs reference.
7. Note the constants and date somewhere outside the browser (they are otherwise lost with the browser profile), and repeat on every device used for operations.

### Assessing accuracy

For each check point: error `e = P_displayed − P_ref`; percentage of full scale `e / barMax × 100`; repeatability = spread of repeated readings at the same point; hysteresis = rising vs falling difference. A rough uncertainty budget (root-sum-square) should include the reference gauge accuracy, transmitter accuracy class, shunt tolerance and tempco, U3 accuracy, and 1 LSB quantisation. None of these numbers are available in the repository, so no quantitative uncertainty can be stated here.

## E. Error analysis

| Source | Effect | Present mitigation |
|---|---|---|
| ADC quantisation (≈ 0.6 mV) | `barMax / 1.92 V × 0.6 mV` per step (0.03 / 0.05 / 0.08 bar for 100 / 150 / 250 bar) | Card averaging (20 samples) |
| Electrical noise, relay/solenoid switching transients | Spikes on single samples | Card averaging only; graphs show min/max envelope when decimated |
| Shunt resistor tolerance and self-heating (20 mA² × 120 Ω = 48 mW) | Gain error / drift | Two-point calibration |
| Ground offsets between loop supply and U3 | Offset | Zero calibration |
| Transmitter temperature coefficient (cryogenic lines on LOX side) | Zero/span shift with temperature | None |
| Long sense lines, loose terminals | Dropouts → 0 V → large negative bar | None (visible but not alarmed) |
| Over-range (> 20 mA) | Voltage above ≈ 2.44 V clips at the ADC | None |
| Per-browser calibration | Different operators see different pressures | None |

### Fault signatures (derived from the formula with default constants)

| Reading on a 100 / 150 / 250 bar channel | Likely cause |
|---|---|
| ≈ −25 / −37.5 / −62.5 bar (0 V) | Open loop, transmitter unpowered, blown fuse |
| Slightly negative (−0.5…0 bar) | Normal zero offset; recalibrate `minVolts` |
| Pinned at ≈ 102 / 153 / 255 bar (≈ 2.44 V) | Over-range or shorted transmitter (ADC clipping) |
| `NaN` / blank graph | Pin missing from the server's `Sensors` list or mismatch between `stands.py` and the frontend |

### Proposed improvements (not implemented)

- Flag `V < 0.40 V` or `V > 2.42 V` as a sensor fault instead of plotting it.
- Store calibration on the server (and in the log header) so all clients and logs agree.
- Oversample/average on the server; log calibrated values as well as volts.
- Derive `formatDataPoint` from `pins.json` instead of hard-coded pin strings.
