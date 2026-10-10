# `src/sequences/abort.py`

Source: [`src/sequences/abort.py`](../../src/sequences/abort.py) (14 lines)

## Purpose

The safing sequence. It is the only sequence the server loads by itself: `run_abort_sequence()` (`server.py` L241-258) loads it when the operator presses **ABORT** (while armed) or after 900 s without any connected client. It cannot be deleted through the UI (`delete_sequence` refuses, `server.py` L336-339) but it *can* be overwritten by an armed client.

## Format

Not a real Python module. Each line is one expression evaluated by `server.load_sequence` with `Open`, `Close`, `Sleep`, `LOX`, `ETH` in scope (see [src_utils.py.md](src_utils.py.md)). Blank lines and comments are not allowed.

## Contents, line by line

| Line | Command | Effect |
|---|---|---|
| 1 | `Close(ETH.Igniter)` | Igniter channel off (ETH pin 10) |
| 2-3 | `Close(LOX.Main)`, `Close(ETH.Main)` | Stop propellant flow to the engine |
| 4-5 | `Close(LOX.Fill)`, `Close(ETH.Fill)` | Isolate fill lines |
| 6-7 | `Close(LOX.Drain)`, `Close(ETH.Drain)` | Close drains |
| 8-9 | `Open(LOX.Purge)`, `Open(ETH.Purge)` | Start nitrogen purge of both lines |
| 10 | `Sleep(seconds=3.000)` | Purge for 3 s (interruptible only by pause/unset – and those are refused while `ABORTING`) |
| 11-12 | `Close(LOX.Purge)`, `Close(ETH.Purge)` | End purge |
| 13-14 | `Close(LOX.Pressure)`, `Close(ETH.Pressure)` | Stop tank pressurisation |

Total duration ≈ 3 s.

## What it does *not* do **[Existing]**

- It does not touch the **vent** valves (`LOX.Vent`, `ETH.Vent`): tanks remain pressurised or vented exactly as they were.
- It does not touch `LOX.Chill`.
- Pressurisation stays open during the 3 s purge and is closed last.

Whether these are deliberate choices should be confirmed with the propulsion team.

## Execution details

- Runs with `status = ABORTING`, which lets `Open/Close.act` drive hardware even though `run_abort_sequence` has just disarmed the system (`sequence_command_allowed`, `server.py` L625).
- Each command broadcasts a `VALVE` event, so the abort is visible as markers on the graphs.
- If this file is missing or has a syntax error, `load_sequence('abort')` raises and **no abort happens** ([known-issues.md](../known-issues.md) B9).

## Maintenance rule

When a new valve is added to `stands.py`, decide its safe state and add the corresponding line here.
