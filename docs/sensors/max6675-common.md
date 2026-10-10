# Thermocouple Channels – Common Measurement Chain (K-type → MAX6675 → SPI → LabJack U3)

Both temperature channels use the same converter IC and the same code. Channel-specific pinouts and caveats: [ETH thermocouple](eth-thermocouple.md) · [LOX thermocouple](lox-thermocouple.md).

## A. Overview

| Item | Value | Source |
|---|---|---|
| Converter | **MAX6675** (Maxim / Analog Devices), cold-junction-compensated K-thermocouple-to-digital converter | Named in `labjack.py` L212, `stands.py` L57/L88, `utils.js` L25 |
| Breakout board / supplier | Unknown | – |
| Thermocouple probe | Type K is the only type the MAX6675 supports **[Theory]**; probe model, sheath, junction style and tolerance class are unknown | – |
| Measured quantity | Temperature of the probe's hot junction | – |

### Operating principle **[Theory]**

A thermocouple is two dissimilar metal wires (type K: chromel/alumel) joined at the measuring ("hot") junction. A temperature difference between that junction and the far ("cold"/reference) end produces a small Seebeck voltage, for type K about 41 µV/°C:

```
V_out ≈ 41 µV/°C × (T_hot − T_cold)
```

The thermocouple only measures a *difference*, so the converter must know `T_cold`. The MAX6675 measures its own die temperature with an internal diode (cold-junction compensation), adds the equivalent voltage, digitises the result with a 12-bit ADC and presents it over a read-only SPI interface. It assumes the linear 41 µV/°C characteristic, so the K-type curve's non-linearity is not corrected.

### MAX6675 specifications **[Theory – from the manufacturer datasheet; confirm against the PDF listed in [references.md](../references.md)]**

| Parameter | Value |
|---|---|
| Thermocouple type | K only |
| Measurement range | **0 °C to +1023.75 °C** ("0 to +1024 °C"); cannot report negative temperatures |
| Resolution | 12 bit, **0.25 °C/LSB** |
| Accuracy | 8 LSB (≈ ±2 °C) for 0 °C to +700 °C; worse above |
| Conversion time | 0.17 s typical, 0.22 s maximum |
| Interface | SPI-compatible, read-only, 16 clock cycles per read, SCK ≤ 4.3 MHz, data read on the falling clock edge |
| Supply | 3.0–5.5 V |
| IC ambient range (cold junction) | −20 °C to +85 °C |
| Open-thermocouple detection | Bit D2 goes high when the input is open; requires T− to be grounded |
| Conversion control | Pulling CS low **stops any conversion in progress**; a new conversion starts when CS returns high |

Output word (MSB first):

| Bit | 15 | 14 … 3 | 2 | 1 | 0 |
|---|---|---|---|---|---|
| Meaning | dummy sign, always 0 | 12-bit temperature, MSB first | open-thermocouple flag | device ID, 0 | three-state |

## B. Data acquisition

### Pin configuration (`src/stands.py`)

```python
# LOX (L62)
Thermocouple = {'sck': 1, 'cs': 0, 'so': 3, 'mosi': 5}
# ETH (L93)
Thermocouple = {'sck': 3, 'cs': 2, 'so': 11, 'mosi': 1}
```

Numbers are LabJack DIO indices (0–7 = FIO0–7, 8–15 = EIO0–7). The MAX6675 has no data input, but the U3's SPI command requires a MOSI pin and drives it as an output during the transfer, so a genuinely unused pin is nominated (`stands.py` L58-61, L89-92).

### Driver (`src/labjack.py`)

**Start-up** – `LabJack.__init__` L164-165: if the stand config has a `Thermocouple` attribute, a daemon thread running `_thermocouple_worker` is started.

**One read** – `_read_thermocouple_once(sck, cs, so, mosi)` L211-250:

```python
with self._device_lock:
    result = self.device.spi([0x00, 0x00], AutoCS=True, SPIMode='B', SPIClockFactor=0,
                             CSPinNum=cs, CLKPinNum=sck, MISOPinNum=so, MOSIPinNum=mosi)
hi, lo = result['SPIBytes']
v = (hi << 8) | lo
if v & 0x4:            # bit 2: open thermocouple fault
    return None
return (v >> 3) * 0.25 # bits 14:3 → °C at 0.25°C resolution
```

