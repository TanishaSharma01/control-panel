# Cross-Component Integration

How the files, sensors and layers behave *together*: the lifecycle of a measurement, timing, concurrency, channel association and command propagation.

## 1. Raw vs processed vs calibrated vs displayed

| Term | Meaning in this project | Where it lives |
|---|---|---|
| **Raw** | What the DAQ returns: volts from `getAIN`; the 16-bit MAX6675 word; UART bytes | inside `labjack.py` only (the SPI word and UART bytes never leave it) |
| **Processed (server)** | Volts as floats; MAX6675 word → °C (`(v>>3)*0.25`, median of ≤3); UART CSV line → L/s float | `STATE.data.labjacks`, server CSV log |
| **Calibrated (browser)** | Volts → bar / kg via two-point linear map; °C → corrected °C via two-point map; L/s passed through | `formatDataPoint` output, `ControlCard.currentValue` |
| **Displayed** | Graph: calibrated values, min/max-decimated for the pixel width; LOX Flow additionally 10-sample rolling mean. Cards: 20-sample block mean, `toFixed(2)` | SVG / DOM |

Consequences **[Existing]**:

- The server log contains only *processed* values (volts, raw °C, L/s). Calibrated values exist only in the browser and in the browser's CSV export.
- The red light-stand logic on the server uses **hard-coded volt thresholds** (0.52 V and 1.47 V, `server.py` L546-560), which are independent of whatever calibration a browser holds.

## 2. Lifecycle of one measurement (worked trace)

**LOX tank pressure, true pressure 30 bar, default calibration.** Values marked ≈ are theoretical.

| Step | Where | Value |
|---|---|---|
| 1. Transmitter output (4–20 mA, 0–100 bar range assumed by default calibration) **[Theory]** | sensor | I = 4 mA + 16 mA × 30/100 = **8.8 mA** |
| 2. Shunt resistor 120 Ω (value from code comments) **[Inferred]** | wiring | V = 0.0088 A × 120 Ω = **1.056 V** |
| 3. U3 12-bit conversion, ~0–2.44 V single-ended range **[Theory]** | LabJack `FIO4` | ≈ 1.056 / 2.44 × 4096 ≈ 1773 counts |
| 4. `device.getAIN(4)` applies the U3's stored calibration (slope/offset) | `labjack.py` L204 | `1.056` (float, volts) |
| 5. `get_state()` | `labjack.py` L84-85 | `{"analog": {4: 1.056, 2: …}}` |
| 6. `sync_state()` stores it and stamps `state.time` | `server.py` L156-159 | `state.labjacks['LOX']`, `time = 1785127770207` |
| 7. `json.dumps` | `server.py` L635 | `"analog": {"4": 1.056, …}` |
| 8. Browser: `data.data.time = parseInt(time)/1000` | `index.js` L118 | `1785127770.207` s |
| 9. `formatDataPoint` → `getBar(1.056, 100, 0.48, 2.4)` | `utils.js` L227, L1-5 | (1.056 − 0.48) / (2.4 − 0.48) × 100 = **30.0 bar** |
| 10. `newData(dataPoint)` → `'LOX Tank'` pushed to the Pressure datalogger, `'LOX Tank V'` to the Pressure (hidden) and Voltage dataloggers | `graph-panel.js` L69-76 | `(1785127770.207, 30.0)` and `(…, 1.056)` |
| 11. `ControlCard` for `lox_tank` recomputes `getBar`, appends to batch `LOX_4`; after 20 samples the card shows their mean | `control-panel.js` L175-196 | `30.00 Bar (1.056V)`, red background because > 2 |
| 12. If server logging is on, the *next* tick writes the whole state dict (volts) to CSV | `server.py` L154-155 | row `t,STATE,"{… 'analog': {4: 1.056 …"` |
| 13. Light stand (1 Hz): 0.52 V < 1.056 V < 1.47 V → red lamp solid on | `server.py` L550-552 | `setDIOState(18, 1)` |

A second trace (thermocouple) is in [sensors/max6675-common.md](sensors/max6675-common.md); a third (flow) in [sensors/lox-cryo-flow.md](sensors/lox-cryo-flow.md).

## 3. Timestamps

| Timestamp | Source clock | Unit | Used for |
|---|---|---|---|
| `STATE.data.time` | Pi, `time.time()` after both LabJacks have been read | ms (→ s in browser) | X value of every graph point; "now" for all graphs (`currentSeconds`) |
| Envelope `time` (all server messages) | Pi, at `emit` | ms | X position of `VALVE` event markers (`index.js` L167) |
| `VALVE.data.time` | Pi, `utils.time_ms()` in `Open/Close.act` | ms | carried along, not used for plotting |
| Client message `time` | browser | ms | `PING` latency only |
| Server CSV `TIME` | Pi, seconds since that log file was opened | s (float) | offline analysis |
| Browser CSV `Timestamp (s)` | Pi sample time minus first sample time | s, 3 dp | offline analysis |

Design intent **[Existing, from code comments]**: all plotted times come from the Pi so that a Pi with a wrong wall clock (no RTC, no internet) still yields consistent graphs (`graph-panel.js` L90-93, `index.js` L164-165).

Limitations **[Inferred]**:

- One timestamp is applied to *all* channels of both LabJacks, although they were read sequentially over several tens of milliseconds, and although the thermocouple (≈1 Hz, cached) and the flow value (as old as the last UART line) are older than the stamp.
- The server CSV logs the state *before* it is refreshed in each loop iteration (L154-159), so each `STATE` row is the previous tick's data stamped with a slightly later log time (~one loop period of skew).

## 4. Sampling rates

