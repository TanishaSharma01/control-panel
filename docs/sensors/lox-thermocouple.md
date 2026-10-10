# LOX Thermocouple Temperature

Shared theory, driver walk-through, formulas, calibration procedure and error analysis: **[max6675-common.md](max6675-common.md)**. This file records the LOX-specific facts.

> ⚠️ **Fundamental limitation [Theory, datasheet].** The MAX6675 measures 0 °C to +1024 °C only. Liquid oxygen boils at about −183 °C at atmospheric pressure, and chilled lines are far below 0 °C. In cryogenic conditions this channel will read **0 °C** (its floor), not the real temperature. As implemented it can show that a line is *at or below freezing*, or how it warms above 0 °C, but it cannot measure LOX temperature. A sub-zero-capable converter (for example a MAX31855K, −200 to +1350 °C) and matching driver code would be required. The hardware team should confirm what this probe is intended to measure.

## A. Overview

| Item | Value | Source |
|---|---|---|
| Role | Temperature on the LOX side; exact probe location not recorded | `pins.json` L109-113 `full_name`: "LOX Thermocouple Temperature" |
| Calibration key | `lox_temp` (`type: 'temperature'`) | `utils.js` L93-98 |
| Converter | MAX6675 + type-K thermocouple (probe model unknown) | `stands.py` L57 |
| LabJack | LOX U3 (`SerialNumber = 2`) | `stands.py` L21 |
| Graph series | `'LOX Temp'` (blue, °C) | `utils.js` L247-249; `graph-panel.js` L568-577 |

## Wiring (`stands.py` L62)

```python
Thermocouple = {'sck': 1, 'cs': 0, 'so': 3, 'mosi': 5}
```

| MAX6675 pin | LabJack DIO | Terminal | Direction |
|---|---|---|---|
| SCK | 1 | FIO1 | U3 → MAX6675 |
| CS | 0 | FIO0 | U3 → MAX6675 |
| SO | 3 | FIO3 | MAX6675 → U3 |
| (none – dummy MOSI) | 5 | FIO5 | driven by the U3 during transfers; must stay unconnected |

The LOX pin budget **[Existing]**: FIO2 = inlet pressure (analog), FIO4 = tank pressure (analog), FIO6/FIO7 = hardware UART TX/RX for the flow meter, leaving FIO0/1/3 for SPI and FIO5 as the dummy MOSI ("the one genuinely free pin on this stand", L58-61). **Consequence:** the LOX N2 pressure sensor cannot return to FIO5 without moving the MOSI assignment ([lox-n2-pressure.md](lox-n2-pressure.md)).

> **Open question:** as for ETH, FIO0–FIO3 are not digital-capable on a U3-HV ([known-issues.md](../known-issues.md) A12).

The `labjack_pin: 0` in the `pins.json` entry is only used as the card's averaging key (`LOX_0`).

## B. Data acquisition (LOX specifics)

- The LOX U3 handle is shared by three consumers: the 20 Hz state loop (valves, analog, `asynchRX`), the light-stand task, and this thermocouple worker thread; `_device_lock` serialises them.
- Value path: `_tc_cache` → `state["temperature"]` → `STATE.data.labjacks.LOX.temperature`.

## C. Calculation

Identical to ETH: server `T_raw = (word >> 3) × 0.25`; browser `getTemperatureC` with `sensorData.lox_temp`.

Example in cryogenic service: the MAX6675 saturates at its lowest code, `0x0000` → `T_raw = 0.0 °C` → with defaults `T = 0.0 °C`. This is indistinguishable from a stuck-low SO line.

## D. Calibration status

**Uncalibrated** (identity defaults). Note that the provided two-point scheme is anchored at +25 °C and a hotter point; it is not designed for, and cannot fix, the sub-zero limitation.

## E. Channel-specific error notes

- **Range floor** – dominant issue, see the warning above.
- **Cold-junction range** – the MAX6675 IC itself is specified for −20 to +85 °C ambient. If the breakout is mounted near cryogenic plumbing its own temperature (and therefore the cold-junction compensation) may leave the specified range **[Theory]**.
- **Condensation/ice** on the board or connector can create leakage paths and spurious readings.
- The diagram card turns red above 2 °C due to the shared pressure-colouring rule ([known-issues.md](../known-issues.md) A6).

## Code references

`src/stands.py` L57-62 · `src/labjack.py` L164-165, L211-289, L90-91 · `reactfront/src/utils.js` L31-34, L93-98, L247-249 · `reactfront/src/components/panels/control-panel.js` L153-158 · `reactfront/src/components/panels/graph-panel.js` L567-577 · `reactfront/src/components/panels/calibration-panel.js` L202-255
