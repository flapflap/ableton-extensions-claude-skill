# API Reference — Ableton Extensions SDK 1.0.0-beta.0

Exact surface of `@ableton-extensions/sdk`. Signatures are from the SDK's generated docs. Generic classes are parameterized by API version, e.g. `Track<"1.0.0">`; the version flows from `initialize(activation, "1.0.0")`. Property getters shown below are also assignable when they represent writable Live state (e.g. `name`, `color`, `mute`, `solo`, `arm`, `warpMode`, `looping`, `loopStart/End`, `startMarker/endMarker`, `muted`, `tempo`); read-only structural getters (e.g. `tracks`, `parent`, `handle`) are not.

## Contents
- [Entry points](#entry-points)
- [ExtensionContext](#extensioncontext)
- [Application & Song](#application--song)
- [Tracks](#tracks)
- [Clips](#clips)
- [Clip slots & scenes](#clip-slots--scenes)
- [Take lanes & cue points](#take-lanes--cue-points)
- [Devices & mixers](#devices--mixers)
- [UI / Commands / Resources / Environment](#services)
- [Enums, interfaces, types](#enums-interfaces-types)

## Entry points

```ts
import { initialize, /* classes, enums, types… */ } from "@ableton-extensions/sdk";

// Live calls this when your extension loads.
export function activate(activation: ActivationContext): void;

// Returns the typed context. Pass the activation object + API version string.
function initialize(activation: ActivationContext, version: "1.0.0"): ExtensionContext<"1.0.0">;
```

`ActivationContext` has `{ hostApiVersion: string }`. The variable `EXTENSIONS_API_VERSIONS` lists supported versions.

## ExtensionContext

```ts
interface ExtensionContext<Version> {
  application: Application<Version>;
  commands:    Commands<Version>;
  environment: Environment<Version>;
  resources:   Resources<Version>;
  ui:          Ui<Version>;

  getObjectFromHandle<T extends DataModelObject<Version>>(
    handle: Handle,
    type: new (...args: never) => T,
  ): T;

  withinTransaction<T>(fn: () => T): T;   // SYNCHRONOUS callback; see SKILL.md rules
}
```

`withinProgressDialog` lives on `ui` (below), not on the context directly.

## Application & Song

```ts
class Application {
  readonly handle: Handle;
  get song(): Song;
  get parent(): DataModelObject | null;
}

class Song {
  readonly handle: Handle;
  get tempo(): number;                 // also settable
  get tracks(): Track[];               // includes audio + midi
  get returnTracks(): Track[];
  get mainTrack(): Track;
  get scenes(): Scene[];
  get cuePoints(): CuePoint[];
  get gridQuantization(): GridQuantization;
  get gridIsTriplet(): boolean;
  get rootNote(): number;
  get scaleName(): string;
  get scaleMode(): boolean;
  get scaleIntervals(): number[];
  get parent(): DataModelObject | null;

  createAudioTrack(): Promise<AudioTrack>;
  createMidiTrack(): Promise<MidiTrack>;
  deleteTrack(track: Track): Promise<void>;
  duplicateTrack(track: Track): Promise<Track>;

  createScene(index: number): Promise<Scene>;
  deleteScene(scene: Scene): Promise<void>;
  duplicateScene(scene: Scene): Promise<Scene>;

  createCuePoint(time: number): Promise<CuePoint>;
  deleteCuePoint(cuePoint: CuePoint): Promise<void>;
}
```

## Tracks

`Track` is the base for `AudioTrack` and `MidiTrack`. Shared members:

```ts
class Track {                          // also AudioTrack / MidiTrack
  readonly handle: Handle;
  get name(): string;                  // settable
  get mute(): boolean;                 // settable
  get solo(): boolean;                 // settable
  get arm(): boolean;                  // settable
  get mutedViaSolo(): boolean;
  get groupTrack(): Track | null;
  get mixer(): TrackMixer;
  get devices(): Device[];
  get clipSlots(): ClipSlot[];         // Session view slots
  get arrangementClips(): Clip[];      // Arrangement view clips
  get takeLanes(): TakeLane[];
  get parent(): DataModelObject | null;

  clearClipsInRange(startTime: number, endTime: number): Promise<void>;  // times in beats
  deleteClip(clip: Clip): Promise<void>;
  createTakeLane(): Promise<TakeLane>;

  insertDevice(deviceName: string, index: number): Promise<Device>;
  deleteDevice(device: Device): Promise<void>;
  duplicateDevice(device: Device): Promise<Device>;
}
```

Type-specific clip creation:

```ts
class AudioTrack extends Track {
  createAudioClip(args: {
    filePath: string;                  // use the path returned by importIntoProject
    startTime: number;                 // arrangement position, in beats
    duration?: number;                 // in beats
    isWarped?: boolean;
    loopSettings?: ClipLoopSettings;
  }): Promise<AudioClip>;
}

class MidiTrack extends Track {
  createMidiClip(startTime: number, duration: number): Promise<MidiClip>;  // beats
}
```

## Clips

`Clip` is the base for `AudioClip` and `MidiClip`. Shared:

```ts
class Clip {                           // also AudioClip / MidiClip
  readonly handle: Handle;
  get name(): string;                  // settable
  get color(): number;                 // settable
  get muted(): boolean;                // settable
  get startTime(): number;
  get endTime(): number;
  get duration(): number;
  get startMarker(): number;           // settable
  get endMarker(): number;             // settable
  get looping(): boolean;              // settable
  get loopStart(): number;             // settable
  get loopEnd(): number;               // settable
  get parent(): DataModelObject | null;
}

class AudioClip extends Clip {
  get filePath(): string;
  get warping(): boolean;              // settable
  get warpMode(): WarpMode;            // settable — see enum; not contiguous
  get warpMarkers(): WarpMarker[];
}

class MidiClip extends Clip {
  get notes(): NoteDescription[];      // settable — assign an array to replace notes
}
```

## Clip slots & scenes

```ts
class ClipSlot {
  readonly handle: Handle;
  get clip(): Clip | null;
  get parent(): DataModelObject | null;

  createAudioClip(args: {
    filePath: string;
    isWarped?: boolean;
    loopSettings?: ClipLoopSettings;
  }): Promise<AudioClip>;              // note: no startTime (it's a session slot)
  createMidiClip(length: number): Promise<MidiClip>;
  deleteClip(): Promise<void>;
}

class Scene {
  readonly handle: Handle;
  get name(): string;                  // settable
  get tempo(): number;
  get signatureNumerator(): number;
  get signatureDenominator(): number;
  get parent(): DataModelObject | null;
}
```

## Take lanes & cue points

```ts
class TakeLane {                       // a comping take lane on a track
  readonly handle: Handle;
  get name(): string;
  get parent(): DataModelObject | null;
  createMidiClip(startTime: number, duration: number): Promise<MidiClip>; // on MIDI-parented lanes
}

class CuePoint {
  readonly handle: Handle;
  // time/name accessible; locate via Song.cuePoints
}
```

## Devices & mixers

`Device` is the base for `RackDevice`, `DrumRack`, `Simpler`. `DataModelObject` is the root base of everything.

```ts
class Device     { readonly handle: Handle; /* className "Device" */ }
class RackDevice extends Device {}
class DrumRack   extends Device { /* className "DrumRackDevice" */ }
class Simpler    extends Device { get sample(): Sample | null; /* className "Simpler" */ }
class Chain      {}                    // rack chains
class DrumChain  extends Chain {}
class DeviceParameter { readonly handle: Handle; /* a controllable parameter */ }
class TrackMixer { readonly handle: Handle; /* className "MixerDevice" */ }
class ChainMixer {}
class Sample     { readonly handle: Handle; }
```

Inspect a specific device/parameter's exact members in the generated `api/` HTML when you need them; the beta surface for devices is thin (insert/duplicate/delete via the owning `Track`).

## Services

```ts
class Commands {
  registerCommand(commandId: string, callback: (...args: unknown[]) => void): void;
  executeCommand(commandId: string, ...args: unknown[]): void;
}

class Ui {
  registerContextMenuAction(
    scope: ContextMenuScope,            // see type below
    title: string,
    commandId: string,
  ): Promise<() => Promise<void>>;       // resolves to an unregister fn

  showModalDialog(url: string, width: number, height: number): Promise<string>;

  withinProgressDialog(
    text: string,
    options: { progress?: number },
    callback: (
      update: (updateText: string, progress?: number) => Promise<void>, // progress 0–100
      abortSignal: AbortSignal,
    ) => Promise<unknown>,
  ): Promise<unknown>;
}

class Resources {
  importIntoProject(filePath: string): Promise<string>;   // returns imported copy's path
  renderPreFxAudio(track: AudioTrack, startTime: number, endTime: number): Promise<string>; // returns WAV path in tempDir; times in beats
}

class Environment {
  get storageDirectory(): string | undefined;  // persistent, read/write
  get tempDirectory(): string | undefined;      // scratch, read/write, may be cleaned
  get language(): string | undefined;           // e.g. "EN", "DE", "JA"
}
```

## Enums, interfaces, types

```ts
enum WarpMode { Beats = 0, Tones = 1, Texture = 2, Repitch = 3, Complex = 4, ComplexPro = 6 } // no 5

enum GridQuantization {
  NoGrid = 0, EightBars = 1, FourBars = 2, TwoBars = 3, Bar = 4,
  Half = 5, Quarter = 6, Eighth = 7, Sixteenth = 8, ThirtySecond = 9,
}

interface Handle { id: bigint; }

interface ArrangementSelection {
  time_selection_start: number;   // beats
  time_selection_end: number;     // beats
  selected_lanes: Handle[];       // tracks / take lanes
}

interface ClipSlotSelection { selected_clip_slots: Handle[]; }

interface ClipLoopSettings {
  looping: boolean;
  startMarker: number;
  endMarker: number;
  loopStart: number;
  loopEnd: number;
}

interface WarpMarker { beatTime: number; sampleTime: number; }

type NoteDescription = {
  pitch: number;            // MIDI note number
  startTime: number;        // beats
  duration: number;         // beats
  velocity?: number;
  muted?: boolean;
  probability?: number;
  releaseVelocity?: number;
  selected?: boolean;
  velocityDeviation?: number;
};

type ContextMenuScope =     // for API version "1.0.0"
  | "AudioClip" | "MidiClip" | "AudioTrack" | "MidiTrack"
  | "ClipSlot"  | "Scene"    | "Simpler"    | "Sample" | "DrumRack"
  | "ClipSlotSelection"
  | "AudioTrack.ArrangementSelection" | "MidiTrack.ArrangementSelection";
```

## Polymorphism cheatsheet

- Unknown clip type → resolve with `Clip`, narrow with `instanceof AudioClip` / `MidiClip`.
- Unknown track type → resolve with `Track`, narrow with `instanceof AudioTrack` / `MidiTrack`.
- Mixed selection (arrangement/clip-slot) → resolve each handle with `DataModelObject`, then `.filter(o => o instanceof Track || o instanceof TakeLane)`.
- A `TakeLane`'s parent tells you audio vs midi: `lane.parent instanceof MidiTrack`.
