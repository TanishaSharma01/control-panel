# Known Issues, Discrepancies and Open Questions

Everything here was found by reading the code while writing this documentation. **No application code was changed.** Items are observations of **[Existing]** behaviour unless marked **[Inferred]**; fixes are **[Proposed]** only.

Severity: 🔴 safety / data-integrity relevant · 🟠 functional bug · 🟡 cosmetic, stale or dead code.

## A. Sensor / measurement issues

| # | Sev | Issue | Evidence |
|---|---|---|---|
| A1 | 🔴 | **MAX6675 cannot read below 0 °C.** Its range is 0 to +1024 °C, so the LOX thermocouple cannot report cryogenic temperatures (LOX boils at ≈ −183 °C); it will bottom out at 0 °C. A reading of exactly `0.0` is also what a stuck-low SO line would produce. **[Theory, datasheet]** | `labjack.py` L246-250; [sensors/lox-thermocouple.md](sensors/lox-thermocouple.md) |
| A2 | 🔴 | **Stale flow value is never invalidated.** If the flow MCU stops sending, `get_cryo_flow_lps` returns the last parsed value indefinitely. | `labjack.py` L324, L339 |
| A3 | 🔴 | **No sensor fault detection for 4–20 mA channels.** A broken loop (0 V) is displayed as a large negative pressure; there is no under-range/over-range alarm. | `utils.js` L1-5 |
| A4 | 🟠 | **`LOX N2` card shows the cryo-flow average labelled "Bar".** In `pins.json` both `lox_n2` and `lox_cryo` have `test_stand:"LOX", labjack_pin:5`, so both `ControlCard`s use the batch key `LOX_5`. Since LOX `analog["5"]` no longer exists, `lox_n2` never contributes samples but *does* display the stored average written by `lox_cryo`. Before the first 20-sample average it shows `NaN Bar`. | `pins.json` L92, L104; `control-panel.js` L148, L191-196 |
| A5 | 🟠 | **`pins.json` still lists LOX N2 on pin 5**, but `stands.py` has `N2Sensor` commented out and uses FIO5 as the SPI dummy MOSI pin for the LOX thermocouple. `formatDataPoint` still computes `'LOX N2'` (always `NaN`). | `stands.py` L35-36, L62; `utils.js` L229 |
| A6 | 🟠 | **Temperature and load-cell cards turn red above 2.** The "pressurised" colouring (`displayValue > 2`) is applied to every non-flow sensor, so any temperature above 2 °C or load above 2 kg is highlighted as if it were pressure. | `control-panel.js` L208-213 |
| A7 | 🟠 | **Red-lamp threshold comment is inconsistent.** `1.47 V ~= 600 psi`: with the current default LOX tank calibration 1.47 V = 51.6 bar = 748 psi; 600 psi (41.4 bar) would be 1.274 V. The `0.52 V ~= 2 atm` comment is consistent (2.08 bar). | `server.py` L551, L555 |
| A8 | 🟠 | **Light-stand rule table has a duplicated rule and a no-op buzzer.** The `> 1.47` rule appears twice; both buzzer rules set the buzzer to `False`, so the buzzer never sounds. Voltages exactly equal to 0.52 or 1.47 match no rule. | `server.py` L554-569 |
| A9 | 🟠 | **`shared(eth)` reads ETH pin 4 using `LOX.LOXSensor[1]`.** It works only because the ETH tank sensor is also on pin 4; the ETH U3's red output therefore follows ETH tank pressure with LOX's thresholds. Whether a physical lamp is wired to both U3s is unknown. | `server.py` L546-560, L573-576 |
| A10 | 🟠 | **Median-of-3 thermocouple reads are probably not independent.** The MAX6675 needs up to 0.22 s per conversion and aborts a conversion whenever CS goes low; three reads 5 ms apart most likely return the same completed conversion (or interrupt new ones). **[Inferred from datasheet]** | `labjack.py` L266-272 |
| A11 | 🟠 | **SPI mode unverified.** The code's own docstring says mode `'B'` has not been verified against the real wiring. | `labjack.py` L225-229 |
| A12 | 🟠 | **U3 variant / pin feasibility unconfirmed.** If either U3 is a U3-HV, FIO0–FIO3 are dedicated high-voltage analog inputs and cannot be SPI/digital pins, yet the thermocouples use FIO0/1/3 (LOX) and FIO1/2/3 (ETH). Conversely the load-cell default `maxVoltage: 5.0` exceeds the ~2.44 V range of a low-voltage input. Needs confirmation from the hardware team. **[Inferred]** | `stands.py` L62, L85, L93; `utils.js` L103 |
| A13 | 🟠 | **Requested UART baud is not exactly achievable.** LabJackPython computes `BaudFactor = 2^16 − 48000000 // (2 × 114286) = 65327`, i.e. a divisor of 209 → actual ≈ 114 833 baud (+0.48 %). Normally within UART tolerance, but worth knowing when debugging framing errors. **[Inferred from library formula]** | `stands.py` L55; `labjack.py` L162 |
| A14 | 🟡 | **Calibration is per-browser.** Constants live in `localStorage`; they are not shared, versioned, or logged with the data. | `utils.js` L110-122 |
| A15 | 🟡 | **Default thermocouple and load-cell calibrations are placeholders** (identity map; 0–5 V ↔ 0–100 kg). The code comments say so. | `utils.js` L87-104 |
| A16 | 🟡 | Calibration panel text says "kg per mV" but the formula is per **volt**. | `calibration-panel.js` L308 |
| A17 | 🟡 | `voltsToBar` / `barToVolts` ("dodgy old sensors") are unused, and `voltsToBar` divides by `R·I₂` (2.4 V) instead of the span `R·(I₂−I₁)` (1.92 V), so it would under-read by 20 % at full scale if it were used. | `utils.js` L199-215 |
| A18 | 🟡 | Browser CSV export writes missing values as `0.000`, indistinguishable from a real zero. | `graph-panel.js` L304-306 |
| A19 | 🟡 | Server CSV logs each tick's state *before* refreshing it, and embeds the whole 50-line debug log in every row. | `server.py` L154-159 |
| A20 | 🟡 | Every received UART chunk and parsed flow line is `print`ed via `log_debug`. On the Pi stdout goes to `logs.txt`, which therefore grows continuously while the flow meter streams, and the 50-line browser debug log is dominated by `[CRYO]` messages. | `labjack.py` L309, L333; `startup.sh` L6 |