- `device.spi` is the U3's *hardware* SPI engine (one USB transaction clocks out two dummy bytes and returns the two bytes read). `AutoCS=True` lets the U3 assert/release chip-select around the transfer.
- `SPIClockFactor=0` selects the fastest clock; LabJack support quotes about 80 kHz for the U3 – far below the MAX6675's 4.3 MHz limit.
- `SPIMode='B'` – the docstring (L225-229) states this is **not verified against the real wiring**, and suggests trying `'A'` if readings look garbled.
- Any exception from the USB call is caught, the traceback printed, and `None` returned.

**Sampling loop** – `_thermocouple_worker` L252-277 (runs forever on its own thread):

```
repeat:
    readings = up to 3 non-None reads, sleeping 5 ms after each
    temp = median of the readings (middle element of the sorted list), or None if all failed
    store temp in self._tc_cache under self._tc_lock
    sleep 1.0 s
```

With 2 valid readings, `sorted(readings)[len//2]` picks the larger one.

**Hand-off** – `get_thermocouple_temp()` L279-289 returns the cached value (non-blocking). `get_state()` L90-91 puts it in `state["temperature"]` on every 20 Hz tick.

| Property | Value |
|---|---|
| Effective update rate | ≈ 1 Hz (loop period ≈ 1.0 s + 3 × (5 ms + USB time)) |
| Reported rate | 20 Hz (same cached value repeated ≈ 20 times) |
| Units / type on the wire | °C, JSON number with 0.25 steps, or `null` |
| Startup value | `null` until the first worker iteration finishes |
| Staleness | If the worker thread died, the last value would be reported forever (the loop body is wrapped in `try/except`, so this is unlikely) |
| Thread safety | `_device_lock` serialises USB access with the main loop; `_tc_lock` guards the cache |

### Timing caveat **[Inferred from the datasheet]**

Because CS-low aborts a conversion and a conversion takes up to 0.22 s, the three reads spaced 5 ms apart cannot each see a fresh conversion: the first read returns the conversion completed during the preceding 1 s sleep, and reads two and three most likely return the same value again (and restart the conversion each time). The "median of 3" therefore rejects *communication* glitches (a corrupted SPI transfer) but does not average three independent temperature samples. The comment "MAX6675 needs time to settle" (L270) understates the required interval. A true multi-sample filter would need ≥ 0.25 s between reads.

### Development mode

`LabJackFake.get_thermocouple_temp` (L410-411) returns `round(25.0 + 5.0·sin(0.1·t + serial_number), 2)` – a slow 20–30 °C sine, for both stands, with no `Thermocouple` check.

## C. Calculations

### 1. Raw word → °C **[Existing]**, `labjack.py` L246-250

```
v      = hi × 256 + lo            (16-bit word)
fault  = v AND 0x0004             (D2)
counts = v >> 3                   (drop D2..D0; D15 is always 0)
T_raw  = counts × 0.25  °C
```

Worked example: SPI bytes `hi = 0x0C`, `lo = 0x88`:

```
v      = 0x0C88 = 3208
v & 4  = 0            → thermocouple connected
counts = 3208 >> 3 = 401
T_raw  = 401 × 0.25 = 100.25 °C
```

Open-circuit example: `0x0C8C` → `v & 4 = 4` → `None`. A floating/stuck-high SO line (`0xFFFF`) also has D2 set and yields `None`; a stuck-low line (`0x0000`) yields a plausible-looking `0.0 °C`.

### 2. Median filter **[Existing]**, L272 – see §B.

### 3. Two-point linear correction **[Existing]**, `getTemperatureC`, `reactfront/src/utils.js` L31-34

```js
const c = 25 + (raw - readingAt25) / (readingAtFullRange - readingAt25) * (fullRange - 25);
```

```
T = 25 + (T_raw − R25) / (RFR − R25) × (TFR − 25)
```

| Symbol | Code name | Unit | Meaning |
|---|---|---|---|
| T_raw | `raw` | °C | Value from the server (`labjacks.<STAND>.temperature`) |
| R25 | `readingAt25` | °C | What the MAX6675 reported when the probe was truly at 25 °C |
| RFR | `readingAtFullRange` | °C | What it reported when the probe was truly at `fullRange` |
| TFR | `fullRange` | °C | True temperature of the second reference point |

A straight line through (R25, 25) and (RFR, TFR). With the defaults (`R25 = 25`, `RFR = 100`, `TFR = 100`, `utils.js` L87-98) it is the identity: `T = 25 + (T_raw − 25)/75 × 75 = T_raw`.

Applied in `formatDataPoint` (L244-249) for the Temperature graph and in `ControlCard` (`control-panel.js` L153-158) for the diagram card. If the server value is `null`, the graph gets `NaN` (a gap).

