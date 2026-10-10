# `reactfront/package.json`

Source: [`reactfront/package.json`](../../reactfront/package.json) (41 lines). The exact resolved versions are pinned in `package-lock.json` (generated; ~18 800 lines; not documented separately).

## 1. Purpose

npm manifest of the React frontend: dependencies, scripts, ESLint and Browserslist configuration.

## 2. Dependencies

| Package | Range | Used by the source? |
|---|---|---|
| `react`, `react-dom` | ^17.0.2 | Yes |
| `react-scripts` | ^5.0.1 | Yes – Create React App build tooling |
| `@material-ui/core` | ^4.12.3 | Yes – `Switch`, `Paper`, `Button`, `Snackbar`, `Table*`, `withStyles` |
| `scroll-lock` | ^2.1.5 | Yes – `graph-panel.js` |
| `@material-ui/lab` | ^4.0.0-alpha.60 | **No import found** |
| `moment` | ^2.29.1 | **No import found** |
| `recharts` | ^2.1.2 | **No import found** (graphs are hand-written SVG) |
| `node-pre-gyp` | ^0.17.0 | **No import found** |

Dev dependency: `cross-env` ^10.1.0 – sets environment variables portably in the scripts.

## 3. Scripts

| Script | Command | Purpose |
|---|---|---|
| `start` | `cross-env NODE_OPTIONS=--openssl-legacy-provider react-scripts start` | Dev server with hot reload on port 3000 |
| `build` | `cross-env NODE_OPTIONS=--openssl-legacy-provider react-scripts build` | Production bundle in `build/` |
| `eject` | `react-scripts eject` | CRA eject (irreversible) |

`--openssl-legacy-provider` works around webpack hashing errors on Node ≥ 17 with older toolchains. There is **no `test` script**.

## 4. Other fields

- `"private": true` – never published.
- `eslintConfig.extends: ["react-app"]` – CRA lint rules, applied during `start`/`build`.
- `browserslist` – production: `>0.2%, not dead, not op_mini all`; development: latest Chrome/Firefox/Safari.
- No `homepage` and no `proxy`: the app is served from the web root and talks to the backend directly over WebSocket.

## 5. Usage

```sh
cd reactfront
npm ci          # install exactly what package-lock.json specifies
npm start       # develop
npm run build   # rebuild reactfront/build (committed to git and served by the Pi)
```

## 6. Notes

- The built output in `reactfront/build/` is tracked by git; remember to rebuild and commit (or `scp`, see `update_frontend.sh`) after frontend changes, otherwise the Pi keeps serving the old bundle.
- `reactfront/.gitignore` excludes `node_modules`, coverage, env files and the build's JS source maps.
