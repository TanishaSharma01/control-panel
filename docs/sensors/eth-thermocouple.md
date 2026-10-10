# ETH Thermocouple Temperature

Shared theory, driver walk-through, formulas, calibration procedure and error analysis: **[max6675-common.md](max6675-common.md)**. This file records the ETH-specific facts.

## A. Overview

| Item | Value | Source |
|---|---|---|
| Role | Temperature on the ethanol side of the stand. The exact probe location is **not recorded** in the repository (the diagram card sits at the bottom of the ETH half) | `pins.json` L139-143 `full_name`: "ETH Thermocouple Temperature" |
| Calibration key | `eth_temp` (`type: 'temperature'`) | `utils.js` L87-92 |
| Converter | MAX6675 + type-K thermocouple (probe model unknown) | `stands.py` L88 |
| LabJack | ETH U3 (`SerialNumber = 1`) | `stands.py` L65 |
| Graph series | `'ETH Temp'` (orange, °C) | `utils.js` L244-246; `graph-panel.js` L568-577 |
| Range / resolution **[Theory]** | 0 to +1023.75 °C, 0.25 °C | MAX6675 datasheet |

## Wiring (`stands.py` L93)

```python
Thermocouple = {'sck': 3, 'cs': 2, 'so': 11, 'mosi': 1}
```

| MAX6675 pin | LabJack DIO | Terminal | Direction |
|---|---|---|---|
| SCK | 3 | FIO3 | U3 → MAX6675 |
| CS | 2 | FIO2 | U3 → MAX6675 |
| SO | 11 | EIO3 | MAX6675 → U3 |
| (none – dummy MOSI) | 1 | FIO1 | driven by the U3 during transfers; must stay unconnected |

Why these pins **[Existing, from comments L88-92]**: FIO0 is the load cell, FIO4/5/6 are pressure inputs, EIO0–2, EIO6–7 and CIO0–1 are valves, EIO4–5 and CIO2–3 are the light stand; FIO1 is "the only FIO pin nothing else on this stand claims". The ETH analog mask (`0b01110001`) leaves FIO1–FIO3 digital, as SPI requires.

> **Open question:** on a U3-**HV**, FIO0–FIO3 are fixed high-voltage analog inputs and cannot be used for SPI. This wiring implies the ETH device is a U3-LV, or that the code has not yet been run against this hardware ([known-issues.md](../known-issues.md) A12).

Note: the `labjack_pin: 2` in the `pins.json` entry is **not** used to read the sensor; temperature comes from `labjacks.ETH.temperature`. It only forms the card's averaging key `ETH_2`.

## B. Data acquisition (ETH specifics)

- Worker thread started in `LabJack.__init__` (`labjack.py` L164-165) because `ETH` has a `Thermocouple` attribute.
- Shares the ETH U3's USB handle with the 20 Hz state loop through `_device_lock`. Each SPI read briefly delays the main loop's reads and vice-versa.
- Value path: `_tc_cache` → `get_thermocouple_temp()` → `state["temperature"]` → `STATE.data.labjacks.ETH.temperature`.

## C. Calculation

1. Server: `T_raw = (word >> 3) × 0.25` °C, `None` if bit 2 is set (`labjack.py` L246-250).
2. Browser: `T = 25 + (T_raw − readingAt25)/(readingAtFullRange − readingAt25) × (fullRange − 25)` (`getTemperatureC`, `utils.js` L31-34), with `sensorData.eth_temp`.

Example (defaults → identity): SPI bytes `0x03 0x38` → `v = 824` → `824 >> 3 = 103` → `T_raw = 25.75 °C` → `T = 25 + (25.75 − 25)/75 × 75 = 25.75 °C`. The card shows `25.75 °C` (after 20 samples: their mean).

## D. Calibration status

**Uncalibrated.** Defaults (`fullRange: 100`, `readingAt25: 25.0`, `readingAtFullRange: 100.0`) are an identity mapping and are described as placeholders in the code. Use *Sensor Calibration → ETH Temperature* and the procedure in [max6675-common.md §D](max6675-common.md#d-calibration).

## E. Channel-specific error notes

- Ethanol-side temperatures are expected to be within the MAX6675's 0–1024 °C window in normal operation; only sub-zero conditions (e.g. cold nitrogen expansion) would clip at 0 °C.
- The diagram card turns red whenever the value exceeds 2 (a pressure rule applied to all non-flow cards; [known-issues.md](../known-issues.md) A6).
- The card shows no voltage line (`unit !== '°C'` guard, `control-panel.js` L239).

## Code references

`src/stands.py` L88-93 · `src/labjack.py` L164-165, L211-289, L90-91 · `reactfront/src/utils.js` L31-34, L87-92, L244-246 · `reactfront/src/components/panels/control-panel.js` L153-158 · `reactfront/src/components/panels/graph-panel.js` L567-577 · `reactfront/src/components/panels/calibration-panel.js` L202-255
