# WebSocket Protocol Reference

Transport: plain (unencrypted, unauthenticated) WebSocket, `ws://<server-ip>:8888`. Every frame is a UTF-8 JSON text message. Defined by `handle_command`, `emit` and `broadcast` in [`src/server.py`](../src/server.py) and by `connect`/`emit` in [`reactfront/src/index.js`](../reactfront/src/index.js).

## Client → server

Envelope (built in `index.js` `emit`, lines 181-188):

```json
{ "header": "<COMMAND>", "data": <parameter or null>, "time": 1785127770207 }
```

`time` is the client's `Date.getTime()` in epoch ms. `header` and `time` are mandatory (the server indexes `data['header']` and `data['time']`; a missing key raises `KeyError`, which is caught and surfaced as a warning). Header strings are defined in `ClientCommandString` ([`src/commands.py`](../src/commands.py) lines 78-93).

| Header | `data` | Gate | Server effect (`server.py` line) |
|---|---|---|---|
| `PING` | `null` | – | Replies `PING` to the sender only, echoing the client `time` (406) |
| `ARMINGSWITCH` | `true`/`false` | no sequence running | Sets `state.arming_switch` (409) |
| `MANUALSWITCH` | `true`/`false` | no sequence running | Sets `state.manual_switch` (416) |
| `OPEN` | `{"name":"LOX","pin":14}` | armed **and** manual | `Open(name, pin).act(server)` (421) |
| `CLOSE` | `{"name":"ETH","pin":15}` | armed **and** manual | `Close(name, pin).act(server)` (425) |
| `DATALOG` | `true`/`false` | – | Start (new CSV file) / stop server-side logging (429) |
| `SETSEQUENCE` | `"operation"` | armed | Load `sequences/<name>.py` into `current_sequence` (432) |
| `BEGINSEQUENCE` | `null` | armed | `execute_sequence()` – start or resume (440) |
| `ABORTSEQUENCE` | `null` | armed | `run_abort_sequence()` (444) |
| `PAUSESEQUENCE` | `null` | not ABORTING | `status = IDLE`; in-flight sleep keeps its remaining time (448) |
| `UNSETSEQUENCE` | `null` | not ABORTING | Stop and unload the sequence (452) |
| `SAVESEQUENCE` | `{"name":"hfon","commands":["Open(ETH.Main)", …]}` | armed | Write file, reload it as current, reply `SEQUENCE_SAVED` (459) |
| `DELETESEQUENCE` | `"hfon"` | armed | Delete file (never `abort`), reply `SEQUENCE_DELETED` (474) |
| `GETSEQUENCE` | `"hfon"` | – | Reply `SEQUENCE_CONTENT` with the file's lines (484) |
| `CRYOCOMMAND` | `{"stand":"LOX","command":"t"}` | – | Send one byte to the flow-meter MCU over UART TX (488) |
| anything else | – | – | `push_warning("Unknown command: …")` (505) |

A gated command that fails its gate is **silently ignored** (no error reply), except `DELETESEQUENCE`, which warns and replies with `success:false`.

## Server → client

Envelope (`emit`, lines 628-635):

```json
{ "type": "<TYPE>", "time": 1785127770207, "data": … }
```

`time` is server epoch ms (`round(time.time()*1000)`).

| Type | Sent to | `data` |
|---|---|---|
| `STATE` | all clients, ~20 Hz | `SystemState.as_dict()` – see below |
| `PING` | requester | the client's original `time` (round-trip latency = now − data) |
| `VALVE` | all clients | `{"time": ms, "header": "OPEN"\|"CLOSE", "pin": 14, "stand": "LOX"}` – sent for every `Open`/`Close` command **even if the hardware action was blocked by the arming gate** (`commands.py` lines 52-58) |
| `SEQUENCE_SAVED` | requester | `{"name":…, "success":bool}` – *the frontend has no handler; it hits `default:` and is `console.error`'d* |
| `SEQUENCE_DELETED` | requester | `{"name":…, "success":bool}` – ditto |
| `SEQUENCE_CONTENT` | requester | `{"name":…, "content":[lines]\|null}` – ditto; the frontend never sends `GETSEQUENCE` |

