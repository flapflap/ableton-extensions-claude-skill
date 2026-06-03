# ableton-live-extensions — a Claude skill

A [Claude Code](https://claude.com/claude-code) / Claude skill for building, scaffolding, and debugging **Ableton Live extensions** with the official **Ableton Extensions SDK** (`@ableton-extensions/sdk`, 1.0.0-beta.0).

It teaches Claude the SDK's mental model and the non-obvious rules that are easy to get wrong from intuition — handles vs. objects, synchronous transactions wrapping async creation, the sandboxed filesystem, the non-contiguous `WarpMode` enum, context-menu scopes, WebView message passing, and the build/`.ablx` packaging flow.

This repo is packaged as a **Claude Code plugin marketplace**, so it can be installed with one command. The skill itself lives at `plugins/ableton-live-extensions/skills/ableton-live-extensions/`.

## What's inside

```
.
├── .claude-plugin/marketplace.json            # marketplace catalog
├── plugins/ableton-live-extensions/
│   ├── .claude-plugin/plugin.json             # plugin manifest
│   └── skills/ableton-live-extensions/
│       ├── SKILL.md                           # mental model, critical rules, workflow
│       ├── references/
│       │   ├── api-reference.md               # full object model: classes, methods, enums, interfaces
│       │   ├── project-setup.md               # scaffolding, dev loop, manifest, packaging
│       │   ├── webviews-and-ui.md             # modal dialogs, message passing, Ableton design guide
│       │   └── recipes.md                     # 12 copy-adaptable patterns from the official examples
│       └── assets/webview-template.html       # Live-themed dialog starter (WebKit + WebView2 bridge)
├── evals/evals.json                           # eval test set used to validate the skill
├── README.md  ·  LICENSE
```

## Install

**As a plugin (recommended).** In Claude Code:

```
/plugin marketplace add flapflap/ableton-extensions-claude-skill
/plugin install ableton-live-extensions@ableton-extensions
```

(Or from the CLI: `claude plugin marketplace add flapflap/ableton-extensions-claude-skill` then `claude plugin install ableton-live-extensions@ableton-extensions`.)

**As a standalone skill.** Copy just the skill folder straight into your skills directory — no clone needed (works with macOS/BSD and GNU `tar`):

```bash
mkdir -p ~/.claude/skills/ableton-live-extensions
curl -fsSL https://codeload.github.com/flapflap/ableton-extensions-claude-skill/tar.gz/refs/heads/main \
  | tar -xz --strip-components=5 -C ~/.claude/skills/ableton-live-extensions \
    'ableton-extensions-claude-skill-main/plugins/ableton-live-extensions/skills/ableton-live-extensions'
```

For a project-scoped install, swap the destination for `.claude/skills/ableton-live-extensions` inside your repo. Or, if you'd rather clone the whole repo, the skill lives at `plugins/ableton-live-extensions/skills/ableton-live-extensions/` — copy that folder wherever you keep skills.

Either way, the skill triggers automatically when you mention Ableton Live extensions, the SDK, or scripting Live (tracks, clips, devices, warp modes, context-menu actions, dialogs, etc.).

## Scope

Covers the offline/command-driven automation the SDK is built for. It is **not** about realtime audio, MIDI routing, control surfaces, or Max for Live — the SDK doesn't do those, and the skill says so.

## Credits

Distilled from the official Ableton Extensions SDK 1.0.0-beta.0 documentation and bundled examples. Not affiliated with or endorsed by Ableton.
