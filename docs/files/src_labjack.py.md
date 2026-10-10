# `src/labjack.py`

Source: [`src/labjack.py`](../../src/labjack.py) (422 lines)

## 1. Purpose and responsibility

Hardware abstraction layer. It hides LabJackPython behind a small interface, implements the three sensor drivers (analog voltage, MAX6675 over SPI, flow meter over UART), abstracts relay polarity, and supplies a hardware-free fake for development.

## 2. Imports (L1-8)

| Import | Use |
|---|---|
| `abc.ABCMeta, abstractmethod` | Abstract base class |
| `traceback.print_exc, format_exc` | Error reporting (terminal / debug log) |
| `sys` | `sys.exit()` on fatal read errors |
| `time`, `math`, `random` | Sleeps, timestamps, fake signals |
| `threading` | Thermocouple worker thread and locks |
| `stands.LOX, ETH` | Vent pin numbers for relay inversion |
| `LabJackPython`, `u3` | Imported **inside** `LabJack.__init__` (L125, L128) so dev mode needs neither |

## 3. Components

### `get_class(dev: bool) -> type` (L10-11)
Returns `LabJackFake` if `dev` else `LabJack`.

### `LabJackBase` (L13-104)

Abstract methods: `_set_digital_state`, `_set_valve_state`, `get_valve_state`, `_get_digital_state`, `get_voltage`, `get_thermocouple_temp`, `get_cryo_flow_lps`, `send_cryo_command`.

Concrete methods:

| Method | Lines | Behaviour |
|---|---|---|
| `open_valve(pin)` / `close_valve(pin)` | 50-54 | `set_valve_state(pin, True/False)` |
| `set_valve_state(pin, open)` | 56-69 | Calls `_set_valve_state`. A block that would have closed Pressure before opening Vent (and vice versa) is **commented out**, so no mutual exclusion is enforced |
| `get_state()` | 71-94 | Builds the snapshot: `digital` (per valve pin, `get_valve_state`), `analog` (per sensor pin, `get_voltage`), `light_stand` (per lamp pin, `_get_digital_state`), `temperature` if the config has `Thermocouple`, `cryo_flow_lps` if it has `CryoFlowUART`. Each section is omitted if its pin list is empty |
| `_is_inverted_relay(pin)` | 96-104 | `True` only for the stand's `Vent` pin (ETH 9 / LOX 10). The comment's "ETH 19, LOX 13" is stale |

Expected instance attributes (set by subclasses): `config`, `digital_pins`, `analog_inputs`, `light_stand_pins`.

### `LabJack` (L106-358) – real hardware

**`__init__(standConfig, log_callback=print)` (L114-165)**

1. Copy pin lists from the config; keep `log_callback` as `self._log`.
2. Import LabJackPython, assert version `2.0.4`, import `u3`.
3. `self.device = u3.U3(firstFound=False, serial=standConfig.SerialNumber)`.
4. Analog mask: `x |= 1 << pin` for each sensor pin; `configIO(FIOAnalog=x, EIOAnalog=0)`. Only FIO pins (0–7) can therefore be analog; an analog sensor on an EIO pin would silently not be configured.
5. Initialise caches: `_tc_cache`, `_cryo_cache`, `_cryo_line_buffer`, `_cryo_last_debug_print`.
6. Create `_device_lock` (guards all `self.device.*` calls) and `_tc_lock` (guards `_tc_cache`).
7. If `CryoFlowUART` present: assert `rx == tx + 1`; `configIO(EnableUART=True, TimerCounterPinOffset=tx)`; `asynchConfig(UARTEnable=True, DesiredBaud=baud, configurePins=False)`; log the result.
8. If `Thermocouple` present: start the daemon thread `_thermocouple_worker`.

Note: valve and lamp pins are not written during initialisation.

**Digital I/O**

| Method | Lines | Notes |
|---|---|---|
| `_get_digital_state(pin)` | 167-175 | `getDIOState` under the lock. On *any* exception: prints the pin and traceback, `sys.exit()` |
| `_set_digital_state(pin, state)` | 177-180 | `setDIOState(pin, state=int(state))` under the lock; exceptions propagate |
| `_set_valve_state(pin, open)` | 182-194 | Inverts `open` for vent pins, then `_set_digital_state`. Docstring: success and timing are not guaranteed |
| `get_valve_state(pin)` | 196-199 | `_get_digital_state`, inverted for vent pins → `True` = mechanically open |

**Analog** – `get_voltage(pin)` (L201-209): `getAIN(pin)` under the lock; any exception → print + `sys.exit()`.

