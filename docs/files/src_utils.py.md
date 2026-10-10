# `src/utils.py`

Source: [`src/utils.py`](../../src/utils.py) (47 lines)

## 1. Purpose

Four small helpers used by the server.

## 2. Imports

`time`, `asyncio`; `socket` is imported inside `get_local_ip`.

## 3. Functions

### `get_local_ip() -> str` (L5-12)

Creates a UDP socket, `connect(("8.8.8.8", 80))`, reads `getsockname()[0]`, closes it. Connecting a UDP socket sends no packets; it just makes the OS choose the outbound interface, whose address is returned. Used once, in `server.py` `__main__`, to decide which address the WebSocket server binds to and prints.

- **Failure mode:** with no default route (e.g. laptop with Wi-Fi off) `connect` raises `OSError: Network is unreachable` and the server will not start. On the Pi, the static `192.168.0.5` with a configured router address satisfies this.
- The server binds to that one address only, so `ws://localhost:8888` does **not** work, even in dev mode; use the printed IP.

### `time_ms() -> int` (L14-15)
`time.time_ns() // 1_000_000` – epoch milliseconds. Used for warnings, `VALVE` events and `Sleep` timing.

### `async get_output(cmd) -> str` (L17-29)
Runs a shell command with `asyncio.create_subprocess_shell`, captures stdout, discards stderr, returns decoded text (empty string if the command is missing or only errors). Used for `apcaccess status` (and `echo …` in dev mode).

### `exec_expr_with_locals(expr: str, **locals) -> object` (L31-47)

Evaluates a string as a Python expression with extra names in scope, returning the value. Implementation:

1. Inner `exec_expr(code)` runs `exec('global _cursed_global; _cursed_global = ' + code)` and returns the module global `_cursed_global`.
2. `with_context(**kw)` is a decorator that injects `kw` into the function's globals – i.e. into **this module's global namespace** (`fn.__globals__.update(kw)`).
3. The call `with_context(**locals)(exec_expr)(expr)` therefore adds `Sleep`, `Open`, `Close`, `LOX`, `ETH` to `utils`' globals and assigns the evaluated expression to `_cursed_global`.

It is functionally equivalent to `eval(expr, {...})`, written this way to use `exec`.

Used by `server.load_sequence`, once per line of a sequence file.

## 4. Edge cases and security

- An empty string or comment line is a `SyntaxError`.
- Because it is `exec` on an *assignment statement*, a line can contain arbitrary Python after a `;` (e.g. `Sleep(1); import os`). Sequence files are trusted input, but they can be written over the network by any armed client via `SAVESEQUENCE` – see [known-issues.md](../known-issues.md) B4.
- Injected names persist in module globals between calls (harmless here).

## 5. Example

```python
>>> from commands import Open; from stands import LOX
>>> c = exec_expr_with_locals("Open(LOX.Main)", Open=Open, LOX=LOX)
>>> c.as_dict()
{'name': 'OPEN', 'stand': 'LOX', 'pin': 14}
```
