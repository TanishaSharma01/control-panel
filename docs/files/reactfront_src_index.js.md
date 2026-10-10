# `reactfront/src/index.js`

Source: [`reactfront/src/index.js`](../../reactfront/src/index.js) (237 lines)

## 1. Purpose

Frontend entry point. Defines the root `App` component, which owns the WebSocket connection, holds the latest server state, routes incoming messages to the graphs and child panels, and exposes `emit` for sending commands. Also defines the `DebugLogPanel`.

## 2. Imports (L1-11)

| Import | Use |
|---|---|
| `React`, `ReactDOM` | Components, mounting |
| `./index.css` | Global styles |
| `TopBar` from `./components/index` | Header bar |
| `GraphPanel`, `newData`, `newEvent`, `pinFromID` from `graph-panel` | Graph column; functions to push data/events; pin metadata lookup |
| `Sequences` from `sequence-panel` | Sequence editor view |
| `ControlPanel`, `SequenceExecutionList` from `control-panel` | P&ID view and execution list |
| `formatDataPoint`, `emptyDataPoint` from `./utils` | Unit conversion; all-NaN point for graph gaps |
| `undefOnBadRef` from `graph-utils` | Safe property access |
| `Snackbar`, `Button` from Material-UI | Warning toast |
| `CalibrationPanel` | Calibration view |

## 3. `DebugLogPanel({ debugLog })` (L16-41)

Fixed bar at the bottom of the window showing `state.data.debug_log` (server `log_debug` lines, mostly `[CRYO]` diagnostics). Local state `collapsed`; a `useEffect` keyed on `debugLog` scrolls to the bottom on each update. Header shows the line count and a show/hide toggle.

## 4. `class App`

### State (constructor, L45-69)

| Field | Initial | Meaning |
|---|---|---|
| `data` | `null` | Latest `STATE.data` |
| `mostRecentWarning`, `showWarning` | `{}`, `false` | Snackbar content |
| `wsAddress` | `localStorage['wsaddr']` or `"192.168.0.5"` | Server host |
| `defaultWSAddress` | `"192.168.0.5"` | Input placeholder |
| `events` | `[]` | Valve events received |
| `socketStatus` | `-1` | `WebSocket.readyState` |
| `activePanel` | `'control'` | `'control'` / `'sequence'` / `'calibration'` |
| `fullSequence`, `fullSequenceName` | `[]`, `null` | Snapshot of a sequence taken when it first appears |

Instance fields: `loxFlowBuffer = []`, `LOX_FLOW_BUFFER_SIZE = 10`. The constructor binds `emit` and `setActivePanel` and calls `this.connect()` immediately.

### Lifecycle

- `componentDidMount` (L70-73): `setInterval(() => this.emit('PING'), 200)`; `mounted = true`.
- `componentWillUnmount` (L74-77): clear the interval; `mounted = false` (messages are ignored afterwards).

### `pushWarning(time, message)` (L78-83)
Shows the snackbar when there is no previous warning or when **both** the time and the message differ from the previous one.

### `updateSocketStatus()` (L84-91)
Copies `socket.readyState` into state; if not `OPEN`, raises "Server connection lost". (Because it is also called right after `new WebSocket`, while `CONNECTING`, this warning appears during every connection attempt.)

### `connect()` (L92-180)

1. Close any existing socket; `new WebSocket("ws://<wsAddress>:8888")`.
2. `onopen`: log, update status.
3. `onclose`: update status, null the socket, push `emptyDataPoint` (breaks graph lines), retry `connect()` after 1 s if nothing else reconnected.
4. `onmessage`: parse JSON and `switch (data.type)`:

**`STATE` (L117-159)**

- `data.data.time = parseInt(time) / 1000` (ms → s).
- If more than 1 s elapsed since the previous state time, push `emptyDataPoint` and clear the flow buffer (segment break after a stall).
- `dataPoint = formatDataPoint(data.data)` – volts → engineering units.
- LOX flow smoothing: append non-NaN `'LOX Flow'` to `loxFlowBuffer` (max 10, FIFO); if the buffer is non-empty replace `dataPoint['LOX Flow']` with the buffer mean. Consequence: while the server reports `null` (NaN), the graph keeps plotting the last mean rather than a gap.
- `newData(dataPoint)` → all dataloggers.
- Sequence snapshot: if `current_sequence_name` is set and differs from `fullSequenceName`, store `current_sequence` as `fullSequence` (at that instant nothing has executed). If the name became empty, clear the snapshot. (Re-loading the *same* name after edits does not refresh the snapshot.)
- `setState({data, …})` → re-render of the whole tree.
- If `latest_warning` is present → `pushWarning`.

**`PING` (L160-162)** – `ping = now − data.data` (round-trip ms). Stored, not displayed.

**`VALVE` (L163-172)** – `time = data.time / 1000` (server envelope time); look up the pin with `pinFromID(pin, stand)`; label `"Opened LOX main"` / `"Closed ETH vent"` (`test_stand + ' ' + abbrev`); `newEvent({time, label})` for the graphs; append to `state.events`.

**default** – `console.error(data)` (this is where `SEQUENCE_SAVED/DELETED/CONTENT` end up).

The whole body is wrapped in `try/catch` that logs construction errors (e.g. an invalid address).

### `emit(commandHeader, parameter = null)` (L181-188)
If the socket is open, sends `{header, data, time: Date.now()}`; otherwise silently does nothing.

### `render()` (L193-230)
`TopBar`; left column: the active view (`SequenceExecutionList` + `ControlPanel`, or `Sequences`, or `CalibrationPanel`); `GraphPanel` always; warning `Snackbar` with Dismiss; `DebugLogPanel`.

### Mount (L233-238)
`ReactDOM.render(<React.StrictMode><App/></React.StrictMode>, #root)`.

## 5. Execution flow

Bundle loads → `App` constructed (socket opens) → mounted (ping timer) → ~20 `STATE` messages/s drive `setState` → children re-render; user actions call `emit`.

## 6. Inputs / outputs

In: WebSocket messages, `localStorage['wsaddr']`, user events. Out: WebSocket commands, DOM, `datalogger-new-data` / `datalogger-new-event` events, `localStorage['wsaddr']` (written from `TopBar`).

## 7. Relationships

Parent of every UI component; the only file that touches the WebSocket. Depends on `utils.js` for conversion and `graph-panel.js` for graph ingestion.

## 8. Edge cases

- An exception in `formatDataPoint` (e.g. `labjacks.ETH.analog` missing) is thrown inside `onmessage`; that message is lost and the UI does not update.
- A `VALVE` for a pin not in `pins.json` makes `pinFromID` return `undefined` → `TypeError`.
- Under React 17 StrictMode the constructor runs twice in development, briefly opening two sockets.
- Changing the server address in the top bar calls `connect()` on every keystroke.
- Port 8888 and the `ws://` scheme are hard-coded; the page must therefore be served over HTTP, not HTTPS.

## 9. Example

Incoming `{"type":"VALVE","time":1785127770104,"data":{"header":"OPEN","pin":14,"stand":"LOX",…}}` → `newEvent({time: 1785127770.104, label: "Opened LOX main"})` → a dashed vertical line on every graph.
