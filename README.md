# ableton-live-extensions — a Claude skill

A [Claude Code](https://claude.com/claude-code) / Claude skill for building, scaffolding, and debugging **Ableton Live extensions** with the official **Ableton Extensions SDK** (`@ableton-extensions/sdk`, 1.0.0-beta.0).

It teaches Claude the SDK's mental model and the non-obvious rules that are easy to get wrong from intuition — handles vs. objects, synchronous transactions wrapping async creation, the sandboxed filesystem, the non-contiguous `WarpMode` enum, context-menu scopes, WebView message passing, and the build/`.ablx` packaging flow.

## What's inside

```
ableton-live-extensions/
├── SKILL.md                       # mental model, critical rules, workflow
├── references/
│   ├── api-reference.md           # full object model: classes, methods, enums, interfaces
│   ├── project-setup.md           # scaffolding, dev loop, manifest, packaging
│   ├── webviews-and-ui.md         # modal dialogs, message passing, Ableton design guide
│   └── recipes.md                 # 12 copy-adaptable patterns from the official examples
└── assets/
    └── webview-template.html      # Live-themed dialog starter (WebKit + WebView2 bridge)
```

## Install

**Claude Code (personal skills):**

```bash
git clone https://github.com/<you>/ableton-live-extensions.git \
  ~/.claude/skills/ableton-live-extensions
```

Then restart Claude Code. The skill triggers automatically when you mention Ableton Live extensions, the SDK, or scripting Live (tracks, clips, devices, warp modes, context-menu actions, dialogs, etc.).

You can also drop the folder into a project at `.claude/skills/ableton-live-extensions/` to scope it to one repo.

## Scope

Covers the offline/command-driven automation the SDK is built for. It is **not** about realtime audio, MIDI routing, control surfaces, or Max for Live — the SDK doesn't do those, and the skill says so.

## Credits

Distilled from the official Ableton Extensions SDK 1.0.0-beta.0 documentation and bundled examples. Not affiliated with or endorsed by Ableton.
