# Testing, Debugging and Troubleshooting

## 1. What exists today

| Kind | Exists? | Details |
|---|---|---|
| Unit tests | **No** | No test files, no test runner configuration, no `test` npm script |
| Integration tests | **No** | – |
| Hardware tests | **No** scripted tests | – |
| CI | **No** | No workflow files |
| Simulation | **Yes** | `python server.py --dev` → `LabJackFake` (synthetic voltages, temperature, flow; valve state memory; simulated UPS cycling every minute) |
| Diagnostics | **Yes** | Browser debug-log panel (server `log_debug`), warning snackbar, `logs.txt` on the Pi, server CSV data log, raw-voltage graph, `(x.xxxV)` on sensor cards, `[CRYO]` UART trace, graph CSV/SVG export |

Everything under "Recommended" below is a proposal, not existing functionality.

## 2. Running the software

### Development (no hardware)

```sh
# terminal 1 – backend
cd src
pip install websockets          # LabJackPython is not needed in --dev mode
python server.py --dev          # prints "Hosting server on <ip> port 8888"

# terminal 2 – frontend
cd reactfront
npm ci
npm start                       # http://localhost:3000
```

In the page, set **Server IP** to the address the backend printed (not `localhost` – the server binds only to the LAN address returned by `get_local_ip()`).

### On the Raspberry Pi

Automatic at boot via `startup.sh`. Manual: `pkill python3.10`, then `cd ~/control-panel/src && python3.10 server.py`. Frontend at `http://192.168.0.5:3000`. Full provisioning steps are in the root [`README.md`](../README.md).

## 3. Verifying each component

| Component | Check | Expected |
|---|---|---|
| Backend start | terminal / `logs.txt` | `Hosting server on … port 8888`; in real mode a `[CRYO] asynchConfig on LOX: {...}` line |
| WebSocket | top bar | "Status: Connected" (green) |
| State stream | graphs | Lines advance continuously; in dev mode slow sine waves |
| Arming interlock | flip a valve switch un-armed | Switch is disabled; nothing happens |
| Manual control | Arming on, Manual on, flip a switch | Switch moves within ~0.1 s; a dashed event marker appears on graphs; on hardware the relay clicks |
| Vent inversion (hardware) | open a vent valve | The relay output goes **low** while the UI shows "open" |
| Sequence | Arming on, choose `operation`, Start | Execution list advances; sleeps count down in the Config panel; status returns to idle |
| Pause / resume | Pause during a sleep, then Start | Countdown freezes, then continues from the same value |
| Abort | ABORT during a sequence | List switches to `abort`; arming and manual turn off |
| Logging | Logging toggle on | A new file in `src/logs/`; rows accumulate |
| UPS | pull UPS mains (hardware) / wait 1 min (dev) | Top bar changes; warning snackbar |
| Light stand | connect / arm / manual | Green solid; yellow solid; yellow blinking |
| Inactivity abort | close all clients for > 15 min | `ABORTING` row in the log (if logging), valves per `abort.py` |
| Cryo UART | Debug log panel | `received N bytes` and `parsed line … -> L/s=…`; or `polling, 0 bytes in RX buffer` every 2 s if silent |
| Thermocouples | Temperature graph | Room temperature ± a few °C; changes when the tip is warmed by hand |

## 4. Are the sensors plausible?

Use the **Raw Voltages** graph first; engineering values depend on per-browser calibration.

| Sensor | Plausible at rest | Suspicious |
|---|---|---|
| Pressure (any) | ≈ 0.48 V (4 mA) at zero pressure → ≈ 0 bar | ≈ 0 V (open loop / unpowered); ≈ 2.44 V (clipped); noisy jumps when relays switch |
| Tank pressures after pressurising | Tracks the regulator setting; N₂ supply ≥ tank ≥ inlet when valves are open | Tank above supply; inlet above tank |
| Load cell | Stable voltage at no load; rises (or falls, depending on wiring) when the stand is pushed by hand | Stuck at 0 V or at ≈ 2.44 V; no response to a push |
| Thermocouples | Ambient ± ~3 °C; both channels similar when at the same temperature | Exactly `0.0` (stuck-low SO, wrong SPI mode, or below 0 °C); blank (`null` → open thermocouple or SPI error); values that are exactly half/double (bit shift → wrong SPI mode) |
| Cryo flow | 0 with no flow; non-zero only when liquid moves; test mode produces the firmware's synthetic pattern | A constant non-zero value with valves closed (stale cache or test mode on) |

Cross-checks: compare a transmitter with a mechanical gauge on the same volume; compare both thermocouples in the same water bath; integrate flow over a fill and compare with the tank volume.

## 5. Common failure scenarios

