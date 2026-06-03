---
name: ableton-live-extensions
description: >-
  Build, scaffold, and debug Ableton Live extensions with the official Ableton
  Extensions SDK (TypeScript/JavaScript running in Node.js alongside Live). Use
  this skill whenever the user is writing an Ableton Live extension, mentions
  the `@ableton-extensions/sdk`, `extensions-cli`, `create-extension`, `.ablx`
  files, the Live "Extension Host", or wants to script/automate Ableton Live —
  e.g. batch-rename clips, create tracks/scenes, read or write warp modes, edit
  MIDI notes, import audio, render stems, add right-click context-menu actions,
  or show modal/progress dialogs in Live. Trigger even if the user only says
  "Ableton extension", "Live extension", or describes manipulating tracks,
  clips, devices, or the Live Set programmatically — this SDK is new (1.0.0
  beta) and its API differs from Max for Live and the Live Object Model, so do
  not rely on prior knowledge; follow this skill.
---

# Ableton Live Extensions SDK

Extensions extend Ableton Live with TypeScript/JavaScript. They run in **Node.js alongside Live** (the "Extension Host"), giving programmatic access to the Live Set, custom WebView UIs, the filesystem, and the npm ecosystem.

This skill targets **SDK 1.0.0-beta.0**. The API is new and unlike Max for Live or the LOM — write against the surface documented here and in `references/api-reference.md`, not from memory.

## When NOT to use extensions

Extensions are an offline/command-driven automation layer, **not** a realtime audio tool. They cannot do real-time audio processing, MIDI routing/real-time MIDI, drawing into Live's native UI, control-surface integration, or running in the background without Live open. If the user needs any of those, point them to **Max for Live** instead.

## The mental model (read this first)

Every extension follows the same spine. Internalize it before writing code:

1. **Activate** — Live calls your exported `activate(activation)` when the extension loads.
2. **Initialize** — call `initialize(activation, "1.0.0")` to get the `context` (a.k.a. `api`). This version-checks against the host and gives type-safe access.
3. **Register** — inside `activate`, register **commands** (named callbacks) and wire them to **context-menu actions** (and/or dialogs). `activate` returns immediately; your code runs later when commands fire.
4. **Resolve handles** — commands receive **handles** (lightweight IDs), not objects. Resolve a handle into a typed object with `context.getObjectFromHandle(handle, SomeClass)`.
5. **Read / mutate** — read properties, assign to writable properties, call methods (most mutations are `async` and return Promises). Group mutations with `withinTransaction` for a single undo step. Wrap long work in `withinProgressDialog`.

```ts
import { initialize, AudioClip, WarpMode, type ActivationContext, type Handle } from "@ableton-extensions/sdk";

export function activate(activation: ActivationContext) {
  const context = initialize(activation, "1.0.0");

  // 1. Register the command (the code that runs)
  context.commands.registerCommand("my-ext.cycle-warp", (arg: unknown) => {
    const clip = context.getObjectFromHandle(arg as Handle, AudioClip);
    const modes = [WarpMode.Beats, WarpMode.Tones, WarpMode.Texture, WarpMode.Repitch, WarpMode.Complex, WarpMode.ComplexPro];
    clip.warpMode = modes[(modes.indexOf(clip.warpMode) + 1) % modes.length];
  });

  // 2. Surface it as a right-click action on audio clips
  context.ui.registerContextMenuAction("AudioClip", "Cycle Warp Mode", "my-ext.cycle-warp");
}
```

## Critical rules that are easy to get wrong

These are the things an LLM writing this SDK from intuition will get wrong. Honor them.

- **Always `initialize(activation, "1.0.0")` first.** Nothing on `context` exists until you do. The version string is required and is the API version, not your extension's version.
- **Handles are not objects and are not permanent.** A `Handle` is just `{ id: bigint }`. It has no `.name` or methods. Resolve it every time with `getObjectFromHandle`; do **not** cache resolved objects or handles long-term. Handles become invalid when the object is deleted, **moved** (moving allocates a new handle), or the Live Set changes/closes. Using a stale handle throws.
- **`withinTransaction` is strictly synchronous — never `await` inside it.** It groups mutations into one undo step. To group *async* operations (creating tracks/clips), return a `Promise.all([...])` from the callback and `await` the `withinTransaction(...)` call itself. You **cannot** create an object and then modify it in the same transaction (you need the instance first) — do that as two sequential transactions. Nested transactions collapse into one undo step automatically.
- **Most mutations are async.** `createAudioTrack`, `createMidiClip`, `createAudioClip`, `clearClipsInRange`, `insertDevice`, `deleteTrack`, `importIntoProject`, `renderPreFxAudio`, etc. all return Promises. `await` them. Property assignments (`clip.name = "x"`, `track.mute = true`) are synchronous.
- **Filesystem is sandboxed.** You may only read/write `context.environment.storageDirectory` (persistent) and `context.environment.tempDirectory` (scratch). Do **not** touch the user's Documents/Downloads/Desktop or arbitrary paths — a stricter OS sandbox is coming and will break it. To pull in an external file, use `context.resources.importIntoProject(path)`, which runs host-side and returns the **imported copy's path** — always use that returned path afterward, never the original.
- **`WarpMode` is not contiguous:** `Beats=0, Tones=1, Texture=2, Repitch=3, Complex=4, ComplexPro=6` (note: **no 5**). Don't do `(mode + 1) % 6` blindly — iterate over an explicit array of the modes.
- **Context-menu scopes are a fixed set.** See the list below. Object scopes (e.g. `"AudioClip"`) pass a single `Handle`; selection scopes (`"AudioTrack.ArrangementSelection"`, `"MidiTrack.ArrangementSelection"`, `"ClipSlotSelection"`) pass a selection object instead.
- **Bundle to a single file.** The host does not resolve `node_modules` at runtime. esbuild (via the generated `build.ts`) bundles `src/extension.ts` → one JS file declared in `manifest.json`.
- **Use the type system for polymorphism.** Resolve to a base class (`Clip`, `Track`, `DataModelObject`) when the type is unknown, then narrow with `instanceof`. Generic types carry the version, e.g. `Track<"1.0.0">` — match the project's tsconfig (`strict`, `noUncheckedIndexedAccess`).