### `STATE.data` schema

Produced from the `SystemState` dataclass (`server.py` lines 48-97). Example (dev mode, abridged):

```json
{
  "arming_switch": false,
  "manual_switch": false,
  "current_sequence": [
    {"name": "CLOSE", "stand": "ETH", "pin": 9},
    {"name": "SLEEP", "ms": 5000, "remaining": 5000}
  ],
  "current_sequence_name": "operation",
  "available_sequences": ["abort", "hfoff", "hfon", "operation"],
  "status": 0,
  "command_in_flight": null,
  "data_logging": false,
  "latest_warning": [1785127740356, "UPS status changed: ✅ Line Powered"],
  "debug_log": ["[14:49:30] [CRYO] LOX: polling, 0 bytes in RX buffer"],
  "cryo_test_mode": {"LOX": true},
  "labjacks": {
    "LOX": {
      "digital":     {"14": false, "16": false, "17": false, "9": false, "10": false, "8": false, "15": false},
      "analog":      {"4": 0.5312, "2": 0.4811},
      "light_stand": {"13": true, "12": false, "18": false, "19": false},
      "temperature": 29.5,
      "cryo_flow_lps": 1.0668
    },
    "ETH": {
      "digital":     {"15": false, "16": false, "17": false, "8": false, "9": false, "14": false, "10": false},
      "analog":      {"5": 0.48, "4": 0.52, "6": 0.49, "0": 0.4957},
      "light_stand": {"13": true, "12": false, "18": false, "19": false},
      "temperature": 25.75
    }
  },
  "time": 1785127770207,
  "UPS_status": "LINE_POWERED"
}
```

Field notes:

- **Pin keys are strings.** Python uses integer dict keys; `json.dumps` converts them to strings. This is why the frontend indexes `analog["4"]` (`utils.js` line 227) and `digital[props.labjack_pin]` (JS coerces the number to a string key).
- `digital[pin]` – `true` = valve **mechanically open** (relay inversion for vent valves already removed by `LabJack.get_valve_state`). It is the read-back of the U3 output latch, **not** a position sensor on the valve.
- `analog[pin]` – calibrated **volts** from `u3.getAIN`. No engineering-unit conversion is done server-side.
- `temperature` – raw MAX6675 °C (median of up to 3 reads, refreshed ≈ 1 Hz), or `null` (open thermocouple, SPI error, or not yet read).
- `cryo_flow_lps` – last successfully parsed L/s value from the flow MCU, or `null` if nothing has ever been parsed. Present for LOX only.
- `status` – `0` IDLE, `1` RUNNING, `2` ABORT_REQUESTED, `3` ABORTING.
- `current_sequence` – only the **remaining** commands; `command_in_flight` is the one executing. `SLEEP.remaining` counts down while in flight.
- `latest_warning` – `[epoch_ms, message]` or `null`; latched until replaced.
- `time` – epoch ms at which the `labjacks` field was refreshed. The frontend converts it to seconds and uses it as *the* time axis.
- `UPS_status` – `LINE_POWERED`, `BATTERY_POWERED` or `UNKNOWN`.

## Sequence file command grammar

Each non-empty line of `src/sequences/<name>.py` is a Python expression evaluated with `Sleep`, `Open`, `Close`, `LOX`, `ETH` in scope (`load_sequence`, `server.py` lines 232-239):

```
Open(LOX.Main)          # Open(('LOX', 14))
Close(ETH.Vent)
Sleep(seconds=3.0)
```

Blank lines and `#` comment lines are **not** supported – each line is spliced into `_cursed_global = <line>` and `exec`'d, so an empty or comment-only right-hand side is a `SyntaxError` and the whole load fails with the warning "Could not load sequence …".
