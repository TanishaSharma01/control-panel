# `reactfront/src/components/index.js`

Source: [`reactfront/src/components/index.js`](../../reactfront/src/components/index.js) (171 lines)

## 1. Purpose

Shared presentational components: the header bar with the global safety toggles, the generic `Panel` frame, and the styled toggle switches.

## 2. Imports

`React`; Material-UI `Paper`, `withStyles`, `Switch`.

## 3. Components

### `SectionTitle({ children })` (L6-14)
A 40 px title strip containing an `<h1>`.

### `UPSSymbols`, `UPSNames` (L17-22)
Map the server's `UPS_status` strings (`LINE_POWERED`, `BATTERY_POWERED`, `UNKNOWN`) to an emoji and a short name ("✅ Powered", "🔋 Battery", "❔ Unknown").

### `TopBar({ state, emit, sockStatus, that, activePanel, setActivePanel })` (L24-101)

Row 1: logo (`./logo.png`), title, UPS status, connection status (`sockStatus === WebSocket.OPEN` → green "Connected", else red "Disconnected"), and the **Server IP** input. Editing the input calls `that.setState({wsAddress})` on the `App` instance (passed as `that`), stores it in `localStorage['wsaddr']`, and reconnects – on every keystroke.

Row 2:

| Control | Shows | Sends |
|---|---|---|
| View buttons | `activePanel` | (local) `setActivePanel('control' \| 'sequence' \| 'calibration')` – labelled "Control Panel", "Config Panel", "Sensor Calibration" |
| Arming toggle | `state.data.arming_switch` | `ARMINGSWITCH` with the new boolean |
| Manual toggle | `state.data.manual_switch` | `MANUALSWITCH` |
| Logging toggle | `state.data.data_logging` | `DATALOG` |
| **ABORT** button | enabled (red) only when armed | `ABORTSEQUENCE` |

Toggles display server state, so a refused change (e.g. arming while a sequence runs) simply does not move.

### `Panel({ children, title, ...props })` (L103-112)
`<div class="panel">` + `SectionTitle` + Material-UI `Paper`. Extra props (including `className`, `style`) are spread onto the outer div; a passed `className` replaces the default, which is why callers write `className='panel control'`.

### `BigSwitch` (L114-149)
`withStyles`-customised 60 × 32 px `Switch`, green when checked.

### `NormalSwitch` (L151-158)
Another styled switch – **not used anywhere**.

### `ToggleSwitch({ value, setValue })` (L160-168)
"Off" label, `BigSwitch`, "On" label; `onChange` calls `setValue(!value)`.

## 4. Execution flow

`TopBar` is rendered by `App` on every state update. `Panel` wraps every view. `ToggleSwitch` is used by `TopBar` (and by the unused `SafetyCard`).

## 5. Edge cases

- The ABORT button is disabled when not armed; the server enforces the same rule ([known-issues.md](../known-issues.md) B1).
- With no connection, toggles and buttons call `emit`, which drops the command silently.
- An invalid address typed into the IP box makes `new WebSocket` throw; `App.connect` catches and logs it.
- `Paper` height style has an unbalanced parenthesis (`"calc(100% - 52px"`); the browser treats the declaration as invalid and ignores it.

## 6. Relationships

Imported by `index.js` (`TopBar`), all panels (`Panel`), `safety-panel.js` (`ToggleSwitch`). Server counterparts: `ARMINGSWITCH`, `MANUALSWITCH`, `DATALOG`, `ABORTSEQUENCE` branches of `handle_command`.
