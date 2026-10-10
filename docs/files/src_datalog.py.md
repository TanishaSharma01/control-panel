# `src/datalog.py`

Source: [`src/datalog.py`](../../src/datalog.py) (25 lines)

## 1. Purpose

A minimal append-only CSV logger used for server-side data logging.

## 2. Imports

`time` (timestamps, file name), `csv` (writer), `os` (path join), `pathlib` (directory creation).

## 3. `class Datalog`

### `__init__(self, path)` (L7-18)

1. Create the directory (`mkdir(parents=True, exist_ok=True)`).
2. `begin_time = time.time()`.
3. File name `time.strftime("%Y%m%d-%H%M%S") + ".csv"` (local time), e.g. `20260727-144930.csv`.
4. Print `Datalogging to <path>`, open the file for writing (truncating), create a `csv.writer`.
5. Write the header row `TIME,CODE,DATA` and flush.

### `log_data(self, data, code="MISC")` (L20-24)

Writes `[time.time() − begin_time, code, str(data).replace('\n', '')]` and flushes.

- `TIME` is seconds since the logger was created (float), **not** wall-clock time; the absolute start time is only in the file name (to 1 s).
- `DATA` is the Python `str()` of the object. For state dictionaries that is a Python literal (`True`, `None`, single quotes, tuples) – parse it with `ast.literal_eval`, not `json.loads`.

## 4. Execution flow

Created by `ControlPanelServer.set_datalogging_enabled(True)` (a new file each time logging is switched on); called through `ControlPanelServer.log_data`, which also checks the `data_logging` flag. Discarded (not closed) when logging is switched off.

## 5. Record types written by the server

| `CODE` | `DATA` | Written from (`server.py`) |
|---|---|---|
| `DATA_LOGGING` | `True` / `False` | L398, L400 |
| `STATE` | `SystemState.as_dict()` – ~20 rows/s | L155 |
| `REQUEST` | Raw JSON text of every client message, **including the 5-per-second PINGs from each client** | L219 |
| `ARMING_SWITCH`, `MANUAL_SWITCH` | new value | L411, L418 |
| `ABORTING` | `True` | L242 |
| `SEQUENCE_EDIT` | `{'sequence': name, 'action': 'saved'\|'deleted'}` | L311, L348 |

Example rows:

```
TIME,CODE,DATA
0.0013380050659179688,DATA_LOGGING,True
0.00645899772644043,STATE,"{'arming_switch': False, … 'labjacks': {'LOX': {'digital': {14: False, …}, 'analog': {4: -0.69, …}, 'temperature': 29.6, 'cryo_flow_lps': 1.0668}, …}, 'time': 1785127770207, 'UPS_status': 'LINE_POWERED'}"
```

## 6. Error handling and limits

- No exception handling: a full disk or permission error propagates into the caller (`sync_state` or `event_handler`).
- No rotation or size limit. At ~20 rows/s of ≥ 1 kB (more with the embedded 50-line `debug_log`), expect several MB per minute.
- The file handle is never closed explicitly; per-row flushing protects against data loss on a crash (but not OS-level power loss before the page cache is written).
- Sensor values in the log are **volts / raw °C / L/s** – no calibration applied.

## 7. Reading a log (example)

```python
import csv, ast
with open('src/logs/20260727-144930.csv') as f:
    for t, code, data in list(csv.reader(f))[1:]:
        if code == 'STATE':
            s = ast.literal_eval(data)
            print(float(t), s['labjacks']['LOX']['analog'].get(4))
```
