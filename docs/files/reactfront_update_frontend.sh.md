# `reactfront/update_frontend.sh`

Source: [`reactfront/update_frontend.sh`](../../reactfront/update_frontend.sh) (8 lines)

## Purpose

Developer convenience script to rebuild the frontend and push it to the Pi over the range network, without git.

## Steps

| Line | Command | Effect |
|---|---|---|
| 3 | `ssh pi@192.168.0.5 "cd ~/control-panel/reactfront; rm -r build"` | Delete the old build on the Pi |
| 5 | `npm run build` | Produce a fresh `build/` locally |
| 7 | `scp -r build pi@192.168.0.5:/home/pi/control-panel/reactfront` | Copy it across |

Then reload `http://192.168.0.5:3000`; the Pi's `http.server` serves files straight from disk, so no restart is needed.

## Assumptions and caveats

- Must be run from `reactfront/` with dependencies installed (`npm ci`), with SSH access to `pi@192.168.0.5` (key copied via `ssh-copy-id`, per the root README).
- No shebang and no `set -e`: if the build fails, the script still continues to `scp` (which then fails or copies a stale directory) **after** the remote build has already been deleted, leaving the Pi with no frontend until a successful run.
- Only updates the frontend. Backend changes (`src/`) must be copied separately and the server restarted.
- Calibration constants are stored in the browser per origin, so they survive a redeploy.
