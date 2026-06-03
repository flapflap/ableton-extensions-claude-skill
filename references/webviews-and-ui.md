# WebViews, Dialogs & Design

Extensions render custom UI as **WebViews** — standard HTML/CSS/JS shown in a Live modal dialog. Use them for forms, configuration, data tables, multi-step flows. For background work with a progress bar, use `withinProgressDialog` instead (see `recipes.md`).

## Showing a modal dialog

```ts
const result: string = await context.ui.showModalDialog(url, width, height);
```

- Returns the **string** the dialog posts back (typically stringified JSON).
- While open, the extension is suspended waiting on the Promise.
- Pass HTML inline as a **data URL** so you don't have to host anything:

```ts
import modalInterface from "./interface.html";   // esbuild .html text loader

const result = await context.ui.showModalDialog(
  `data:text/html,${encodeURIComponent(modalInterface)}`,
  360, 240,
);
const { name } = JSON.parse(result);
```

## Message-passing contract

Communication is via message passing. **Extension → WebView:** inline the data into the HTML string before encoding, or append query params to the data URL and read them in the page. **WebView → Extension:** post a message with method `close_and_send`; its `params` must be an **array containing a single string** (usually stringified JSON). That string becomes the resolved value of `showModalDialog`.

Cross-platform sender (macOS uses WebKit, Windows uses WebView2):

```js
function doSendMessage(message) {
  if (window.webkit?.messageHandlers?.live) {
    window.webkit.messageHandlers.live.postMessage(message);   // macOS
  } else if (window.chrome?.webview) {
    window.chrome.webview.postMessage(message);                // Windows
  }
}
function closeWithResult(result) {
  doSendMessage({ method: "close_and_send", params: [JSON.stringify(result)] });
}
```

Always provide a Cancel path that still resolves the Promise (e.g. `closeWithResult({ name: null })`) and handle Enter/Escape keys, so the extension never hangs waiting on a dialog the user dismissed.

A complete, Live-themed starter (dark theme tokens, styled input/buttons, keyboard handling) is in **`assets/webview-template.html`** — copy it into `src/` and adapt.

## Design principles (from Ableton's guide)

Aim for UIs that feel native to Live:

- **Visual hierarchy = sonic importance.** The most impactful controls are the most prominent; demote secondary ones with smaller size or reduced contrast. Scan the UI with your eyes — do the key controls stand out?
- **Choose a focal point** (a visualization or LCD) and arrange controls around it.
- **Group related parameters** with dividers, spacing, or color accents; organize many params into logical groups.
- **Vary layout and rhythm** — don't repeat one control type mechanically; five options could be radios *or* a select.
- **Use whitespace** as a design element; if controls feel cramped, add padding/margins.
- **Reflect data/processing flow** top-to-bottom or left-to-right (matches hardware and Live).
- **Visualize what you can't hear** — extensions can't process realtime audio, so preview output visually when params change, where feasible.
- **Hide advanced/optional controls** in expandable sections; show essentials.
- **Disable, don't hide** — grey out unavailable controls (`:disabled`) rather than removing them, to keep layout stable.
- **Stay consistent with Live** — familiar terminology and interaction patterns keep users in flow.

### Color & components

- Live's signature **yellow** and **blue** signal interactivity. With custom colors, meet **WCAG** contrast.
- **Buttons:** verbs as labels; single words best; equal widths in a group; make confirm labels say what happens; "Cancel" stays "Cancel". Don't use a button for on/off — use a checkbox or toggle.
- **Radio vs select:** radios for ≤5 options you want visible at once; a select for ~4+ items or long labels.
- **Toggles vs checkboxes:** prominent toggles for on/off that invite experimentation; standard checkboxes for related options/preferences ("Save my settings").
- **Sliders:** for continuous range/relative position. Avoid sliders for 2-step discrete values — use a radio group so you don't imply false continuity.

### Live dark-theme CSS tokens (from the official example)

```css
:root {
  --p-live-ui-bg: hsl(0,0%,21%);
  --p-live-control-bg: hsl(0,0%,16%);
  --p-live-input-bg: hsl(0,0%,12%);
  --p-live-text-primary: hsl(0,0%,71%);
  --p-live-text-secondary: hsl(0,0%,41%);
  --p-live-control-border: hsl(0,0%,7%);
  --p-live-accent-primary: hsl(31,100%,67%);   /* Live's orange/yellow */
}
```

See `assets/webview-template.html` for these mapped to usable `--c-*` variables and applied to inputs/buttons.
