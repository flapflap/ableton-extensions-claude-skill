# Recipes

Copy-adaptable patterns mirroring the official SDK examples. Every snippet assumes:

```ts
const context = initialize(activation, "1.0.0");
```

## 1. Context-menu command on a single object

```ts
context.commands.registerCommand("my-ext.process-clipslot", () => {
  console.log("You right-clicked on a ClipSlot!");
});
context.ui.registerContextMenuAction("ClipSlot", "Process this ClipSlot", "my-ext.process-clipslot");
```

## 2. Resolve + mutate a clip (polymorphic over audio/MIDI)

```ts
context.commands.registerCommand("my-ext.rename", (arg: unknown) => {
  const clip = context.getObjectFromHandle(arg as Handle, Clip);   // base class → works for both
  const suffix = clip instanceof AudioClip ? " (Audio)" : clip instanceof MidiClip ? " (MIDI)" : "";
  clip.name = "Processed" + suffix;
});
(["AudioClip", "MidiClip"] as const).forEach((scope) =>
  context.ui.registerContextMenuAction(scope, "Rename this clip", "my-ext.rename"),
);
```

## 3. Cycle warp mode (respect the non-contiguous enum)

```ts
context.commands.registerCommand("my-ext.cycle-warp", (arg: unknown) => {
  const clip = context.getObjectFromHandle(arg as Handle, Clip);
  if (!(clip instanceof AudioClip)) return console.error("Not an AudioClip.");
  const modes = [WarpMode.Beats, WarpMode.Tones, WarpMode.Texture, WarpMode.Repitch, WarpMode.Complex, WarpMode.ComplexPro];
  clip.warpMode = modes[(modes.indexOf(clip.warpMode) + 1) % modes.length]!;
});
context.ui.registerContextMenuAction("AudioClip", "Cycle Warp Mode", "my-ext.cycle-warp");
```

## 4. Arrangement selection across multiple tracks

The selection scope hands you an `ArrangementSelection` (times in **beats**), not a handle. Resolve each lane with the most generic base, then narrow.

```ts
context.commands.registerCommand("my-ext.process-selection", async (arg: unknown) => {
  const selection = arg as ArrangementSelection;
  const objects = selection.selected_lanes.map((h) => context.getObjectFromHandle(h, DataModelObject));
  const tracksAndLanes = objects.filter(
    (o): o is Track<"1.0.0"> | TakeLane<"1.0.0"> => o instanceof Track || o instanceof TakeLane,
  );

  const midiLanes = tracksAndLanes.filter(
    (o): o is MidiTrack<"1.0.0"> | TakeLane<"1.0.0"> =>
      o instanceof MidiTrack || (o instanceof TakeLane && o.parent instanceof MidiTrack),
  );

  const len = selection.time_selection_end - selection.time_selection_start;
  const newClips = await Promise.all(
    midiLanes.map((lane) => lane.createMidiClip(selection.time_selection_start, len)),
  );
  newClips.forEach((clip, i) => { clip.name = `New Clip ${i + 1}`; });
});
context.ui.registerContextMenuAction("MidiTrack.ArrangementSelection", "Process selection", "my-ext.process-selection");
```

## 5. Import an audio file into a clip slot vs. the arrangement

`importIntoProject` returns the **imported copy's path** — always pass that to `createAudioClip`. Note the two different `createAudioClip` shapes.

```ts
// Into a session ClipSlot (no startTime):
context.commands.registerCommand("my-ext.slot-clip", async (arg: unknown) => {
  const slot = context.getObjectFromHandle(arg as Handle, ClipSlot);
  const imported = await context.resources.importIntoProject("/path/to/audio.wav");
  await slot.createAudioClip({
    filePath: imported, isWarped: true,
    loopSettings: { looping: true, startMarker: 1, endMarker: 5, loopStart: 1, loopEnd: 5 },
  });
});

// Into an AudioTrack arrangement (startTime + duration in beats):
context.commands.registerCommand("my-ext.arr-clip", async (arg: unknown) => {
  const sel = arg as ArrangementSelection;
  const track = context.getObjectFromHandle(sel.selected_lanes[0]!, AudioTrack);
  const imported = await context.resources.importIntoProject("/path/to/audio.wav");
  await track.createAudioClip({
    filePath: imported,
    startTime: sel.time_selection_start,
    duration: sel.time_selection_end - sel.time_selection_start,
    isWarped: false,
  });
});
```

## 6. Download a file from an API, then add it to the Set

Stay inside the sandbox: write to `tempDirectory`, then `importIntoProject`.

```ts
import * as fs from "fs/promises";
import * as path from "path";

const res = await fetch("https://api.example.com/audio/generate");
const buf = Buffer.from(await res.arrayBuffer());
const tmp = path.join(context.environment.tempDirectory!, "generated.wav");
await fs.writeFile(tmp, buf);                                   // tempDir is writable
const imported = await context.resources.importIntoProject(tmp);
await slot.createAudioClip({ filePath: imported, isWarped: false });
```