### Not implemented (theory only)

- NIST ITS-90 type-K polynomial linearisation.
- Use of the MAX6675's sub-zero-capable successors' data formats (e.g. MAX31855's 14-bit signed value and separate cold-junction reading).
- Any time-domain filtering on the server.

## D. Calibration

| Kind | Status |
|---|---|
| Factory | The MAX6675 is factory-trimmed (hence the ±8 LSB specification) **[Theory]** |
| Software conversion | `(v>>3)*0.25` – exact digital decoding, not a calibration |
| Project calibration | **None recorded.** Defaults are an identity map and the code comments call `fullRange` a "placeholder, needs a real known-hot reference" (`utils.js` L89) |
| Mechanism available | Two-point correction above, edited in *Sensor Calibration → ETH/LOX Temperature* (`calibration-panel.js` `renderTemperatureSensor`, L202-255), stored in the browser's `localStorage` under `sensorCalibration.eth_temp` / `.lox_temp` |

Errors addressed by the two-point correction: probe tolerance (a standard-grade type-K probe is typically ±2.2 °C or ±0.75 % **[Theory]**), converter offset/gain error, static cold-junction error. Not addressed: non-linearity, cold-junction error that changes with board temperature, noise.

### Procedure **[Proposed; matches the instructions shown in the panel]**

Equipment: a trusted reference thermometer (calibrated RTD/thermistor probe), a stirred water bath or dry-block.

1. Co-locate the thermocouple tip and the reference probe at room temperature; wait for both to stabilise. Read the *uncorrected* value (reset the channel to defaults first so the card shows `T_raw`).
2. If the room is at exactly 25 °C, enter the reading as `readingAt25`. If the room is at another temperature `T1` with reading `R1`, use a second point to solve for the 25 °C reading: `R25 = R1 + (25 − T1) × (R2 − R1)/(T2 − T1)`.
3. Heat the bath to a second known temperature `T2` (e.g. boiling water corrected for altitude, or a dry-block at the upper end of the expected range). Record the reading `R2`.
4. Enter `fullRange = T2`, `readingAtFullRange = R2`, `readingAt25 = R25`. Save.
5. Validate at an intermediate temperature and, for an absolute check, in a stirred ice bath (true 0 °C; the MAX6675 should read 0–2 °C).

### Numerical example (illustrative values)

Reference points: probe truly at 25.0 °C → MAX6675 reports 26.50; truly at 100.0 °C → reports 98.00. Enter `readingAt25 = 26.5`, `readingAtFullRange = 98.0`, `fullRange = 100`.

Later raw reading 60.00 °C:

```
T = 25 + (60.00 − 26.50) / (98.00 − 26.50) × (100 − 25)
  = 25 + 33.50 / 71.50 × 75
  = 25 + 35.14
  = 60.14 °C
```

Accuracy assessment: compare corrected readings with the reference at check points; report error (°C), repeatability over repeated immersions, and note that the 0.25 °C resolution bounds achievable precision.

## E. Error analysis

| Source | Effect | Mitigation in code |
|---|---|---|
| Range floor at 0 °C | Sub-zero temperatures read 0 °C | None – see [lox-thermocouple.md](lox-thermocouple.md) |
| Quantisation | 0.25 °C steps | None |
| Cold-junction error (MAX6675 board at a different temperature from the thermocouple connector; board self-heating; board outside −20…+85 °C) | Offset that drifts with ambient | None |
| K-type non-linearity vs the assumed 41 µV/°C | Grows towards range extremes | Partially, via two-point correction near the calibration points |
| EMI from relays/solenoids/igniter on the µV-level thermocouple leads | Spikes or SPI corruption | Median of 3 SPI reads (rejects bad transfers) |
| Grounded-junction probe touching grounded plumbing | Ground loop; wrong or open-fault readings | None |
| Open/broken thermocouple | D2 flag | Returned as `None` → gap in graph |
| Wrong SPI mode / wiring | Bit-shifted values (e.g. half/double) or constant 0 / fault | None; flagged as unverified in the code |
| Thermal lag of the probe | Slow response to transients | None |
| 1 Hz update | Fast transients (ignition) under-sampled | None |

### Proposed improvements (not implemented)

- Space the reads ≥ 0.25 s apart or read once per cycle.
- Treat `0x0000` as suspect; mask D15.
- Add a staleness timestamp to the cached value.
- Use a converter with a sub-zero range (e.g. MAX31855K: −200 to +1350 °C) for the LOX side.
