# `startup.sh`

Source: [`startup.sh`](../../startup.sh) (9 lines)

## Purpose

Boot-time launcher for the Raspberry Pi. The root README configures `/etc/rc.local` to run `sudo sh /home/pi/control-panel/startup.sh &`.

## Line by line

| Line | Code | Explanation |
|---|---|---|
| 1 | `#!/bin/sh` | POSIX shell |
| 2 | `sleep 20` | Wait for networking and USB enumeration after boot (the backend needs a network route for `get_local_ip()` and both LabJacks present) |
| 4 | `cd /home/pi/control-panel/src` | The backend's flat imports require `src/` as the working directory |
| 6 | `python3.10 server.py > /home/pi/control-panel/logs.txt 2>&1 &` | Start the WebSocket server in the background; stdout and stderr go to `logs.txt`, **overwritten** on every boot |
| 8 | `cd /home/pi/control-panel/reactfront/build` | |
| 9 | `python3.10 -m http.server 3000 &` | Serve the pre-built frontend on port 3000 |

## Assumptions

- Repository cloned at `/home/pi/control-panel`.
- `python3.10` on root's `PATH` with the requirements installed system-wide.
- `reactfront/build` exists (it is committed to git).

## Failure modes

- LabJacks not connected/powered at boot → `server.py` prints the hint and exits; nothing restarts it (README: `sudo reboot`).
- No network route after 20 s → `get_local_ip()` raises; server exits.
- There is no supervision (no systemd unit, no restart loop) and no log rotation. With the flow meter streaming, `[CRYO]` messages make `logs.txt` grow continuously.
- The frontend server starts regardless, so a page that loads but shows "Disconnected" usually means the backend died – check `logs.txt`.

## Manual equivalent (debugging)

```sh
pkill python3.10
cd ~/control-panel/src && python3.10 server.py          # foreground, see output
cd ~/control-panel/reactfront/build && python3.10 -m http.server 3000
```