## B. Control / safety logic

| # | Sev | Issue | Evidence |
|---|---|---|---|
| B1 | 🔴 | **Abort is ignored when not armed** (both the UI button and the server require `arming_switch`). | `server.py` L444-446; `components/index.js` L89-97 |
| B2 | 🔴 | **Any USB read error terminates the server** via `sys.exit()`, with no supervisor to restart it and no "safe state" on exit. Note `sys.exit()` raised inside an asyncio task/thread may not even stop the process cleanly. | `labjack.py` L171-175, L205-209 |
| B3 | 🔴 | **Outputs are not initialised at start-up** and not driven to a safe state at shutdown. Valve states shown after boot are whatever the U3 pins read. **[Inferred]** | `LabJack.__init__` |
| B4 | 🔴 | **Sequence loading executes arbitrary Python** (`exec`) and `SAVESEQUENCE` lets any armed network client write those files; `SETSEQUENCE` does not validate the name (`../` traversal to another `.py` file is possible). The WebSocket has no authentication or TLS. Acceptable only on the isolated range network. | `utils.py` L31-47; `server.py` L232-239, L432-438 |
| B5 | 🟠 | **`BEGINSEQUENCE` while already running starts a second concurrent task** popping the same command list. | `server.py` L440-442, L593-615 |
| B6 | 🟠 | An exception inside the sequence task (e.g. `setDIOState` failing) leaves `status` stuck at RUNNING/ABORTING; arming/manual switches then stay locked. | `server.py` L599-613 |
| B7 | 🟠 | `VALVE` events are broadcast even when the arming gate blocked the hardware action. | `commands.py` L52-58 |
| B8 | 🟠 | The commented-out vent/pressurisation mutual exclusion in `set_valve_state` means nothing prevents opening Pressure and Vent together. | `labjack.py` L56-69 |
| B9 | 🟠 | Overwriting `abort.py` is allowed when armed; `abort.py` is not validated after saving. A broken abort file makes `run_abort_sequence` raise. | `server.py` L282-286, L250 |
| B10 | 🟡 | `Sleep.act` returns `None` for `ms <= 0`; the command lingers as `command_in_flight` until the next pop. | `commands.py` L29 |
| B11 | 🟡 | `App.pushWarning` uses `&&`: a new warning with a *different time but the same text* is suppressed. | `index.js` L78-83 |

