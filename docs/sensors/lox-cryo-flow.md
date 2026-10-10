# LOX Cryogenic Flow Meter (UART)

## A. Overview

| Item | Value | Source |
|---|---|---|
| Role | Volumetric flow of liquid oxygen, displayed in litres per second (LPS) | `pins.json` L103-107 `full_name`: "Cryogenic Flow Sensor" |
| Calibration key | `lox_cryo` (`type: 'flow'`) | `utils.js` L78-86 |
| Sensing element | Referred to only as "**the PT420**" (`stands.py` L50; `utils.js` L305 "PT420 calibration from sheet"). **Manufacturer and exact model could not be identified**; no datasheet is in the repository and a web search for a cryogenic "PT420" was inconclusive | – |
| Signal conditioning | An external **microcontroller** reads the meter and streams text over UART. **Its firmware is not in this repository** | `stands.py` L50-54 |
| LabJack | LOX U3, onboard hardware UART | `stands.py` L55 |
| Graph series | `'LOX Flow'` (brown, LPS) | `utils.js` L243; `graph-panel.js` L538-547 |

### What can be inferred about the meter **[Inferred]**

- The CSV stream has columns `timestamp_ms, freq_Hz, filtered_Hz, L_per_s` (`stands.py` L51, `labjack.py` L295). A frequency output is characteristic of a **turbine (or other pulse-output) flow meter**: the microcontroller measures pulse frequency, filters it, and converts it to flow.
- Before the UART design, the same sensor was read as an **analog 4–20 mA** signal on LOX FIO2 with the calibration "0.80 GPM … 29.00 GPM" ↔ "0.4684 V … 2.342 V" (`utils.js` L81-85, L305-315). That suggests a meter (or a frequency-to-current transmitter) with a 0.8–29 GPM calibrated range (0.0505–1.830 L/s). The voltage end-points imply a shunt of 0.4684 V / 4 mA = 2.342 V / 20 mA ≈ 117.1 Ω.

### Operating principle **[Theory – assuming a turbine meter]**

Fluid spins a bladed rotor; a pick-up coil produces one pulse per blade passage. Over the linear range, pulse frequency is proportional to volumetric flow:

```
Q [L/s] = f [Hz] / K [pulses per litre]
```

`K` (the K-factor) comes from the meter's calibration sheet. In this project, that conversion happens **inside the microcontroller**; the Python and JavaScript code never see `K`.

### Electrical interface (`stands.py` L55)

```python
CryoFlowUART = {'rx': 7, 'tx': 6, 'baud': 114286}
```

| Signal | LabJack pin | Direction |
|---|---|---|
| U3 TX | FIO6 | U3 → microcontroller RX (command bytes) |
| U3 RX | FIO7 | microcontroller TX → U3 (CSV stream) |
| GND | GND | common reference (required) |

- 8 data bits, no parity, 1 stop bit, TTL/CMOS levels **[Theory, U3 datasheet §4.1.12]**. The U3 I/O is 3.3 V logic; the microcontroller's logic level is not documented.
- RX must be the pin immediately after TX; `LabJack.__init__` asserts this (`labjack.py` L155-156).
- Requested baud 114 286. LabJackPython computes `BaudFactor = 2¹⁶ − 48 000 000 // (2 × baud)` = 65 536 − 209 = **65 327**, giving an actual rate of 48 MHz / (2 × 209) ≈ **114 833 baud** (+0.48 %) **[Inferred from the library formula]**. The microcontroller's exact baud rate is unknown; 114 286 corresponds to e.g. 16 MHz / 140.
- UART and SPI on the U3 need hardware version ≥ 1.21 **[Theory]**.

## B. Data acquisition and communication

### Start-up (`labjack.py` L152-163)

```python
self.device.configIO(EnableUART=True, TimerCounterPinOffset=tx)       # route UART to FIO6/FIO7
result = self.device.asynchConfig(UARTEnable=True, DesiredBaud=cfg['baud'], configurePins=False)
self._log(f"[CRYO] asynchConfig on {standConfig.name}: {result} (baud=…, TX=FIO6/RX=FIO7)")
```

`TimerCounterPinOffset=6` explicitly places the UART pair on FIO6/FIO7 instead of the library default FIO4/FIO5; `configurePins=False` stops `asynchConfig` from re-issuing a default `configIO`.

### Polling (`get_cryo_flow_lps`, `labjack.py` L291-339), once per state tick (≤ 20 Hz, main thread)

