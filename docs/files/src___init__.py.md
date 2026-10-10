# `src/__init__.py`

Source: [`src/__init__.py`](../../src/__init__.py) (0 bytes)

## Purpose

An empty file that marks `src/` as a Python package.

## Does it matter?

Not for running the server. Every module in `src/` uses **flat, top-level imports** (`from stands import LOX`, `import utils`), which resolve because `server.py` is launched from inside `src/` (the script's directory is put on `sys.path`). Nothing imports `src.<module>`. The file would only matter if the code were imported as a package from the repository root (e.g. by a future test suite using `from src import commands`) – and in that case the flat imports inside the modules would fail unless `src/` is also on `sys.path`.

## Contents

None: no imports, definitions, execution flow, inputs, outputs or error handling.
