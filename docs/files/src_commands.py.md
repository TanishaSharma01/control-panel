# `src/commands.py`

Source: [`src/commands.py`](../../src/commands.py) (93 lines)

## 1. Purpose

Defines (a) the three **server commands** that make up sequences and manual actions – `Sleep`, `Open`, `Close` – and (b) `ClientCommandString`, the list of WebSocket header names the server understands.

## 2. Imports

`abc.ABCMeta, abstractmethod` (abstract base), `stands.Stand` (type hint only), `asyncio` (sleep, `ensure_future`), `utils` (`time_ms`).

## 3. Definitions

### `ServerCommand` (L11-20)
Abstract base with a `name` string, `async act(server) -> bool` (returns whether the command finished) and `as_dict() -> dict` (JSON form; must contain `"name"`). `__str__` returns the name.

### `Sleep(seconds)` (L22-44)

| Attribute | Meaning |
|---|---|
| `name` | `"SLEEP"` |
| `ms` | Remaining duration to sleep (initially `int(seconds * 1000)`) |
| `remaining` | Live countdown, updated while running |

`act(server)`:

1. If `ms <= 0` return (implicitly `None`).
2. `expiry = time_ms() + ms`.
3. Loop while `expiry − now >= 0`: store `remaining`; if `server.should_interrupt_sleep()` (pause, unset, or abort requested) break; `await asyncio.sleep(0.005)`.
4. `self.ms = max(0, expiry − now)`; return `self.ms <= 0`.

So an interrupted sleep returns `False` and remembers how long is left; calling `act` again resumes it. Granularity is 5 ms (plus event-loop latency – note the loop is blocked during each 20 Hz LabJack poll, so wake-ups can be late by the poll duration).

`as_dict()` → `{"name": "SLEEP", "ms": …, "remaining": …}`.

### `Open(stand, pin=None)` / `Close(stand, pin=None)` (L46-76)

Constructor accepts either `Open('LOX', 14)` (manual commands from `handle_command`) or a tuple `Open(LOX.Main)` (sequence files). `name` is `"OPEN"` / `"CLOSE"`.

`act(server)`:

1. If `server.sequence_command_allowed()` (armed, or aborting): `server.labjacks[self.stand].open_valve(self.pin)` / `close_valve`.
2. **Regardless of step 1**, schedule `server.broadcast('VALVE', {'time', 'header': 'OPEN'|'CLOSE', 'pin', 'stand'})`.
3. Return `True`.

`as_dict()` → `{"name", "stand", "pin"}`.

### `ClientCommandString` (L78-93)
Constants: `PING`, `ARMINGSWITCH`, `MANUALSWITCH`, `OPEN`, `CLOSE`, `DATALOG`, `SETSEQUENCE`, `BEGINSEQUENCE`, `ABORTSEQUENCE`, `PAUSESEQUENCE`, `UNSETSEQUENCE`, `GETSEQUENCE`, `SAVESEQUENCE`, `DELETESEQUENCE`, `CRYOCOMMAND`. Semantics: [websocket-protocol.md](../websocket-protocol.md).

## 4. Execution flow

- Sequence files: each line constructs one command via `utils.exec_expr_with_locals` in `server.load_sequence`.
- Manual control: `handle_command` builds `Open`/`Close` directly and awaits `act`.
- `server.execute_sequence` awaits `act` on each popped command.

## 5. Inputs / outputs / side effects

Inputs: constructor arguments and the `server` object. Side effects: relay state changes, `VALVE` broadcasts, time passing. Output to clients: `as_dict()` inside `STATE.current_sequence` / `command_in_flight`.

## 6. Relationships

Imported by `server.py`. Calls back into `ControlPanelServer` (`labjacks`, `broadcast`, `should_interrupt_sleep`, `sequence_command_allowed`). The frontend parses the dict forms in `sequence-panel.js` (`SequenceRow`, `formatCommandToString`) and `control-panel.js` (`describeCommand`).

## 7. Edge cases

- An unknown stand name raises `KeyError` in `act`; an invalid pin raises inside LabJackPython.
- `Sleep(seconds=0)` does nothing and reports not-complete (`None`).
- A negative or non-numeric `seconds` fails at construction (`int(seconds*1000)`) or is treated as ≤ 0.
- The gate is evaluated at execution time, so a sequence started while armed stops actuating if arming is somehow lost (but arming cannot be changed while a sequence runs).
- `VALVE` events are emitted even when the action was blocked ([known-issues.md](../known-issues.md) B7).

## 8. Example

```python
Open(LOX.Main).as_dict()      # {'name': 'OPEN', 'stand': 'LOX', 'pin': 14}
Sleep(seconds=3.0).as_dict()  # {'name': 'SLEEP', 'ms': 3000, 'remaining': 3000}
```