## C. Stale code and comments

| # | Issue | Evidence |
|---|---|---|
| C1 | `_is_inverted_relay` comment says "ETH 19, LOX 13"; the real vent pins are ETH 9, LOX 10. | `labjack.py` L97 |
| C2 | `LabJackFake._set_default_state` treats pins 13 and 19 as "vent valves default on" – stale; fake vents now default closed and the fake does not model relay inversion. | `labjack.py` L380-382 |
| C3 | `reactfront/src/components/datalogger.js` is an abandoned prototype: not imported anywhere and would not compile if it were (bad import, assignment to `const`, undefined `w`, `h`, `useReducer`). | file |
| C4 | Unused exports/imports: `SafetyCard` (imported in `sequence-panel.js`, never rendered), `NormalSwitch`, `getLPS`, `processSensorBatch`, `createSensorBatchUpdater`, `calculateBatchAverage` (only used by the unused updater), `shrinkInterval` (internal only), `abort` in `sequence-panel.js`; `ping` state is never displayed. | grep |
| C5 | Unused npm dependencies: `recharts`, `moment`, `@material-ui/lab`, `node-pre-gyp`. `asyncio` in `requirements.txt` is unnecessary on Python 3. | manifests |
| C6 | `reactfront/src/labjack-1.svg` is not referenced; unused CSS classes include `.sensor-batch-info`, `.sensor-collection-info`, `.sensor-flow-active`, `.sensor-pressure-high`, `.jump-to-present`, `.panel-row-3`, `.alert`, `.top-bar-status-text`, `.panel.safety`. | grep |
| C7 | The `ControlCard` list uses `key={sensor.pin.name}`; most sensor entries have no `name`, giving duplicate `undefined` keys (React warning). | `control-panel.js` L326 |
| C8 | Root `README.md` says to type a sequence name into a popup; the UI is now a dropdown. `.vscode/settings.json` points at another developer's Windows Python path. | files |
| C9 | The server sends `SEQUENCE_SAVED/DELETED/CONTENT`, but the frontend has no handlers (logged with `console.error`). The UI shows "Sequence saved" before the server has confirmed. | `index.js` L173-174; `sequence-panel.js` L183 |
| C10 | The frontend's `VALVE_ATTR_BY_PIN` and the "Add Command" valve list offer invalid combinations (e.g. `LOX.Igniter`, `ETH.Chill`), which fail at load time. | `sequence-panel.js` L507-517 |

## D. Information needed from the hardware team

1. Exact make/model/range/output of every pressure transmitter, and the measured value and tolerance of each shunt resistor (code assumes 120 Ω; the old cryo calibration implies ≈ 117.1 Ω on that channel).
2. Whether the U3s are U3-LV or U3-HV, and their hardware/firmware versions (SPI and UART need hardware ≥ 1.21; pin routing differs on 1.30).
3. How `SerialNumber = 1` / `2` map to the physical devices (`u3.U3(firstFound=False, serial=…)`); factory U3 serial numbers are normally 9-digit.
4. What "PT420" is (manufacturer, K-factor, calibration sheet) and the flow-meter microcontroller's firmware source: line rate, filter, test-mode behaviour, supply/logic voltage.
5. Thermocouple probe type/grade/junction style and MAX6675 breakout wiring (supply voltage, T− grounding); whether a cryogenic-capable converter is planned for LOX.
6. Load cell model, capacity, excitation, amplifier model/gain/output range, and mounting orientation.
7. Relay board type and which relays are active-low; confirmation that only the vent valves are inverted (normally-open valves).
8. Whether the light stand is wired to one or both U3s.
9. Any calibration records (reference gauge, dates, measured min/max volts) for the values the operators enter in the calibration panel.
