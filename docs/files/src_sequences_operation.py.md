# `src/sequences/operation.py`

Source: [`src/sequences/operation.py`](../../src/sequences/operation.py) (20 lines)

## Purpose

The original "main operation" sequence: pressurise both tanks, flow both propellants for 10 s, then purge and vent. It contains **no igniter command**, so on its own it is a cold-flow (or externally ignited) run. It is also the fallback text the sequence editor inserts when it has nothing to show (`sequence-panel.js` L134-157).

Format and loading rules are the same as for [abort.py](src_sequences_abort.py.md). Loaded with `SETSEQUENCE "operation"`, started with `BEGINSEQUENCE`; both require arming.

## Timeline

| t (s) | Lines | Commands | Effect |
|---|---|---|---|
| 0 | 1-2 | `Close(ETH.Vent)`, `Close(LOX.Vent)` | Seal both tanks |
| 0 → 5 | 3 | `Sleep(seconds=5.0)` | |
| 5 | 4-5 | `Open(ETH.Pressure)`, `Open(LOX.Pressure)` | Pressurise with nitrogen |
| 5 → 20 | 6 | `Sleep(seconds=15.0)` | Tanks come up to pressure |
| 20 | 7-8 | `Open(ETH.Main)`, `Open(LOX.Main)` | Propellant flow (fuel commanded first, same event-loop turn) |
| 20 → 30 | 9 | `Sleep(seconds=10.0)` | 10 s run |
| 30 | 10-11 | `Close(ETH.Main)`, `Close(LOX.Main)` | Stop flow |
| 30 | 12-13 | `Open(ETH.Purge)`, `Open(LOX.Purge)` | Purge lines |
| 30 | 14-15 | `Close(ETH.Pressure)`, `Close(LOX.Pressure)` | Stop pressurising |
| 30 | 16-17 | `Open(ETH.Vent)`, `Open(LOX.Vent)` | Vent tanks |
| 30 → 33 | 18 | `Sleep(seconds=3.0)` | |
| 33 | 19-20 | `Close(ETH.Purge)`, `Close(LOX.Purge)` | End purge |

Total ≈ 33 s. Consecutive `Open`/`Close` commands execute back-to-back with only USB latency between them.

## Notes

- Timing accuracy is that of `Sleep.act`: 5 ms polling on an event loop that is periodically blocked by LabJack polling, so expect jitter of tens of milliseconds.
- Can be paused (the in-flight sleep keeps its remaining time), cleared, or aborted at any point.
- Valves are left: mains closed, pressure closed, vents open, purge closed.
