# Project Setup, Dev Loop & Packaging

## Prerequisites

- **Node.js ≥ 24.14.1** (official installer recommended).
- **Ableton Live Beta** that supports Extensions (from the SDK release page / Centercode).
- The **SDK distribution zip** `extensions-sdk-<version>.zip`, which contains: the project creator (`ableton-create-extension-*.tgz`), the SDK (`ableton-extensions-sdk-*.tgz`), the CLI (`ableton-extensions-cli-*.tgz`), the `examples/`, and the rendered `docs/` + `api/`.
- **Developer Mode** enabled in Live: *Preferences → Extensions → Developer Mode*. Without it, `npm start` cannot connect to Live.

## Create a new extension

Run the project creator inside a fresh folder (it scaffolds in place):

```bash
mkdir ~/Projects/my-extension && cd ~/Projects/my-extension
npx file:/path/to/extracted/ableton-create-extension-1.0.0-beta.0.tgz
```

It prompts for: **Extension name**, **Author**, the **Live application** (detected installs are listed — only Extension-capable ones; or enter a custom path), and **whether you need a UI** (yes adds a Vite-based webview setup).

Resulting layout:

```
my-extension/
├── .env                 # EXTENSION_HOST_PATH=…  (path to Live's Extension Host)
├── .gitignore
├── README.md
├── manifest.json
├── build.ts             # esbuild build script (editable)
├── package.json         # scripts: start, build, package
├── tsconfig.json
├── src/
│   └── extension.ts     # your entry point — exports activate()
├── vendor/              # the SDK + CLI tgz files
└── node_modules/
```

## manifest.json

```json
{
  "name": "my-extension",
  "author": "Your Name or Organisation",
  "version": "1.0.0",
  "entry": "dist/extension.js",
  "minimumApiVersion": "1.0.0"
}
```

`entry` is the single bundled file the host loads. `minimumApiVersion` gates which Live builds will run it.

## build.ts (esbuild)

The generated build bundles `src/extension.ts` to one CJS file. The `.html` text loader lets you `import html from "./interface.html"` for inline WebView markup.

```ts
import * as esbuild from "esbuild";
import * as fs from "node:fs";

const manifest = JSON.parse(fs.readFileSync("manifest.json", "utf8"));
const production = process.argv.includes("--production");

await esbuild.build({
  entryPoints: ["src/extension.ts"],
  outfile: manifest.entry,
  bundle: true,
  format: "cjs",
  platform: "node",
  sourcesContent: false,
  logLevel: "info",
  minify: production,
  sourcemap: !production,
  loader: { ".html": "text" },
});
```

For a `*.html` import to type-check, add `src/html.d.ts`:

```ts
declare module "*.html" {
  const content: string;
  export default content;
}
```

## tsconfig.json (strict baseline used by the examples)

```jsonc
{
  "compilerOptions": {
    "module": "nodenext",
    "target": "esnext",
    "moduleResolution": "nodenext",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedSideEffectImports": true,
    "moduleDetection": "force",
    "esModuleInterop": true,
    "noEmit": true,
    "rootDir": "./src"
  },
  "include": ["src/**/*"]
}
```

`noUncheckedIndexedAccess` means array access yields `T | undefined` — the examples use `!` (e.g. `arr[0]!`) where they know it's present.

## npm scripts & the dev loop

| Script | Does |
|---|---|
| `npm start` | Build (dev) + launch Live's Extension Host with your extension. Thin wrapper around `extensions-cli build --dev && extensions-cli run`. |
| `npm run build` | Production bundle → `dist/extension.js` (minified, no sourcemaps). |
| `npm run build:dev` | Dev bundle (sourcemaps, not minified). |
| `npm run package` | Production build, then produces a shareable `.ablx`. |

Dev cycle with Developer Mode on: edit code → restart `npm start` to pick up the rebuild. You do **not** relaunch Live. `npm start` reads `EXTENSION_HOST_PATH` from `.env`.

Run the CLI directly to override settings:

```bash
npx extensions-cli run --live "/Applications/Ableton Live 12.4 Beta.app"
npx extensions-cli run --inspect        # VS Code debugging (--inspect-brk)
npx extensions-cli run --storage-directory <path> --temp-directory <path>
```

`--live` accepts a `.app` (macOS), `.exe` (Windows), an install root, or a direct path to `ExtensionHostNodeModule.node`.

## Running the bundled examples

The examples have no `.env`, so pass `--live`. Run them **from where you extracted the zip** (copying elsewhere breaks `npm install`):

```bash
cd examples/context-menu
npm install
npm start -- --live "/Applications/Ableton Live Beta.app"
```

## Packaging & distribution (.ablx)

```bash
npm run package
# or call the CLI to control output / include assets:
npx extensions-cli package -o dist/my-extension.ablx
npx extensions-cli package -i assets templates/index.html
```

- **Always build before packaging** — `extensions-cli package` does not run your build.
- Included paths (files or dirs, recursive) must be **relative to the extension dir** and cannot escape it.
- A packaged extension = one JS file + `manifest.json` + any explicitly included assets.
- A user installs a `.ablx` by **dropping it onto the Extensions page** in Live's settings.

To use a different bundler, replace `build.ts` and the `build` script — `extensions-cli run`/`package` only care about the `entry` file in `manifest.json`.