| Quantity | Nominal rate | Limiting code |
|---|---|---|
| `STATE` broadcast, analog voltages, valve states | ≤ 20 Hz: the loop does its reads **then** sleeps 50 ms, so the period is 50 ms + read time | `sync_state`, `STATE_BROADCAST_FREQUENCY` |
| Thermocouples | ≈ 1 Hz (3 SPI reads 5 ms apart, then `sleep(1.0)`) – repeated in every 20 Hz state message | `_thermocouple_worker` |
| Cryo flow | Up to 32 UART bytes fetched per tick; value updates when a full line has arrived | `get_cryo_flow_lps` |
| Light stand | 1 Hz (blink half-period 1 s) | `init_state_handler` |
| UPS | 1 Hz, one subprocess each | `update_UPS_status` |
| Card text | Once per 20 valid samples (≈ 1 s) | `SENSOR_BATCH_SIZE` |
| Graph redraw | At most once per animation frame per datalogger | `deferredForceUpdate` |

Per-tick USB transaction count **[Inferred from `get_state`]**: LOX = 7 valve reads + 2 AIN + 4 light-stand reads + 1 `asynchRX` = 14; ETH = 7 + 4 + 4 = 15. The code's own comment (`labjack.py` L215-217) estimates 1–5 ms per USB round trip, so the real state rate is probably noticeably below 20 Hz on hardware. It has not been measured in the repository.

## 5. Channel association

There is no channel-discovery or self-description. A reading is tied to a meaning by **pin number**, duplicated by hand in four places:

| Place | Role |
|---|---|
| [`src/stands.py`](../src/stands.py) | Source of truth for what the server reads/drives |
| [`reactfront/src/pins.json`](../reactfront/src/pins.json) | Which card/switch on the diagram reads which `(test_stand, labjack_pin)` |
| [`reactfront/src/utils.js`](../reactfront/src/utils.js) `formatDataPoint` | Hard-coded `analog["4"]`, `["5"]`, `["2"]`, `["6"]`, `["0"]` → graph series names |
| [`reactfront/src/components/panels/sequence-panel.js`](../reactfront/src/components/panels/sequence-panel.js) `VALVE_ATTR_BY_PIN` | pin → Python attribute name, to regenerate `Open(ETH.Main)` text |

Plus `control-panel.js` L385, which hard-codes the igniter as `{name:'ETH', pin:10}`. When a sensor moves pins all of these must be edited together; current mismatches are listed in [known-issues.md](known-issues.md).

Multiple sensors are handled "concurrently" simply by being read one after another in the same `get_state()` call and shipped in the same message.

## 6. Buffering and storage

| Buffer | Size / policy |
|---|---|
| U3 UART RX buffer (hardware) | 256 bytes; drained ≤ 32 bytes per tick **[Theory, LabJack docs]** |
| `LabJack._cryo_line_buffer` | Unbounded string trimmed to its last 500 chars if it exceeds 500 |
| `LabJack._cryo_cache`, `_tc_cache` | Single latest value, **never expires** |
| `SystemState.debug_log` | Last 50 lines |
| `App.loxFlowBuffer` | 10 samples; cleared on a > 1 s gap |
| `ControlPanel.sensorBatches` | 20 samples per sensor key, block-averaged, reset after each average (and lost when the Control Panel view unmounts) |
| Graph series (`series.js`) | 524 288 points per segment per series (~7.3 h at 20 Hz); beyond that pushes fail (caught, logged to console). A NaN starts a new segment |
| Server CSV | One file per logging session, flushed per row, never rotated |

## 7. Command and configuration propagation

- **Valve commands**: UI → server → hardware → read-back → all UIs (see [data-flow.md](data-flow.md) §3). Multiple browsers stay consistent because the UI renders only server state.
- **Arming / manual / logging toggles**: same pattern; the toggle shows `state.data.arming_switch` etc.
- **Sequences**: edited in one browser, saved to the Pi's filesystem, then visible to all through `available_sequences` and `current_sequence`.
- **Calibration constants**: *do not propagate*. They are written to the editing browser's `localStorage` and the page is reloaded; no message is sent to the server and other browsers are unaffected.
- **Server address**: `localStorage["wsaddr"]`, per browser.
- **Pin configuration**: compile-time/hand-edited; the U3 direction/analog configuration is written once in `LabJack.__init__`.

## 8. Reliability characteristics

| Scenario | Behaviour **[Existing]** |
|---|---|
| Browser loses connection | `onclose` → warning "Server connection lost", NaN break in graphs, reconnect attempt every 1 s. Valves stay as they were. After 900 s with zero clients the abort sequence runs once |
| LabJack USB read fails (`getAIN` / `getDIOState`) | Traceback printed, **`sys.exit()`** – the whole server process terminates (`labjack.py` L171-175, L205-209). Because `rc.local` does not supervise it, it stays down until reboot; relays stay latched |
| `setDIOState` fails | Exception propagates: for a manual command it is caught in `event_handler` and shown as a warning; inside a sequence task it kills that task, leaving `status` stuck at RUNNING/ABORTING |
| SPI read fails | Caught, that read is skipped; if all 3 fail the temperature becomes `null` |
| UART read fails | Caught, logged to the debug panel, last cached flow value returned |
| Thermocouple open | D2 bit → `None` → `NaN` in graphs, card shows nothing new (keeps last 20-sample average) |
| Flow MCU stops transmitting | Last L/s value is reported forever; only the debug log ("polling, 0 bytes") reveals it |
| 4–20 mA loop broken | 0 V → large negative pressure (e.g. −25 bar on a 100 bar channel); no alarm |
| Malformed client message | Exception caught per message; warning pushed; connection kept |
| Two `BEGINSEQUENCE` in a row | Two tasks consume the same list concurrently – see [known-issues.md](known-issues.md) |
