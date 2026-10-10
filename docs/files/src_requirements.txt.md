# `src/requirements.txt`

Source: [`src/requirements.txt`](../../src/requirements.txt) (3 lines)

```
websockets
asyncio
LabJackPython==2.0.4
```

## Purpose

Python dependency list for the backend, installed with `pip install -r requirements.txt` (on the Pi: `sudo python3.10 -m pip install -r requirements.txt` – the root README stresses `sudo` because the server is started by root from `rc.local`).

## Entries

| Line | Role | Notes |
|---|---|---|
| `websockets` | WebSocket server used by `server.py` | **Unpinned.** The handler signature `event_handler(self, ws, path="None")` tolerates both the legacy API (handler receives `ws, path`) and the newer one (handler receives only the connection). The code also references `websockets.exceptions.ConnectionClosedError` |
| `asyncio` | – | `asyncio` has been part of the standard library since Python 3.4. The PyPI distribution with this name is an old backport; listing it is unnecessary for Python ≥ 3.10 **[Inferred]** and is best removed ([known-issues.md](../known-issues.md) C5) |
| `LabJackPython==2.0.4` | `u3` module for the LabJack U3 | Pinned; `labjack.py` L127 asserts the exact version at run time. Only imported when not in `--dev` mode. Requires the native Exodriver (`liblabjackusb`) built separately (root README) |

## Not listed but required

- Python ≥ 3.10 (`match` statements, `float | None` annotations).
- Exodriver + `libusb-1.0` (real hardware only).
- `apcupsd` providing the `apcaccess` command (optional; absent → UPS status "Unknown").

## Assumptions

No lock file or hash pinning; a future `websockets` release could change behaviour. Recording the versions in use on the Pi (`pip freeze`) is recommended.
