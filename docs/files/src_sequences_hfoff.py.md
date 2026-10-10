# `src/sequences/hfoff.py`

Source: [`src/sequences/hfoff.py`](../../src/sequences/hfoff.py) (19 lines)

## Purpose

"Hot fire off" – added with `hfon.py` in commit `f9b1e3c`. It performs, for LOX and then (five minutes later) for ETH, a short pressurised blow-down through the main and drain valves. **[Inferred]** it is a post-test residual-propellant dump; the repository contains no description of its intent.

Format and loading rules: see [abort.py](src_sequences_abort.py.md).

## Timeline

| t (s) | Lines | Commands | Effect |
|---|---|---|---|
| 0 | 1 | `Close(LOX.Vent)` | Seal LOX tank |
| 0 | 2-4 | `Open(LOX.Pressure)`, `Open(LOX.Main)`, `Open(LOX.Drain)` | Push LOX out through main and drain |
| 0 → 5 | 5 | `Sleep(seconds=5.0)` | |
| 5 | 6-8 | `Close(LOX.Pressure)`, `Close(LOX.Main)`, `Close(LOX.Drain)` | |
| 5 | 9 | `Open(LOX.Vent)` | Vent LOX tank |
| 5 → 305 | 10 | `Sleep(seconds=300.0)` | 5-minute wait |
| 305 | 11 | `Close(ETH.Vent)` | Seal ETH tank |
| 305 | 12-14 | `Open(ETH.Pressure)`, `Open(ETH.Main)`, `Open(ETH.Drain)` | Push ethanol out |
| 305 → 310 | 15 | `Sleep(seconds=5.0)` | |
| 310 | 16-18 | `Close(ETH.Pressure)`, `Close(ETH.Main)`, `Close(ETH.Drain)` | |
| 310 | 19 | `Open(ETH.Vent)` | Vent ETH tank |

Total ≈ 310 s.

## Notes

- The 300 s sleep is interruptible by Pause / Clear / Abort (5 ms polling) and the sequence panel shows its countdown (`remaining`).
- The operator must keep a client connected or at least have connected within the last 15 minutes; otherwise the inactivity abort could fire during the long sleep – in which case the abort sequence replaces the remainder.
- It does not touch the igniter or the purge valves.
