# `reactfront/src/components/panels/sequence-panel.js`

Source: [`reactfront/src/components/panels/sequence-panel.js`](../../reactfront/src/components/panels/sequence-panel.js) (562 lines)

## 1. Purpose

The "Config Panel" view: choose a sequence file from the server, start / pause / clear it, watch remaining commands, and edit sequences (text lines, an "add command" builder, drag-and-drop reordering, Save / Save As / Delete).

## 2. Imports (L1-5)

Material-UI table components; React `useState`, `useEffect`; `Panel`; `SafetyCard` (imported, **unused**); `pinFromID`.

## 3. Definitions

### `VALVE_ATTR_BY_PIN` (L11-14)
`{ETH: {15:'Main', 16:'Fill', 17:'Drain', 8:'Pressure', 9:'Vent', 14:'Purge', 10:'Igniter'}, LOX: {14:'Main', 16:'Fill', 17:'Drain', 9:'Pressure', 10:'Vent', 8:'Purge', 15:'Chill'}}` – a third hand-kept copy of the valve pin map, needed to turn `{stand, pin}` back into the Python attribute name used in sequence files (`pins.json` names such as "ETH Pressurisation" are display labels, not attribute names).

### `SequenceRow(data)` (L16-29)
One table row for a command dict. Command name in title case; parameter is the valve's display name (via `pinFromID`) or, for sleeps, seconds to one decimal – `remaining` when in flight (live countdown), otherwise `ms`. In-flight rows are green.

### `Sequences({ state, emit })` (default export, L31-563)

**Local state:** `currentSequenceName`, `isEditing`, `editableCommands` (array of strings), `isAddingCommand`, `newCommandAction/Stand/Valve/Seconds`, `draggedIndex`.

**Derived from server state:** `sequences` (remaining commands), `current_executing` (`command_in_flight`), `aborting` (`status === 3`), `sequenceLoaded`, `serverSequenceName`, `availableSequences`, `armed`.

A `useEffect` (L54-58) copies the server's `current_sequence_name` into local state whenever it changes, so Save targets the right file after a page refresh.

| Handler | Lines | Behaviour |
|---|---|---|
| `handleChange` | 62-68 | Dropdown selection → `emit('SETSEQUENCE', name)` |
| `formatCommandToString` | 71-95 | `{name:'SLEEP', ms}` → `Sleep(seconds=5.0)`; `{name:'OPEN', stand, pin}` → `Open(ETH.Main)` via `VALVE_ATTR_BY_PIN`; unknown → `""` |
| `handleEdit` | 98-165 | Builds the editor's lines from the in-flight command plus the remaining commands; **if that yields nothing, it pre-fills a hard-coded copy of the `operation` sequence**; enters edit mode |
| `saveCommandsAs(name)` | 168-184 | `emit('SAVESEQUENCE', {name, commands})` with blank lines removed; leaves edit mode; shows an `alert` immediately (before any server confirmation) |
| `handleSave` | 187-189 | Save under the current name (overwrite) |
| `handleSaveAs` | 192-201 | `prompt` for a name; must match `/^[a-zA-Z0-9]+$/` |
| `handleCancel` | 204-207 | Leave edit mode |
| `handleDelete` | 210-227 | Requires arming; `confirm`; `emit('DELETESEQUENCE', name)`; clears local state |
| `updateCommand`, `removeCommandLine`, `reorderCommand` | 230-277 | Edit, delete (keeps at least one, possibly empty, line), drag-reorder |
| `startAddCommand`, `confirmAddCommand`, `cancelAddCommand` | 237-257 | Builder: action (Open/Close/Sleep) + stand + valve, or seconds → appends `Open(LOX.Main)` / `Sleep(seconds=5.0)` |

**Render:** left column – sequence `<select>` (disabled unless armed and not editing), **Start** (`BEGINSEQUENCE`), **Edit** or the Save / Save As New File / Cancel / Delete Sequence group. Right column – either the command table with **Pause** (`PAUSESEQUENCE`, shown while a command is in flight) and **Clear** (`UNSETSEQUENCE`), both hidden while aborting; or the line editor with drag handles, delete buttons and the add-command builder.

## 4. Execution flow

Mounted when `activePanel === 'sequence'`. Re-rendered at the state rate; all persistent data lives on the server, except unsaved edits.

## 5. Inputs / outputs

Props `state`, `emit`. Emits `SETSEQUENCE`, `BEGINSEQUENCE`, `PAUSESEQUENCE`, `UNSETSEQUENCE`, `SAVESEQUENCE`, `DELETESEQUENCE`. Never emits `GETSEQUENCE` – editor content is reconstructed from the loaded (parsed) sequence, not from the file text.

## 6. Relationships

Server side: `handle_command`, `save_sequence`, `delete_sequence`, `load_sequence` in `src/server.py`; command dict formats from `src/commands.py`.

## 7. Edge cases

- **Editing a partially executed or paused sequence saves only what remains.** The editor is built from `command_in_flight` + `current_sequence`; already-executed commands are gone, and a paused sleep is written with its *remaining* time. Reload the sequence (select it again) before editing to get the full file.
- Free-text lines are sent unvalidated; a typo makes the saved file unloadable ("Saved … but could not reload it" / "Could not load sequence …").
- The builder offers every valve for both stands, including non-existent ones (`LOX.Igniter`, `ETH.Chill`), which fail on load.
- `Sleep` precision is one decimal place on round-trip (`toFixed(1)`), so `Sleep(seconds=0.25)` becomes `0.3` after an edit/save cycle (and `abort.py`'s `3.000` becomes `3.0`).
- Saving requires arming on the server; un-armed, the request is silently ignored while the UI still says "Sequence saved".
- Clearing the selection of a deleted sequence relies on the server removing it from `available_sequences`.
- The unused `abort` helper (L279) is a leftover from a removed button.

## 8. Example

Loaded `hfon`, nothing executed, click **Edit** → editor lines `Open(ETH.Igniter)`, `Sleep(seconds=3.0)`, `Open(ETH.Main)`, … Add `Close(ETH.Igniter)` with the builder, drag it into place, **Save** → `{"header":"SAVESEQUENCE","data":{"name":"hfon","commands":[…]}}` → server rewrites `src/sequences/hfon.py` and reloads it.
