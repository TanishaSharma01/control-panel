# `reactfront/src/components/panels/safety-panel.js`

Source: [`reactfront/src/components/panels/safety-panel.js`](../../reactfront/src/components/panels/safety-panel.js) (22 lines)

## 1. Purpose

Defines `SafetyCard`, a small titled card that wraps either a `ToggleSwitch` or arbitrary children. It belonged to an earlier "Safety" panel that held the arming / manual / logging switches; those controls now live in `TopBar` (`components/index.js`).

## 2. Status

**Effectively unused.** The only import is in `sequence-panel.js` (L4), which never renders it. The matching CSS (`.panel.safety`, `.safety-card`) is also orphaned.

## 3. Imports

`React`; `ToggleSwitch` from `../index`.

## 4. `SafetyCard(props)`

| Prop | Meaning |
|---|---|
| `title` | Heading text |
| `label` | Optional tooltip; when present the heading gets a dotted underline and a help cursor |
| `style` | Extra heading styles |
| `children` | If given, rendered instead of the switch |
| `switchValue`, `setSwitchValue` | Passed to `ToggleSwitch` when there are no children |

Renders `<div class="safety-card"><h2 title={label}>{title}</h2>{children or ToggleSwitch}</div>`.

## 5. Inputs, outputs, errors

Pure presentational component; no side effects, no error handling.

## 6. Example

```jsx
<SafetyCard title="Arming Switch" label="Controls if the state can change"
            switchValue={armed} setSwitchValue={x => emit('ARMINGSWITCH', x)} />
```