| Symptom | Likely cause | What to do |
|---|---|---|
| Page loads, "Disconnected" | Backend not running (LabJack missing at boot, crash, no network route) or wrong Server IP | `ssh pi@192.168.0.5`, `tail ~/control-panel/logs.txt`; restart the server or `sudo reboot` with LabJacks plugged in |
| Backend prints "Could not connect to the LabJacks" | USB unplugged/unpowered; Exodriver missing; wrong `SerialNumber`; LabJackPython ≠ 2.0.4 | Check `lsusb`; reinstall per README; verify serials |
| `AssertionError: … must be exactly v2.0.4` | Wrong LabJackPython version | `sudo pip install LabJackPython==2.0.4` |
| `OSError: [Errno 101] Network is unreachable` at start | `get_local_ip()` with no default route | Connect a network / fix static IP config |
| Server dies during operation | A `getAIN`/`getDIOState` USB error → `sys.exit()` | Check USB cabling/hubs/EMI; restart; valves remain in their last state – make the stand safe manually |
| Switches greyed out | Arming and/or Manual off, or a sequence is running (manual is forced off) | Enable both |
| Arming toggle will not move | A sequence is running or stuck (`status ≠ 0`) | Clear the sequence; if stuck after an error, restart the server |
| "Could not load sequence X" | Syntax error, blank line, comment, or invalid valve (e.g. `LOX.Igniter`) in the file | Fix `src/sequences/X.py` on the Pi |
| "Sequence saved" but nothing changed | Not armed (server ignored `SAVESEQUENCE`) | Arm, save again |
| ABORT does nothing | Not armed | Arm first (or use manual hardware safing) |
| Pressure shows large negative value | 0 V input: broken loop, no loop power | Check wiring, supply, shunt |
| Pressure offset by a constant | Zero not calibrated in this browser | Calibrate `minVolts` |
| Different numbers on two laptops | Per-browser calibration | Enter the same constants on both / reset both |
| `LOX N2` card shows `NaN Bar` or a flow-like number | Sensor disabled server-side; batch-key collision | Known issue A4/A5 |
| Temperature card is red | Colour rule `> 2` applies to all non-flow cards | Cosmetic (A6) |
| LOX temperature reads 0 °C | MAX6675 lower limit | Hardware limitation (A1) |
| Temperature blank | Open thermocouple (D2), SPI failure, or worker not yet run | Check probe connection and wiring; watch terminal for tracebacks |
| Flow frozen at a value | MCU stopped; cable; or test mode | Debug log: look for `polling, 0 bytes`; toggle test mode; power-cycle the MCU |
| Debug log shows garbled `received` text | Baud mismatch / wrong TX-RX wiring / missing ground | Check wiring: MCU TX → FIO7, MCU RX → FIO6, common GND; confirm firmware baud ≈ 114 286 |
| "line … doesn't have 4 comma-separated fields" | Firmware header/log lines, or RX overflow | Harmless if occasional; if constant, reduce firmware output rate |
| Graph is empty / flat | Series toggled off in the legend; window scrolled into the past; no data | Click the legend entry; click the "jump to present" arrow |
| Graph line has gaps | > 1 s between state messages (server stall, Wi-Fi) or reconnect | Check link quality and server load |
| Graph stops accumulating after hours | Segment capacity (≈ 7.3 h at 20 Hz) reached | Reload the page (export CSV first) |
| Calibration "lost" | Different browser/profile/origin, or site data cleared | Re-enter; keep a written record |
| Valve markers at the wrong time | Should not happen (server timestamps); if seen, check Pi clock jumps (NTP sync mid-test) | – |
| UPS shows "Unknown" | `apcupsd` not installed/running, or USB link to the UPS missing | `apcaccess status` on the Pi |

## 6. Debugging procedures

### Communication (browser ↔ server)

1. Browser dev tools → Network → WS: is the socket open, are `STATE` frames arriving at ~20 Hz?
2. `ping 192.168.0.5` from the laptop; check the Ubiquiti link.
3. On the Pi: `ss -ltnp | grep 8888`; `tail -f ~/control-panel/logs.txt`.
4. Minimal client:

```python
import asyncio, json, websockets
async def main():
    async with websockets.connect("ws://192.168.0.5:8888") as ws:
        print(json.loads(await ws.recv())["data"]["labjacks"])
asyncio.run(main())
```

### Communication (server ↔ LabJack)

Stop the server (only one process can hold a U3) and test interactively on the Pi:

```python
import u3
d = u3.U3(firstFound=False, serial=2)   # LOX
print(d.configU3())                      # hardware/firmware version, U3-LV vs HV
print(d.getAIN(4))                       # tank pressure volts
```

### Incorrect readings

