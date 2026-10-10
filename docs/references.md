# References

## Internal sources

| Source | Used for |
|---|---|
| [`README.md`](../README.md) (repository root) | Pi provisioning, network layout, UPS setup, deployment |
| Source code comments in `src/stands.py`, `src/labjack.py`, `src/server.py`, `reactfront/src/utils.js` | Sensor types, pin rationale, shunt value, design decisions |
| `git log` (415 commits; e.g. `f9b1e3c` hot-fire sequences, `9db31f0` UART pins, `25a0317` calibration mA → V, `41e80ab` median of 3 thermocouple reads, `305aa9b` load cell) | History of sensor changes |
| `src/logs/*.csv` (local, git-ignored) | Example of the log format |

There are **no datasheets, schematics, wiring diagrams, calibration certificates or firmware sources** in the repository.

## External documentation

### LabJack U3 and LabJackPython

| Topic | Link | Facts taken from it |
|---|---|---|
| U3 datasheet – analog inputs | <https://support.labjack.com/docs/2-6-ain-u3-datasheet> | 12-bit resolution; single-ended range ≈ 0–2.44 V; special 0–3.6 V range; U3-HV high-voltage inputs |
| U3-HV product page | <https://labjack.com/products/u3-hv> | First four flexible I/O replaced by ±10 V analog inputs |
| U3 datasheet §4.1.10 – SPI | <https://support.labjack.com/docs/4-1-10-spi-serial-communication-u3-datasheet> | SPI master on hardware ≥ 1.21; mode and clock-factor options |
| LabJack forum – SPI with U3 | <https://files.labjack.com/utilities/archive/forum_html_backup/forums/u3/spi-u3.html> | Clock factor 0 ≈ 80 kHz maximum |
| U3 datasheet §4.1.12 – asynchronous serial | <https://support.labjack.com/docs/4-1-12-asynchronous-serial-communication-u3-datash> | 8N1 TTL UART; TX/RX placed after timers/counters at the pin offset |
| AsynchConfig / AsynchTX low-level functions | <https://support.labjack.com/docs/asynchconfig>, <https://support.labjack.com/docs/asynchtx> | `BaudFactor16 = 2¹⁶ − 48 MHz / (2 × baud)`; pins must be assigned with ConfigIO; TX up to 56 bytes |
| LabJackPython `u3.py` source | <https://github.com/labjack/LabJackPython/blob/master/src/u3.py> | Signatures/behaviour of `configIO`, `getAIN`, `getDIOState`, `spi`, `asynchConfig`, `asynchRX` (32 bytes per read from a 256-byte buffer), `asynchTX`; integer division in the baud calculation. *Read from the master branch; the project pins v2.0.4 – verify there if exact behaviour matters* |
| Exodriver | <https://github.com/labjack/exodriver> | Native USB library required on Linux/macOS |

### MAX6675

| Topic | Link | Facts taken from it |
|---|---|---|
| MAX6675 datasheet (Analog Devices / Maxim) | <https://www.analog.com/media/en/technical-documentation/data-sheets/MAX6675.pdf> | K-type only; 0 to +1024 °C; 0.25 °C resolution; 8 LSB accuracy 0–700 °C; conversion time 0.17 s typ / 0.22 s max; 16-bit read-only SPI word (D15 dummy sign, D14–D3 temperature, D2 open-thermocouple, D1 ID, D0 three-state); CS low stops a conversion, CS high starts a new one; read on the falling clock edge |

The official PDF could not be fetched while writing; the figures were cross-checked against search-indexed copies of the datasheet. Re-verify against the PDF before using them for design decisions.

### Flow meter ("PT420")

No authoritative source found. A web search for a cryogenic "PT420" flow meter returned only an unrelated general-purpose analog flow transmitter (Badger Meter PFT420). The manufacturer, K-factor and the "sheet" mentioned in `utils.js` L305 must be supplied by the hardware team.

### Pressure transmitters, load cell and amplifier

No models are recorded, so no datasheets can be cited.

### UPS monitoring

| Topic | Link |
|---|---|
| apcupsd manual (`apcaccess status` output, `STATUS : ONLINE / ONBATT`) | <http://www.apcupsd.org/manual/manual.html> |

### Software libraries

| Library | Link |
|---|---|
| `websockets` (Python) | <https://websockets.readthedocs.io/> |
| React 17 | <https://17.reactjs.org/> |
| Material-UI v4 | <https://v4.mui.com/> |
| Create React App | <https://create-react-app.dev/> |
| scroll-lock | <https://github.com/FL3NKEY/scroll-lock> |

## Background theory (general knowledge, not project-specific)

- 4–20 mA current-loop signalling and live-zero fault detection.
- Thermocouple Seebeck effect, cold-junction compensation, NIST ITS-90 type-K tables (<https://srdata.nist.gov/its90/main/>).
- Strain-gauge load cells and Wheatstone bridges.
- Turbine flow meter K-factor relationship `Q = f / K`.

## Confidence notes

| Statement class | Confidence |
|---|---|
| Anything describing what the code does, with file/line references | High – read directly from the source at commit `4357133` (branch `documentation`) |
| LabJack/LabJackPython behaviour | Medium-high – from vendor documentation and library source, not exercised on hardware here |
| MAX6675 specifications | Medium-high – widely reproduced datasheet values; official PDF not re-read |
| Shunt value, transmitter type and ranges | Medium – inferred from code comments and defaults |
| Flow-meter principle and firmware behaviour | Low-medium – inferred from the CSV column names and old calibration constants |
| Physical layout, wiring, device variants | Unknown – requires the hardware team |
