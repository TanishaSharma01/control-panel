# Repository configuration files

Small configuration files that affect tooling rather than runtime behaviour.

## `.gitignore` (root)

```
.DS_Store
**/__pycache__/
logs
src/venv
venv
```

Ignores macOS metadata, Python bytecode, virtual environments and **any path named `logs`** – so server data logs in `src/logs/` are never committed. Test data must be archived separately. (`logs.txt` on the Pi is not matched, but lives outside normal development checkouts.)

## `reactfront/.gitignore`

Ignores `node_modules`, `.pnp*`, `coverage`, local env files, npm/yarn debug logs, and `build/static/js/*.map`. The rest of `build/` is **tracked**.

## `.vscode/settings.json`

```json
{ "python.pythonPath": "C:\\Users\\pstefa\\AppData\\Local\\Programs\\Python\\Python310\\python.exe" }
```

A previous developer's machine-specific interpreter path. It has no effect on other machines (and `python.pythonPath` is a deprecated VS Code setting); it only confirms Python 3.10 as the intended version.

## `README.md` (root)

The original project guide: development setup, first-time Raspberry Pi provisioning (Exodriver, static IP, `apcupsd`, Python 3.10 build), field setup, debugging and deployment notes. It remains the authority for Pi provisioning; this `docs/` suite complements it. Known stale point: it describes typing a sequence name into a popup, whereas the UI now offers a dropdown populated from `available_sequences`. It also contains the default Pi password in an untested deploy snippet.

## `src/logs/`

Created at run time by `Datalog`; contains one CSV per logging session. Format: [src_datalog.py.md](src_datalog.py.md).