1. Look at volts, not units. 2. Measure the same node with a multimeter (across the shunt for 4–20 mA channels). 3. If volts agree → calibration constants (reset to defaults and compare). 4. If volts disagree → pin mapping (`stands.py` vs wiring) or analog configuration (`FIOAnalog` mask).

### Calibration errors

Reset the sensor to defaults, confirm the default formula gives the expected value for the measured voltage (worked examples in each sensor document), then re-enter constants. Inspect the stored values in the browser console: `JSON.parse(localStorage.sensorCalibration)`.

### Missing data

`NaN` in a graph ⇒ the key is missing from the `STATE` message. Print one frame (script above) and compare `analog` keys with `formatDataPoint`'s hard-coded pins and with `stands.py` `Sensors`.

### Visualisation issues

Check the browser console for exceptions in `onmessage` (a throw in `formatDataPoint` stops all updates). Reload the page. Verify `pins.json` positions if widgets are misplaced after CSS or SVG changes.

### Offline analysis of a server log

See the snippet in [files/src_datalog.py.md](files/src_datalog.py.md#7-reading-a-log-example). Remember the values are uncalibrated volts.

## 7. Recommended tests (not yet implemented)

### Unit tests – backend (`pytest`)

| Target | Cases |
|---|---|
| `Sleep.act` | completes after N ms; interruption stores remaining time; resume completes; `seconds=0` |
| `Open/Close.act` with a stub server | gate respected; fake LabJack state changes; `VALVE` broadcast content |
| `load_sequence` | every shipped sequence parses; invalid lines raise; all referenced valves exist |
| `ControlPanelServer` with `LabJackFake` | each `handle_command` branch and its gate; abort from IDLE and from RUNNING; pause/resume; double `BEGINSEQUENCE` |
| `SystemState.as_dict` | JSON-serialisable; restores attributes |
| MAX6675 decoding (extract into a pure function) | `0x0C88 → 100.25`; `0x0004` bit → `None`; `0x0000 → 0.0`; `0x7FF8 → 1023.75` |
| Cryo line parser (extract) | split lines across chunks; `\r\n`; header lines; non-numeric field; buffer overflow trim |
| `_is_inverted_relay` / `get_valve_state` | vent inversion on both stands |
| `Datalog` | header, row format, newline stripping |

### Unit tests – frontend (Jest, already bundled with `react-scripts`)

| Target | Cases |
|---|---|
| `getBar` | 0.48 V → 0; 2.4 V → `barMax`; 1.44 V → 50 % |
| `getLoadCellKg`, `getTemperatureC` | end points; identity with defaults; the numerical examples in the sensor docs |
| `formatDataPoint` | full message; missing LOX keys; `null` temperature/flow |
| Calibration store | merge of stored over defaults; reset |
| `series.js` | decimation example in the file doc; NaN segmentation; `sample` level selection |
| `graph-utils.js` | `binarySearch` boundaries; `generateAxisTicks`; `formatUnit` |
| Consistency | `pins.json` buttons ↔ `VALVE_ATTR_BY_PIN`; unique sensor batch keys |

### Cross-language consistency test

A script that imports `stands.py` and reads `pins.json`, asserting every valve pin matches and every analog sensor pin in `pins.json` is in the corresponding `Sensors` list. This would have caught the LOX N₂ mismatch.

### Hardware-in-the-loop checks

- **Relay walk:** toggle each valve in turn with the solenoid supply disabled and confirm the correct relay/LED, including vent inversion and the igniter channel into a dummy load.
- **Analog injection:** drive each analog input from a 4–20 mA loop calibrator (4, 8, 12, 16, 20 mA) and record volts and displayed bar; expected 0.48 / 0.96 / 1.44 / 1.92 / 2.40 V and 0 / 25 / 50 / 75 / 100 % of `barMax`.
- **Thermocouple:** ice bath (0 °C) and boiling water; or a thermocouple simulator.
- **Load cell:** dead weights across the range, loading and unloading.
- **Flow:** firmware test mode for the data path; bucket-and-stopwatch water test for the meter.
- **Timing:** log `STATE.time` deltas to measure the real update rate on the Pi; scope a relay output against `Sleep` durations in a test sequence.
- **Failure injection:** unplug a LabJack, a sensor loop, the thermocouple, the UART cable and the network link in turn and record what the operator sees.
- **Abort drills:** abort from idle, mid-sleep, mid-sequence, while un-armed, and by inactivity.

### Validating calculations against references

For each sensor, tabulate reference value, raw value (volts / °C / L/s), displayed value and error at ≥ 5 points across the range, both directions; compute maximum error, % of full scale, and repeatability. Keep the table with the test records together with the calibration constants used.