## Context-menu scopes (the `registerContextMenuAction` first argument)

Object scopes (command receives a `Handle`):
`"AudioClip"`, `"MidiClip"`, `"AudioTrack"`, `"MidiTrack"`, `"ClipSlot"`, `"Scene"`, `"Simpler"`, `"Sample"`, `"DrumRack"`.

Selection scopes (command receives a selection object):
- `"AudioTrack.ArrangementSelection"` / `"MidiTrack.ArrangementSelection"` → `ArrangementSelection` (`{ time_selection_start, time_selection_end, selected_lanes: Handle[] }`). Times are in **beats**.
- `"ClipSlotSelection"` → `ClipSlotSelection` (`{ selected_clip_slots: Handle[] }`).

`registerContextMenuAction` returns a Promise resolving to an `unregister()` function for dynamic removal.

## The `context` object at a glance

- `context.application` → `.song` (root of the object model: tracks, scenes, tempo, …).
- `context.commands` → `registerCommand(id, cb)`, `executeCommand(id, ...args)`.
- `context.ui` → `registerContextMenuAction`, `showModalDialog(url, w, h)`, `withinProgressDialog(text, {progress?}, cb)`.
- `context.environment` → `storageDirectory`, `tempDirectory`, `language` (may be `undefined`).
- `context.resources` → `importIntoProject(path)`, `renderPreFxAudio(track, start, end)`.
- `context.getObjectFromHandle(handle, Class)` → resolve a handle.
- `context.withinTransaction(fn)` → group mutations into one undo step.

For the **full object model** — every class, property, method signature, enum, and interface — read **`references/api-reference.md`**. Do this whenever you touch a class/method not shown above; the signatures are exact and several (e.g. `createAudioClip`'s args object) are non-obvious.

## Building UIs (modal dialogs / WebViews)

WebViews are HTML/CSS/JS shown via `context.ui.showModalDialog(url, width, height)`, which returns a Promise of the string the dialog posts back. Pass HTML as a data URL (`` `data:text/html,${encodeURIComponent(html)}` ``) so you don't host anything. The dialog returns data by posting a `close_and_send` message.

When the task involves a dialog, custom UI, or visual design that should feel native to Live, **read `references/webviews-and-ui.md`** — it has the message-passing contract, the cross-platform `postMessage` snippet, and Ableton's design guidance. A ready-to-style Live-themed HTML starter is in `assets/webview-template.html`.

For long operations, prefer `withinProgressDialog` (standard Live progress bar, cancellable via the `AbortSignal`) over a modal. See `references/recipes.md`.

## Project setup, dev loop, and packaging

When scaffolding a new extension or setting up the toolchain, **read `references/project-setup.md`** for the exact commands, file layout, `manifest.json`, `build.ts`, npm scripts, Developer Mode, and `.ablx` packaging. The short version:

- **Prereqs:** Node ≥ 24.14.1, the Live **Beta** that supports Extensions, and Developer Mode enabled in *Preferences → Extensions*.
- **Create:** `npx file:/path/to/ableton-create-extension-1.0.0-beta.0.tgz` inside a fresh folder; answer the prompts (it writes `.env` with `EXTENSION_HOST_PATH`).
- **Dev loop:** `npm start` (builds dev + launches the Extension Host; restart it to pick up changes — no need to relaunch Live).
- **Package:** `npm run package` → a `.ablx` the user installs by dropping it on Live's Extensions settings page.

## Debugging

`console.log/info/warn/error` and uncaught-exception stack traces go to the Extension Host log:
- **macOS:** `~/Library/Preferences/Ableton/Live x.x.x/ExtensionHost.txt`
- **Windows:** `\Users\<name>\AppData\Roaming\Ableton\Live x.x.x\Preferences\ExtensionHost.txt`

Wrap command bodies in `try/catch` and log errors — an unhandled rejection in an async command otherwise vanishes silently. Use `extensions-cli run --inspect` for VS Code debugging.

## Common recipes

For worked, copy-adaptable patterns — batch operations across a selection, importing audio into clip slots vs. the arrangement, rendering + analyzing audio, grouped async creation, progress + transaction combos — read **`references/recipes.md`**. These mirror the official SDK examples and encode the correct async/transaction shapes.

## Workflow when asked to build something

1. Clarify the trigger (context menu on what? a dialog? a one-shot command?) and the mutation.
2. Pick the right scope and the right object types; confirm signatures in `references/api-reference.md`.
3. Write `activate` → `initialize` → `registerCommand` → `registerContextMenuAction`.
4. Inside the command: resolve handles, do async reads, then group mutations in `withinTransaction`; wrap anything slow in `withinProgressDialog`.
5. Respect the filesystem sandbox; `importIntoProject` for external files.
6. Bundle (`npm run build:dev`), run via Developer Mode (`npm start`), check the log, iterate.
