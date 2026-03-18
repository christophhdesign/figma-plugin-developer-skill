# Figma Plugin Developer Skill for Claude Code


**Give Claude deep expertise in Figma plugin development**

[![Claude Code](https://img.shields.io/badge/Claude_Code-Skill-blueviolet?style=for-the-badge)](https://docs.anthropic.com/en/docs/claude-code)
[![Figma API](https://img.shields.io/badge/Figma_API-Latest-1abcfe?style=for-the-badge&logo=figma&logoColor=white)](https://www.figma.com/plugin-docs/)
[![License](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](#license)

</div>

---

## What is this?

A **Claude Code skill** that transforms Claude into an expert Figma plugin developer. Install it once, and Claude will automatically apply deep knowledge of the Plugin API, REST API, sandbox architecture, and UI best practices whenever you're working on Figma plugins.

```
  You                Claude Code              Figma Plugin
  ────               ───────────              ────────────
   │                      │                        │
   │  "Build a plugin     │                        │
   │   that swaps icons"  │                        │
   │─────────────────────>│                        │
   │                      │  ┌──────────────────┐  │
   │                      │  │  Skill activates  │  │
   │                      │  │  automatically    │  │
   │                      │  └──────────────────┘  │
   │                      │                        │
   │   manifest.json      │───────────────────────>│
   │   code.ts            │───────────────────────>│
   │   ui.html            │───────────────────────>│
   │                      │                        │
   │  Working plugin!     │                        │
   │<─────────────────────│                        │
```

## Coverage

The skill covers the **full Figma plugin development surface**:

```
┌──────────────────────────────────────────────────────────┐
│                    SKILL KNOWLEDGE MAP                    │
├──────────────────┬───────────────────┬───────────────────┤
│                  │                   │                   │
│  Plugin API      │  REST API         │  Architecture     │
│  ─────────       │  ────────         │  ────────────     │
│  Node tree       │  Authentication   │  Dual-context     │
│  Components      │  Files & images   │  Sandbox limits   │
│  Auto Layout     │  Components       │  Message passing  │
│  Grid Layout     │  Variables        │  manifest.json    │
│  Variables       │  Webhooks         │  TypeScript setup │
│  Text & fonts    │  Comments         │  Build tooling    │
│  Styles & paint  │  Pagination       │  UI patterns      │
│  Export          │  Rate limits      │  Theme support    │
│  Interactions    │  OAuth & PATs     │  Security         │
│                  │                   │                   │
├──────────────────┴───────────────────┴───────────────────┤
│  Editors: Figma  ·  FigJam  ·  Slides  ·  Buzz          │
└──────────────────────────────────────────────────────────┘
```

## Quick Start

### 1. Install the skill

Copy the skill files into your Claude Code skills directory:

```bash
# Create the skill directory
mkdir -p ~/.claude/skills/figma-plugin-dev

# Copy skill files
cp SKILL.md ~/.claude/skills/figma-plugin-dev/
cp plugin-api-reference.md ~/.claude/skills/figma-plugin-dev/
cp rest-api-reference.md ~/.claude/skills/figma-plugin-dev/
cp examples.md ~/.claude/skills/figma-plugin-dev/
```

### 2. Start building

Open Claude Code in your plugin project directory and start asking:

```
> Create a Figma plugin that finds all detached instances on the current page

> Add a UI panel that shows component usage stats with thumbnails

> Set up a TypeScript plugin project with esbuild

> Build a color audit tool that lists all unique colors used in the file
```

The skill activates automatically when Claude detects Figma plugin context.

## What's Inside

```
figma-plugin-developer-skill/
│
├── SKILL.md                   Core knowledge: architecture, APIs, patterns,
│                              best practices, gotchas, and deprecations
│
├── plugin-api-reference.md    Complete figma.* API reference — nodes,
│                              text, components, variables, layout, events
│
├── rest-api-reference.md      REST API endpoints — auth, files, images,
│                              components, webhooks, variables, pagination
│
├── examples.md                Battle-tested patterns — component swap,
│                              thumbnails, text replace, color audit, UI
│
└── README.md                  You are here
```

## Skill Highlights

### Dual-Context Architecture Awareness

Claude understands the sandbox/UI split and writes code accordingly:

```
┌─────────────────────────────────┐
│  UI Thread (ui.html)            │
│  ├─ DOM, fetch, Canvas          │
│  ├─ Web Workers                 │
│  └─ Full browser APIs           │
│          │                      │
│     postMessage (structured     │
│       clone only — no           │
│       functions or DOM nodes)   │
│          │                      │
│  Sandbox (code.js)              │
│  ├─ figma.* API                 │
│  ├─ No DOM, no fetch            │
│  ├─ No setInterval              │
│  └─ No btoa/atob                │
└─────────────────────────────────┘
```

### Common Gotcha Prevention

The skill encodes hard-won knowledge so you don't hit these pitfalls:

| Gotcha | What Claude Knows |
|--------|-------------------|
| Colors are 0-1, not 0-255 | Always divides hex by 255 |
| Fonts must be loaded first | Calls `loadFontAsync` before any text edit |
| Fills/strokes are readonly | Clones via `JSON.parse(JSON.stringify(...))` |
| No `fetch` in sandbox | Routes network calls through the UI iframe |
| No `btoa` in sandbox | Uses custom base64 encoder |
| `mainComponent` can be null | Uses `getMainComponentAsync()` + null check |
| PATs expire in 90 days | Implements token refresh flows |
| `resetOverrides()` deprecated | Uses `removeOverrides()` instead |

### Ready-Made Patterns

The skill includes complete, production-ready patterns for:

- **Component scanning & swapping** — with import caching and progress reporting
- **Thumbnail generation** — custom base64 encoding for the sandbox
- **Figma-native UI** — theme-aware CSS with Figma's design tokens
- **REST API integration** — pagination, rate limiting, token storage
- **TypeScript project setup** — esbuild, typings, tsconfig
- **Grid layout construction** — CSS Grid API patterns
- **Variable theming** — extended collections and mode overrides

## API Coverage Timeline

```
                              Covered by this skill
                    ─────────────────────────────────────
  2025              │                                   │
    Apr  PAT 90-day expiry, scoped permissions ──────── ●
    May  OAuth token refresh, Dev Mode webhooks ─────── ●
    Jul  Grid Layout, Noise effects, UI position ────── ●
    Oct  Buzz editor, fontStyle segments ────────────── ●
    Nov  Extended Variables, Grid HUG, removeOverrides  ●
  2026              │                                   │
    Jan  TextPath, TransformGroups, Complex Strokes ─── ●
                    │                                   │
                    ─────────────────────────────────────
```

## Contributing

Contributions are welcome! If you find a missing API, a new best practice, or a useful pattern:

1. Fork the repo
2. Add your changes to the appropriate file
3. Submit a PR with a clear description

**Guidelines:**
- Keep examples minimal and self-contained
- Include gotchas and edge cases you've encountered
- Reference the Figma API update/changelog when adding new APIs
- Test patterns against the latest Figma desktop app

## License

MIT License. See [LICENSE](LICENSE) for details.

---

<div align="center">

**Built for the [Claude Code](https://docs.anthropic.com/en/docs/claude-code) ecosystem**

*Give Claude the knowledge to build Figma plugins like a pro.*

</div>
