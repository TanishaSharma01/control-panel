# `src/server.py`

Source: [`src/server.py`](../../src/server.py) (649 lines)

## 1. Purpose and responsibility

The backend's entry point and orchestrator. It owns the authoritative system state, accepts WebSocket clients, polls both LabJacks and broadcasts their state ~20 times a second, enforces the arming/manual interlocks, runs/pauses/aborts sequences, manages sequence files, drives the light stand, monitors the UPS, and optionally logs everything to CSV.

## 2. Dependencies and imports (L1-13)

| Import | Why |
|---|---|
| `dataclasses.dataclass, field, asdict` | `SystemState` definition and dict conversion |
| `websockets` | `websockets.serve`, `websockets.exceptions.ConnectionClosedError` |
| `asyncio` | Event loop, `ensure_future`, `sleep`, `Future` |
| `time`, `json`, `sys` | Timestamps, wire format, `--dev` flag |
| `datalog.Datalog` | CSV logging |
| `traceback.print_exc` | Diagnostics |
| `labjack.get_class` | Real vs fake LabJack class |
| `stands.ETH, LOX` | Pin maps (constructor arguments, sequence namespace, light-stand logic) |
| `pathlib.Path` | Sequence and log paths relative to this file |
| `utils` | `get_local_ip`, `time_ms`, `get_output`, `exec_expr_with_locals` |
| `commands.Sleep, Open, Close, ServerCommand, ClientCommandString` | Command objects and protocol constants |

## 3. Module-level definitions

| Name | Line | Value / meaning |
|---|---|---|
| `devMode` | 18 | `'--dev' in sys.argv` → simulated LabJacks and simulated UPS |
| `STATE_BROADCAST_FREQUENCY` | 20 | `20` – the sleep between state loops is `1/20` s |
| `ABORT_SEQUENCE_TIMEOUT` | 21 | `900` s with no clients before the automatic abort |
| `LOG_PATH` | 23 | `src/logs` |
| `UPSStatus` | 25-29 | String constants `LINE_POWERED`, `BATTERY_POWERED`, `UNKNOWN` |
| `UPSStatusMessages` | 31-35 | Human-readable text for warnings |
| `SequenceStatus` | 37-46 | `IDLE=0`, `RUNNING=1`, `ABORT_REQUESTED=2`, `ABORTING=3` |

### `SystemState` (L48-97)

Dataclass with defaults; every field is sent to clients. Fields: `arming_switch`, `manual_switch`, `current_sequence` (remaining commands), `current_sequence_name`, `available_sequences`, `status`, `command_in_flight`, `data_logging`, `latest_warning` `(epoch_ms, msg)`, `debug_log` (≤ 50 lines), `cryo_test_mode` `{stand: bool}`, `labjacks` `{stand: get_state() dict}`, `time` (ms), `UPS_status`.

`as_dict()` (L90-97): `ServerCommand` objects are not dataclasses and not JSON-serialisable, so the method temporarily sets `command_in_flight` and `current_sequence` to `None`, calls `asdict(self)` (a deep copy of everything else), then substitutes `c.as_dict()` / `[x.as_dict() for x in s]` and restores the attributes. This is safe only because everything runs on one event-loop thread with no `await` in between.

## 4. `ControlPanelServer` – block-by-block

### Construction (L102-128)

Creates state, `datalog = None`, `time_of_last_disconnect = now`, `inactive_abort_fired = False`; instantiates `self.labjacks['LOX']` and `['ETH']` with `log_callback=self.log_debug` so `[CRYO]` diagnostics reach the browser. A bare `except` prints the traceback and a hint, then `exit(0)` (note: exit status 0 even though it failed). Stores `ip`, `port`, an empty `clients` set, and scans the sequence directory.

### `refresh_available_sequences` (L130-136)
Creates `src/sequences` if missing and stores the sorted stems of `*.py` in `state.available_sequences`.

### Periodic coroutines

