# `src/sequences/hfon.py`

Source: [`src/sequences/hfon.py`](../../src/sequences/hfon.py) (16 lines)

## Purpose

"Hot fire on" – added in commit `f9b1e3c` ("hot fire commands"). Fires the igniter, opens both main valves for 3 s, then purges and vents. The name's meaning is **[Inferred]** from the commit message and content.

Format and loading rules: see [abort.py](src_sequences_abort.py.md).

## Timeline

| t (s) | Lines | Commands | Effect |
|---|---|---|---|
| 0 | 1 | `Open(ETH.Igniter)` | Igniter channel on (ETH pin 10) |
| 0 → 3 | 2 | `Sleep(seconds=3.0)` | Igniter lead time |
| 3 | 3-4 | `Open(ETH.Main)`, `Open(LOX.Main)` | Main propellant flow |
| 3 → 6 | 5 | `Sleep(seconds=3.0)` | 3 s burn |
| 6 | 6-7 | `Close(ETH.Main)`, `Close(LOX.Main)` | Shutdown |
| 6 | 8-9 | `Open(ETH.Purge)`, `Open(LOX.Purge)` | Purge |
| 6 | 10-11 | `Close(ETH.Pressure)`, `Close(LOX.Pressure)` | Stop pressurising |
| 6 | 12-13 | `Open(ETH.Vent)`, `Open(LOX.Vent)` | Vent tanks |
| 6 → 16 | 14 | `Sleep(seconds=10.0)` | 10 s purge |
| 16 | 15-16 | `Close(ETH.Purge)`, `Close(LOX.Purge)` | End purge |

Total ≈ 16 s.

## Observations **[Existing]**

- **Preconditions are not established by the sequence.** It never closes the vents or opens pressurisation, so the tanks must already be pressurised (manually or by another sequence) before it is started.
- **The igniter is never switched off.** There is no `Close(ETH.Igniter)`; after the sequence the igniter output stays on until closed manually or by an abort (`abort.py` line 1). Confirm whether this is intended.
- Unlike a manual igniter activation, running it from a sequence shows **no confirmation dialog** (the dialog only wraps the manual switch in `control-panel.js`).