**Thermocouple** – `_read_thermocouple_once` (L211-250), `_thermocouple_worker` (L252-277), `get_thermocouple_temp` (L279-289). Full walk-through, bit layout and timing analysis: [sensors/max6675-common.md](../sensors/max6675-common.md#b-data-acquisition).

**Cryo flow** – `get_cryo_flow_lps` (L291-339), `send_cryo_command` (L341-358). Full walk-through: [sensors/lox-cryo-flow.md](../sensors/lox-cryo-flow.md#b-data-acquisition-and-communication).

### `LabJackFake` (L360-422) – simulation

| Member | Lines | Behaviour |
|---|---|---|
| `__init__` | 366-378 | Same attributes; `self.state = {"digital":{}, "analog":{}, "light_stand":{}}`; `_tc_last_read` (unused) |
| `_set_default_state(pin)` | 380-382 | Initial valve state `True` only for pins 13 and 19 (stale "vent" numbers) → in practice every valve starts closed |
| `_set_digital_state` / `_get_digital_state` | 384-395 | Store/read `state["light_stand"]`; unknown pins lazily default to `False` |
| `_set_valve_state` / `get_valve_state` | 387-402 | Store/read `state["digital"]`; **no relay inversion modelled** |
| `get_voltage(pin)` | 404-408 | `0.4·sin(t + pin + serial) + 2·sin(t/10 + pin + 7) + 1 + U(−0.1, 0.1) + spike`, spike = 1 V with probability 0.005 |
| `get_thermocouple_temp()` | 410-411 | `round(25 + 5·sin(0.1t + serial), 2)` |
| `get_cryo_flow_lps()` | 413-416 | `round(0.9 + 0.5·sin(t/5 + serial), 4)` if the config has `CryoFlowUART`, else `None` |
| `send_cryo_command(char)` | 418-422 | Logs "(fake) would send command", returns `True` (or `False` without UART config) |

## 4. Execution flow

- Constructed twice by `ControlPanelServer.__init__`.
- `get_state()` called from `server.sync_state` each tick.
- `open_valve`/`close_valve` called from `commands.Open/Close.act`.
- `_set_digital_state`, `get_voltage` called from `server.init_state_handler` (light stand).
- `send_cryo_command` called from `server.handle_command` (`CRYOCOMMAND`).
- `_thermocouple_worker` runs autonomously on its thread.

## 5. Inputs and outputs

| Method | In | Out |
|---|---|---|
| `get_state` | – | `{"digital":{int:bool}, "analog":{int:float V}, "light_stand":{int:bool}, "temperature": float\|None, "cryo_flow_lps": float\|None}` |
| `get_voltage` | DIO/AIN number | volts |
| `set_valve_state` | pin, mechanical open flag | side effect on the relay line |
| `send_cryo_command` | single character | `bool` success |

Example `get_state()` for LOX in dev mode:

```python
{'digital': {14: False, 16: False, 17: False, 9: False, 10: False, 8: False, 15: False},
 'analog': {4: 0.91, 2: 2.13},
 'light_stand': {13: True, 12: False, 18: False, 19: False},
 'temperature': 29.6,
 'cryo_flow_lps': 1.0668}
```

## 6. Error handling summary

| Operation | On failure |
|---|---|
| Open device / configIO / asynchConfig | Exception → caught in `server.py` → exit |
| `getDIOState`, `getAIN` | `sys.exit()` |
| `setDIOState` | Propagates to caller |
| SPI read | Caught → `None` for that read |
| Worker loop body | Caught, printed, loop continues |
| `asynchRX` | Caught, logged, cached value returned |
| `asynchTX` | Caught, logged, returns `False` |

## 7. Configuration and assumptions

- LabJackPython exactly 2.0.4.
- `standConfig` provides `name`, `SerialNumber`, `Valves`, `Sensors`, `LightStand.LightStand`, `Vent`, optionally `CryoFlowUART` and `Thermocouple`.
- Hard-coded: SPI mode `'B'`, clock factor 0, 3 reads × 5 ms, 1 s cycle; UART chunk 32 bytes, line buffer cap 500 chars, heartbeat 2 s, 4-field CSV.
- The U3 handle is not thread-safe, hence `_device_lock`.

## 8. Related documentation

[components/hardware.md](../components/hardware.md) · [sensors/README.md](../sensors/README.md) · [known-issues.md](../known-issues.md) (A2, A10–A13, B2, B3, B8, C1, C2)