| Coroutine | Lines | Period | Behaviour |
|---|---|---|---|
| `timeout_counter` | 138-150 | 1 s | If `now − time_of_last_disconnect > 900`, no clients, and not already fired → set flag, `run_abort_sequence()` |
| `sync_state` | 152-162 | 50 ms + work | (1) if logging, write the *current* (previous-tick) state as a `STATE` row; (2) `state.labjacks[stand] = labjack.get_state()` for each stand; (3) `state.time = ms`; (4) schedule `broadcast('STATE', state.as_dict())` |
| `update_UPS_status` | 206-212 | 1 s | Calls `get_UPS_status`; on change, pushes a warning |
| `init_state_handler` | 508-583 | 1 s | Light-stand rule table (below) |

### Logging helpers (L164-179)

- `log_data(data, type)` – forwards to `Datalog.log_data` only if logging is enabled.
- `push_warning(msg)` – sets `state.latest_warning = (time_ms, msg)`; delivered with the next `STATE`.
- `log_debug(msg)` – `print` + append `"[HH:MM:SS] msg"` to `state.debug_log`, trimmed to the last 50.

### UPS (L181-204)

Real mode: `await utils.get_output('apcaccess status')`. Dev mode: cycles `ONBATT → ONLINE → UNKNOWN` every minute using `echo` and prints the result each second. Classification by substring: `"ONBATT"` → battery, else `"ONLINE"` → line, else unknown (which is also the result if `apcaccess` is missing, since stderr is discarded and stdout is empty).

### `event_handler(ws, path="None")` (L214-230)

One invocation per client connection. Adds `ws` to `clients`; for each message: reset `inactive_abort_fired`, log `REQUEST`, `json.loads`, `handle_command(ws, header, data, time)`. Any exception → traceback + `push_warning(str(e))`, loop continues. `IncompleteReadError` / `ConnectionClosedError` end the loop quietly. `finally`: remove client, stamp `time_of_last_disconnect`.

### Sequence loading and abort (L232-258)

- `load_sequence(name)` – opens `sequences/<name>.py` and evaluates every line with `utils.exec_expr_with_locals(line, Sleep=…, Open=…, Close=…, LOX=LOX, ETH=ETH)`, returning the list of resulting command objects. Raises on any missing file or bad line.
- `run_abort_sequence()` – logs `ABORTING`, **disarms** (`arming_switch = manual_switch = False`), clears `command_in_flight`, then by status:
  - `ABORTING` / `ABORT_REQUESTED`: do nothing (already aborting);
  - `IDLE`: load `abort`, `execute_sequence(initial_state=ABORTING)`;
  - `RUNNING`: set `ABORT_REQUESTED` and swap `current_sequence` for the abort list – the running task picks it up.

### Sequence file management (L262-392)

- `save_sequence(name, commands)` – requires `name.isalnum()`; refuses to overwrite `abort` unless armed (the caller already requires arming, so this check is redundant); writes each stripped non-empty command on its own line; logs `SEQUENCE_EDIT`; returns `True/False`. **Commands are not validated** (comment L288-289).
- `delete_sequence(name)` – `isalnum` check; `abort` can never be deleted; unlinks the file; unloads it if it was the current sequence.
- `get_sequence_content(name)` – returns stripped non-empty lines or `None`. (No name validation.)

### `set_datalogging_enabled` (L394-402)
On: set flag, create a new `Datalog(LOG_PATH)` (new timestamped file), log `DATA_LOGGING True`. Off: log `False`, clear flag, drop the object.

### `handle_command(ws, header, data, time)` (L404-506)