1. `result = self.device.asynchRX()` (under `_device_lock`) – one USB call returning up to **32** of the oldest bytes from the U3's **256-byte** RX buffer, plus `NumAsynchBytesInRXBuffer`.
2. `n = min(32, NumAsynchBytesInRXBuffer)`; the first `n` bytes are decoded as ASCII (`errors='replace'`) and appended to `self._cryo_line_buffer`. Each non-empty chunk is logged: `[CRYO] LOX: received n bytes: '…'`.
3. If the buffer exceeds 500 characters it is cut to its last 500 (protects against an endless stream without newlines).
4. If no bytes arrived, a heartbeat `[CRYO] LOX: polling, 0 bytes in RX buffer` is logged at most every 2 s.
5. While the buffer contains `\n`: split off one line, `strip()` it (removes `\r`), split on commas.
   - exactly 4 fields and the 4th parses as `float` → `self._cryo_cache = float(parts[3])` (logged);
   - 4 fields, 4th not numeric → logged, ignored;
   - other non-empty lines → logged as "probably a header/log line", ignored.
6. Return `self._cryo_cache` (the latest good value, or `None` if none yet).

On any exception from the USB call the traceback goes to the debug log and the previous cached value is returned.

### Throughput constraint **[Inferred]**

At most 32 bytes are drained per tick and there are ≤ 20 ticks/s → ≤ 640 bytes/s. A line such as `123456,412.5,410.2,1.0668\r\n` is ≈ 27 bytes, so the microcontroller must send fewer than roughly 20 lines/s or the 256-byte hardware buffer will overflow and lines will be corrupted (and then rejected by the 4-field check). The actual line rate is set by the unseen firmware.

### Onward path

`get_state()` (L92-93) → `state["cryo_flow_lps"]` → `STATE.data.labjacks.LOX.cryo_flow_lps` → browser.

| Property | Value |
|---|---|
| Units / type | L/s, float, or `null` before the first valid line |
| Timestamp | The MCU's `timestamp_ms` field is parsed but **discarded**; the value is stamped with the server's state time |
| Missing data | No timeout: the last value is repeated indefinitely ([known-issues.md](../known-issues.md) A2) |
| Invalid data | Non-4-field or non-numeric lines dropped; no plausibility/range check; `nan`/`inf` strings would be accepted by `float()` |

### Commands to the microcontroller

`send_cryo_command(char)` (L341-358) transmits one byte with `asynchTX([ord(char)])`. It is reached through the WebSocket header `CRYOCOMMAND` (`server.py` L488-503), sent by the "Test Mode" button in the calibration panel (`calibration-panel.js` L170-200) with `{stand:'LOX', command:'t'}`. Per the code comments, the firmware treats `'t'` as a test-mode toggle. There is **no acknowledgement**: the server flips its own belief `state.cryo_test_mode['LOX']` on each successful send, so the button label can desynchronise from the real firmware state (documented at `server.py` L77-81). The command is deliberately not gated on arming (L491-492).

### Development mode

`LabJackFake.get_cryo_flow_lps` (L413-416) returns `round(0.9 + 0.5·sin(t/5 + serial), 4)` L/s (0.4–1.4, 31 s period). `send_cryo_command` only logs "(fake) would send command".

## C. Calculations

### Implemented in this repository **[Existing]**

| Step | Formula | Code |
|---|---|---|
| Parse | `Q = float(4th CSV field)` | `labjack.py` L332 |
| Graph smoothing | `Q̄ₙ = mean(last ≤ 10 non-NaN values)` – rolling (moving) average, 0.5 s at 20 Hz; buffer reset after a > 1 s data gap | `index.js` L64-67, L129-139 |
| Card smoothing | mean of each consecutive block of 20 values (not rolling); the card refreshes about once a second | `control-panel.js` L253-288 |
| Card colour | green if `Q̄ > 0.1 GPM × 0.06309 = 0.006309 L/s` | `control-panel.js` L201-207 |

There is **no voltage, frequency or K-factor arithmetic** in the Python or JavaScript code for the current design.

Worked example: the MCU sends `"98231,412.50,410.20,1.0668\n"`.

```
parts = ['98231', '412.50', '410.20', '1.0668']   → len 4
cryo_flow_lps = 1.0668 L/s                        (≈ 16.9 GPM, for reference only)
```