## 7. Persist config across sessions

```ts
const cfg = path.join(context.environment.storageDirectory!, "config.json");
await fs.writeFile(cfg, JSON.stringify({ apiKey: "…" }));       // storageDir survives sessions
const saved = JSON.parse(await fs.readFile(cfg, "utf-8"));
```

## 8. Group async creation into one undo step

`withinTransaction` is synchronous — return `Promise.all(...)` and await the call. You can't create-then-modify in one transaction; split into two.

```ts
const newTracks = await context.withinTransaction(() =>
  Promise.all([context.application.song.createAudioTrack(), context.application.song.createAudioTrack()]),
);
context.withinTransaction(() => {                                // second undo step
  newTracks.forEach((t, i) => { t.name = `Grouped Track ${i + 1}`; });
});
```

## 9. Batch rename in one undo step (purely synchronous mutations)

```ts
context.withinTransaction(() => {
  context.application.song.tracks.forEach((t, i) => { t.name = `Track ${i + 1}`; });
});
```

## 10. Progress dialog for a long task (cancellable)

`update(text, percent 0–100)`; `percent` may be `undefined` for indeterminate. Check `signal.aborted` / `signal.throwIfAborted()` between steps. The dialog opens on call and closes when the callback settles.

```ts
context.commands.registerCommand("my-ext.long", () => {
  void context.ui.withinProgressDialog("Doing a long task", {}, async (update, signal) => {
    for (let i = 0; i < 100; i++) {
      await someStep();
      await update("Working…", i);
      signal.throwIfAborted();          // bail cleanly if the user cancels
    }
    await update("Cleaning up", undefined);
  });
});
context.ui.registerContextMenuAction("AudioTrack", "Start long task", "my-ext.long");
```

## 11. Render → analyze → strip silence (progress + transaction combo)

The canonical "real tool" shape: do async work (render + analyze) inside the progress dialog, then apply all mutations in one transaction.

```ts
context.commands.registerCommand("my-ext.strip-silence", (arg: unknown) =>
  void (async (selection: ArrangementSelection) => {
    const tracks = selection.selected_lanes
      .map((h) => context.getObjectFromHandle(h, DataModelObject))
      .filter((o): o is AudioTrack<"1.0.0"> => o instanceof AudioTrack);
    if (!tracks.length) return console.log("No audio tracks selected.");

    await context.ui.withinProgressDialog("Strip Silence", {}, async (update, signal) => {
      const results: { track: AudioTrack<"1.0.0">; ranges: { start: number; end: number }[] }[] = [];
      for (let i = 0; i < tracks.length; i++) {
        if (signal.aborted) return;
        const track = tracks[i]!;
        update(`Analyzing ${i + 1}/${tracks.length}: ${track.name}`, (i / tracks.length) * 50);
        const wav = await context.resources.renderPreFxAudio(
          track, selection.time_selection_start, selection.time_selection_end,
        );
        const ranges = await analyzeSilence(wav);     // your analysis (seconds)
        if (ranges.length) results.push({ track, ranges });
      }
      if (signal.aborted || !results.length) return;

      update("Stripping silence", 80);
      const beatsPerSecond = 60 / context.application.song.tempo;
      const promises = context.withinTransaction(() =>
        results.flatMap(({ track, ranges }) =>
          ranges.map((r) =>
            track.clearClipsInRange(
              selection.time_selection_start + r.start / beatsPerSecond,
              selection.time_selection_start + r.end / beatsPerSecond,
            ),
          ),
        ),
      );
      await Promise.all(promises);                     // await before the dialog closes
    });
  })(arg as ArrangementSelection).catch((e) => console.error(e)),
);
context.ui.registerContextMenuAction("AudioTrack.ArrangementSelection", "Strip Silence", "my-ext.strip-silence");
```

## 12. Modal dialog returning data (rename via a form)

See `webviews-and-ui.md` for the HTML side and `assets/webview-template.html` for a styled starter.

```ts
import modalInterface from "./interface.html";

context.commands.registerCommand("my-ext.rename-dialog", (arg: unknown) =>
  (async (handle: Handle) => {
    const clip = context.getObjectFromHandle(handle, Clip);
    const result = await context.ui.showModalDialog(
      `data:text/html,${encodeURIComponent(modalInterface)}`, 360, 240,
    );
    const { name } = JSON.parse(result);
    if (name) clip.name = name;
  })(arg as Handle),
);
```

## Units & gotchas recap

- All arrangement/clip times are in **beats**, not seconds. Convert with `beatsPerSecond = 60 / song.tempo` when working with audio analysis (seconds).
- Array indexing under `noUncheckedIndexedAccess` is `T | undefined` — use `!` only when you've checked length.
- Wrap async command bodies in `try/catch` (or `.catch`) — unhandled rejections vanish into the log at best.