A `match` on the header; table of every branch and its gate in [websocket-protocol.md](../websocket-protocol.md#client--server). Noteworthy details:

- `ARMINGSWITCH` also `print`s the value (L413).
- `SETSEQUENCE` wraps the load in a bare `except` → warning "Could not load sequence …".
- `SAVESEQUENCE` reloads the saved file as the current sequence so all clients see it, refreshes the list, and replies.
- `CRYOCOMMAND` validates the stand has `CryoFlowUART`, sends the byte, toggles `cryo_test_mode[stand]` when `'t'` succeeded.

### `init_state_handler` – light stand (L508-583)

Builds, for each LabJack `x`, a list of `{condition, effect}` lambdas (`shared(x)`), concatenates LOX's and ETH's lists, then every second runs every effect whose condition is true and toggles the closure variable `cycle` (used for blinking). Rules and thresholds: [components/hardware.md](../components/hardware.md#light-stand-and-buzzer-lightstand-standspy-l8-15). The tank-pressure rules call `x.get_voltage(...)` up to five times per LabJack per second – extra blocking USB reads on the event loop.

### `start_server` (L585-591)
Schedules the four coroutines with `ensure_future` and enters `websockets.serve(self.event_handler, ip, port)`, awaiting a never-completed `Future`.

### `execute_sequence(initial_state=RUNNING)` (L593-615)

1. If a command is in flight (resuming from pause) push it back to the front of the list.
2. Start `task()`: set `status`, force `manual_switch = False`; loop while commands remain and status is not `IDLE`: pop → `command_in_flight` → `await act(self)` → clear `command_in_flight` if it reported completion → if status became `ABORT_REQUESTED`, change it to `ABORTING`. Finally `status = IDLE`.

### Predicates (L617-626)

| Method | True when |
|---|---|
| `should_interrupt_sleep()` | status is `ABORT_REQUESTED` or `IDLE` (pause/unset) |
| `sequence_running()` | status is `RUNNING`, `ABORTING` or `ABORT_REQUESTED` |
| `sequence_command_allowed()` | armed, or status is `ABORTING` |

### `emit` / `broadcast` (L628-643)
`emit` wraps data as `{"type", "time", "data"}` and sends to one socket. `broadcast` emits to each client sequentially, swallowing all exceptions (a slow client delays the others for that message).

### `__main__` (L645-649)
`ip = utils.get_local_ip()`, `port = 8888`, construct, `asyncio.run(server.start_server())`.

## 5. Execution flow

`python server.py [--dev]` → module constants → `ControlPanelServer(ip, 8888)` → `asyncio.run(start_server())` → background coroutines + one `event_handler` per client → runs until killed. See [architecture.md](../architecture.md).

## 6. Inputs and outputs

| Direction | What |
|---|---|
| In | CLI flag `--dev`; WebSocket JSON commands; LabJack readings; `apcaccess` output; sequence files |
| Out | WebSocket `STATE`/`PING`/`VALVE`/`SEQUENCE_*`; LabJack digital outputs and UART bytes; `logs/*.csv`; sequence files; stdout |

## 7. Relationships

Imports `labjack`, `stands`, `commands`, `datalog`, `utils`. Is called back by `commands.py` (`server.labjacks`, `server.broadcast`, `server.should_interrupt_sleep`, `server.sequence_command_allowed`) and by `labjack.py` through the `log_callback`. Its protocol is consumed by `reactfront/src/index.js` and the panels.

## 8. Error handling and edge cases

- Client JSON / command errors → caught, warning.
- LabJack open failure → message, `exit(0)`.
- `broadcast` failures → ignored.
- Exceptions inside background coroutines started with `ensure_future` are **not** observed; e.g. an exception in `sync_state` would silently stop all state updates ("Task exception was never retrieved" at exit).
- No client → no broadcast work, but acquisition and logging continue.
- See [known-issues.md §B](../known-issues.md#b-control--safety-logic) for B1–B11.

## 9. Configuration and assumptions

Hard-coded: port 8888, 20 Hz, 900 s timeout, light-stand thresholds 0.52 V / 1.47 V, debug-log length 50, `apcaccess` command name, sequence directory `src/sequences`, log directory `src/logs`. Assumes working directory `src/`, Python ≥ 3.10, a network route (for `get_local_ip`), and both LabJacks present before start.

## 10. Examples

Manual open while armed + manual:

```
→ {"header":"OPEN","data":{"name":"ETH","pin":15},"time":1785127770100}
   Open('ETH',15).act → labjacks['ETH'].open_valve(15) → setDIOState(15,1)
← {"type":"VALVE","time":1785127770104,"data":{"time":1785127770104,"header":"OPEN","pin":15,"stand":"ETH"}}
← {"type":"STATE", … "labjacks":{"ETH":{"digital":{"15":true, …}}}}
```

Pause/resume of `Sleep(seconds=10)` after 4 s: `PAUSESEQUENCE` → status `IDLE` → sleep returns `False` with `ms ≈ 6000` → `command_in_flight` kept → `BEGINSEQUENCE` → reinserted → sleeps the remaining ≈ 6 s.