If the last ten values were 1.05, 1.06, 1.07, 1.06, 1.07, 1.08, 1.06, 1.07, 1.07, 1.0668, the graphed point is their mean = 10.6568 / 10 = **1.0657 L/s**.

Note the smoothing uses *state ticks*, not MCU lines: if the MCU sends 5 lines/s, each value appears in ~4 consecutive ticks, so the 10-sample window spans only 2–3 distinct measurements.

### Performed in the microcontroller (not visible) **[Inferred]**

`freq_Hz` → `filtered_Hz` (unknown filter) → `L_per_s` (unknown K-factor / curve).

### Historical analog path – present only as commented-out code / unused function

`getLPS(volts, minFlow, maxFlow, minVolts=0, maxVolts=5)` (`utils.js` L37-44) is still exported but **no longer called**:

```
Q = max(0, (V − V_min) / (V_max − V_min) × (Q_max − Q_min) + Q_min)
```

with the old constants `Q_min = 0.050472 L/s` (0.80 GPM), `Q_max = 1.82961 L/s` (29.00 GPM), `V_min = 0.4684 V`, `V_max = 2.342 V`. Unit conversion used: 1 US GPM = 0.06309 L/s.

## D. Calibration

| Kind | Status |
|---|---|
| Factory | A "sheet" for the PT420 existed (comment `utils.js` L305), giving the 0.80–29.00 GPM span. It is not in the repository |
| Firmware conversion | In the microcontroller; constants unknown |
| Software conversion in this repo | None (value passed through) |
| Project calibration performed | **None recorded** |
| Calibration UI | The *LOX Cryo Flow* tab is a remote-control card (Test Mode button), not a calibration form (`calibration-panel.js` L170-200). `sensorData.lox_cryo` contains only `type: 'flow'` |

To calibrate, the K-factor would have to be changed in the firmware, or a scale/offset stage added in `formatDataPoint`. A practical verification **[Proposed]**: flow a known volume of water (or LN₂ if rated) through the meter into a weighed catch tank, integrate the logged L/s over time, and compare `∫Q dt` with `mass / density`. Percentage error = `(V_indicated − V_reference) / V_reference × 100`. Repeat at several flow rates to check linearity; note that turbine K-factors change with fluid viscosity and that cryogenic two-phase flow invalidates the reading.

## E. Error analysis

| Source | Effect | Present handling |
|---|---|---|
| MCU stops / cable fault | Value freezes | None in the data path; heartbeat line in the debug log only |
| RX buffer overflow, baud mismatch (+0.48 %), noise on the UART line | Corrupted lines | Lines without exactly 4 comma fields or with a non-numeric last field are discarded |
| A corrupted line that still has 4 fields and a numeric tail (e.g. a dropped digit) | Wrong value accepted | None |
| Two-phase flow / cavitation during chill-down | Over-reading, erratic frequency | Unknown firmware filter; 10-sample mean on the graph |
| Flow below the meter's minimum (0.8 GPM per the old sheet) | Non-linear or zero output | None |
| Bearing friction change at cryogenic temperature, viscosity difference from calibration fluid | K-factor shift | None |
| Test mode left on | Synthetic data mistaken for real | UI label is a best-effort guess only |
| Time skew | Value may be up to one MCU line period + buffering old | None |

Side effect to be aware of **[Existing]**: every received chunk and every parsed line is passed to `log_debug`, which `print`s it. On the Pi, stdout is redirected to `logs.txt` (`startup.sh`), so that file grows continuously while the meter is streaming, and the 50-line debug panel is dominated by `[CRYO]` messages.

### Proposed improvements (not implemented)

- Timestamp each parsed value and report `null` when older than ~1 s.
- Range-check the parsed value; use the MCU timestamp to detect dropped lines.
- Have the firmware acknowledge `'t'` and report its mode in the stream.
- Reduce `[CRYO]` logging to state changes.
- Add the firmware source and the meter's calibration sheet to the repository.

## Code references

`src/stands.py` L50-55 · `src/labjack.py` L137-139, L152-163, L291-358, L92-93, L413-422 · `src/server.py` L77-81, L171-179, L488-503 · `src/commands.py` L93 · `reactfront/src/utils.js` L37-44, L78-86, L239-243 · `reactfront/src/index.js` L64-67, L125, L129-139 · `reactfront/src/components/panels/control-panel.js` L167-173, L201-207 · `reactfront/src/components/panels/graph-panel.js` L538-547 · `reactfront/src/components/panels/calibration-panel.js` L170-200
