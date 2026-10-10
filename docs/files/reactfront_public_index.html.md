# `reactfront/public/index.html`

Source: [`reactfront/public/index.html`](../../reactfront/public/index.html) (13 lines)

## Purpose

The HTML shell that Create React App copies into the build and into which it injects the compiled script and stylesheet tags.

## Contents

| Element | Meaning |
|---|---|
| `<meta charset="utf-8">` | Encoding |
| `<meta name="viewport" content="width=device-width, initial-scale=1">` | Mobile viewport (the layout itself is fixed-width desktop) |
| `<meta name="theme-color" content="#000000">` | Browser chrome colour |
| `<title>Control Panel</title>` | Tab title |
| `<noscript>` | Message when JavaScript is disabled |
| `<div id="root"></div>` | Mount point used by `ReactDOM.render` in `src/index.js` |

There is no `<link rel="icon">`; browsers request `/favicon.ico` by default, which exists in `public/`. `public/logo.png` is referenced by `TopBar` as `./logo.png`.

## Execution flow

Browser requests `/` from the Pi's `http.server` → receives the built `index.html` → loads `static/js/main.<hash>.js` and `static/css/main.<hash>.css` → React mounts into `#root`.

## Notes

No inline configuration: the backend address is not set here (it defaults to `192.168.0.5` in `src/index.js` and is user-editable in the top bar).
