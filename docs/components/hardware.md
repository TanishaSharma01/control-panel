# Hardware Overview

What the software assumes about the physical system. Everything here is derived from code, comments and the root `README.md`; there are **no schematics, wiring diagrams or bills of materials in the repository**. Items that need confirmation are collected in [known-issues.md §D](../known-issues.md#d-information-needed-from-the-hardware-team).

## Computing and network

| Item | Detail | Source |
|---|---|---|
| Server computer | Raspberry Pi (2B or 4B mentioned), Raspberry Pi OS, Python 3.10 built from source | root README |
| Static IP | `192.168.0.5/24`, router `192.168.0.1` | root README |
| Link to control site | Ubiquiti point-to-point Wi-Fi bridge + router; operator laptop joins the router Wi-Fi | root README |
| Ports | 3000 (static frontend), 8888 (WebSocket) | `startup.sh`, `server.py` L647 |
| Power | UPS with USB link to the Pi ("PowerChute USB Port"); monitored by `apcupsd` | root README, `server.py` L181-212 |

## LabJack U3 data-acquisition devices

Two units, one per stand, each opened by serial number (`u3.U3(firstFound=False, serial=…)`, `labjack.py` L130): `LOX.SerialNumber = 2`, `ETH.SerialNumber = 1` (`stands.py` L21, L65).

**[Theory – LabJack U3 datasheet]**

- 16 flexible I/O (FIO0–7, EIO0–7) usable as digital in/out or 12-bit analog inputs, plus 4 dedicated digital lines (CIO0–3). In LabJackPython, DIO numbers 0–19 map to FIO0–7, EIO0–7, CIO0–3.
- Low-voltage analog inputs: single-ended range ≈ 0–2.44 V (a special 0–3.6 V range exists), 12-bit.
- U3-**HV** variant: FIO0–FIO3 are replaced by dedicated high-voltage analog inputs (±10 V) and cannot be digital.
- Hardware SPI master and asynchronous UART (8N1, TTL levels) on hardware version ≥ 1.21; 256-byte UART receive buffer.
- Digital outputs are 3.3 V logic.

How the software uses it:

| Function | LabJackPython call | Code |
|---|---|---|
| Choose analog pins | `configIO(FIOAnalog=mask, EIOAnalog=0)` | `labjack.py` L131-135 |
| Read voltage | `getAIN(pin)` | L204 |
| Read output/input line | `getDIOState(pin)` (does not change pin direction) | L170 |
| Drive line | `setDIOState(pin, state=0/1)` (makes the pin an output) | L180 |
| SPI | `spi([0,0], AutoCS=True, SPIMode='B', …)` | L232-241 |
| UART | `configIO(EnableUART=True, TimerCounterPinOffset=6)`, `asynchConfig`, `asynchRX`, `asynchTX` | L161-162, L305, L353 |

The full pin allocation table is in [sensors/README.md](../sensors/README.md#pin-map-per-labjack-from-srcstandspy).

## Actuators

### Valves (relay-driven solenoids)

| Stand | Valve | DIO | Role (from `pins.json` full names) |
|---|---|---|---|
| ETH | Main | 15 | Main fuel valve |
| ETH | Fill | 16 | Fuel fill |
| ETH | Drain | 17 | Fuel dump/drain |
| ETH | Pressure | 8 | Tank N₂ pressurisation |
| ETH | Vent | 9 | Tank vent (**inverted relay**) |
| ETH | Purge | 14 | Line N₂ purge |
| ETH | Igniter | 10 | Igniter channel (treated as a "valve" in software) |
| LOX | Main | 14 | Main oxidiser valve |
| LOX | Fill | 16 | LOX fill |
| LOX | Drain | 17 | LOX dump/drain |
| LOX | Pressure | 9 | Tank N₂ pressurisation |
| LOX | Vent | 10 | Tank vent (**inverted relay**) |
| LOX | Purge | 8 | Line N₂ purge |
| LOX | Chill | 15 | Line chill-down |

**Relay polarity [Existing]** – `_is_inverted_relay` (`labjack.py` L96-104): for every valve except the vents, relay output high ⇔ valve mechanically open. For the vent valves, relay high ⇔ valve **closed**. `_set_valve_state` and `get_valve_state` hide this, so everywhere else `True` means "mechanically open". **[Inferred]** this corresponds to normally-open vent valves, so a de-energised (power-lost) system vents its tanks.

There is no valve position feedback; the "state" shown is the commanded output read back.

### Light stand and buzzer (`LightStand`, `stands.py` L8-15)

Green = DIO 13, yellow = 12, red = 18, buzzer = 19, same numbers on both U3s. Driven at 1 Hz by `init_state_handler` (`server.py` L508-583):

| Lamp | Meaning |
|---|---|
| Green blinking | No client connected |
| Green solid | ≥ 1 client connected |
| Yellow off | Not armed |
| Yellow solid | Armed, manual control off |
| Yellow blinking | Manual control on |
| Red off | Tank sensor voltage < 0.52 V (≈ 2 bar) |
| Red solid | 0.52 V – 1.47 V |
| Red blinking | > 1.47 V |
| Buzzer | Always off (both rules write `False`) |

Rules are evaluated in order each second; later rules override earlier ones. "Blinking" toggles each second (0.5 Hz).

## Sensors

See [sensors/README.md](../sensors/README.md).

## Power-loss behaviour **[Inferred]**

- The UPS keeps the Pi (and whatever else is plugged into it) alive; the UI shows "🔋 Battery" and a warning is raised on change.
- If the Pi/server stops, nothing drives the LabJack outputs to a safe state; the U3s hold their last output values as long as they are USB-powered.
- If the U3 loses power, its lines revert to their power-up defaults (inputs unless configured otherwise in the device), de-energising relays: non-inverted valves close and the vents open – assuming the relay board is active-high and the valves are spring-return. This must be confirmed on hardware.
