---
name: figma-plugin-dev
description: Expert Figma plugin developer with deep knowledge of the Plugin API, REST API, sandbox architecture, Grid/Auto Layout systems, Variables/theming, and UI best practices. Use when building, debugging, or extending Figma plugins or widgets, working with manifest.json, code.js/code.ts, ui.html, or any figma.* API code. Covers Figma, FigJam, Slides, and Buzz editor types.
---

# Figma Plugin Development

You are an expert Figma plugin developer. Apply this knowledge whenever building, modifying, or debugging Figma plugins.

## Plugin Architecture

Figma plugins run in a **dual-context** architecture:

```
┌─────────────────────────────┐
│  UI Thread (ui.html)        │  ← iframe, full browser APIs, DOM, fetch
│  postMessage ↕              │
│  Sandbox (code.js)          │  ← No DOM, limited JS, access to figma.*
└─────────────────────────────┘
```

- **Sandbox** (`code.js`/`code.ts`): Runs in a minimal JS environment. Has access to `figma.*` global. No DOM, no `window`, no `fetch`, no `XMLHttpRequest`. Use `setTimeout` but not `setInterval`.
- **UI** (`ui.html`): Runs in an `<iframe>`. Full browser APIs including DOM, `fetch`, Canvas, Web Workers. Communicates with sandbox via `postMessage`.

### manifest.json

```json
{
  "name": "Plugin Name",
  "id": "unique-id",
  "api": "1.0.0",
  "main": "code.js",
  "ui": "ui.html",
  "editorType": ["figma"],
  "capabilities": [],
  "enableProposedApi": false,
  "networkAccess": {
    "allowedDomains": ["api.figma.com"],
    "reasoning": "Why network access is needed"
  }
}
```

Key fields:
- `editorType`: `["figma"]`, `["figjam"]`, `["slides"]`, `["buzz"]`, or combinations
- `networkAccess.allowedDomains`: Required for any network calls from UI. Use `["none"]` if no network needed, `["*"]` for unrestricted
- `capabilities`: `["inspect"]` for Dev Mode plugins, `["codegen"]` for code generators
- `enableProposedApi`: Enable experimental APIs (may break)

### Message Passing

**Sandbox → UI:**
```js
figma.ui.postMessage({ type: 'results', data: payload });
```

**UI → Sandbox:**
```js
parent.postMessage({ pluginMessage: { type: 'action', data: payload } }, '*');
```

**Listening (sandbox):**
```js
figma.ui.onmessage = async (msg) => {
  if (msg.type === 'action') { /* handle */ }
};
```

**Listening (UI):**
```js
window.onmessage = (event) => {
  const msg = event.data.pluginMessage;
  if (!msg) return;
  if (msg.type === 'results') { /* handle */ }
};
```

**CRITICAL**: Messages are serialized via structured clone. You cannot send functions, DOM nodes, class instances, or circular references. Plain objects, arrays, strings, numbers, booleans, `null`, `Uint8Array`, and `ArrayBuffer` are safe.

## Showing the UI

```js
figma.showUI(__html__, {
  width: 400,
  height: 500,
  themeColors: true,   // Inject Figma's CSS variables for theme support
  visible: true,       // Set false for headless plugins
  position: { x: 0, y: 0 },
});
```

Resize at runtime: `figma.ui.resize(newWidth, newHeight)`

## Node Tree & Types

Figma's document is a tree:

```
Document
 └─ Page (PageNode)
     ├─ Frame (FrameNode)
     │   ├─ Rectangle (RectangleNode)
     │   ├─ Text (TextNode)
     │   └─ Instance (InstanceNode)
     ├─ Component (ComponentNode)
     ├─ ComponentSet (ComponentSetNode)  ← variant container
     ├─ Group (GroupNode)
     ├─ Section (SectionNode)
     ├─ Vector / Star / Ellipse / Polygon / Line / BooleanOperation
     └─ Slice / Connector / Stamp / Widget
```

### Key Node Properties (shared)

| Property | Type | Notes |
|----------|------|-------|
| `id` | `string` | Unique within file |
| `name` | `string` | Layer name |
| `type` | `string` | e.g. `'FRAME'`, `'TEXT'` |
| `parent` | `BaseNode \| null` | Parent node |
| `children` | `ReadonlyArray<SceneNode>` | Only on container nodes |
| `visible` | `boolean` | Visibility |
| `locked` | `boolean` | Lock state |
| `opacity` | `number` | 0–1 |
| `x`, `y` | `number` | Position relative to parent |
| `width`, `height` | `number` | Dimensions |
| `rotation` | `number` | Degrees |
| `fills` | `ReadonlyArray<Paint>` | Fill paints |
| `strokes` | `ReadonlyArray<Paint>` | Stroke paints |
| `effects` | `ReadonlyArray<Effect>` | Drop shadow, blur, etc. |
| `constraints` | `Constraints` | Horizontal/vertical constraints |
| `layoutMode` | `'NONE' \| 'HORIZONTAL' \| 'VERTICAL'` | Auto-layout direction |

### Traversal

```js
// All instances on current page
const instances = figma.currentPage.findAllWithCriteria({ types: ['INSTANCE'] });

// Find by name
const node = figma.currentPage.findOne(n => n.name === 'Button');

// Find all text nodes
const texts = figma.currentPage.findAll(n => n.type === 'TEXT');

// Walk children manually
function walk(node, callback) {
  callback(node);
  if ('children' in node) {
    for (const child of node.children) walk(child, callback);
  }
}
```

`findAllWithCriteria` is faster than `findAll` with a filter — prefer it when filtering by type.

## Components & Instances

```
ComponentSetNode (variant group, e.g. "Button")
 ├─ ComponentNode (variant: "State=Default, Size=M")
 ├─ ComponentNode (variant: "State=Hover, Size=M")
 └─ ComponentNode (variant: "State=Default, Size=L")
```

- `component.key` — globally unique key (persists across files when published)
- `instance.mainComponent` — resolves the source ComponentNode (may be `null` if missing)
- `await instance.getMainComponentAsync()` — preferred async version, resolves remote components
- `instance.swapComponent(newComponent)` — swap to a different component
- `await figma.importComponentByKeyAsync(key)` — import from a library by key
- `await figma.importComponentSetByKeyAsync(key)` — import variant set by key
- `component.createInstance()` — create a new instance of the component

### Variant Properties

```js
// On a ComponentSetNode:
const variantGroupProperties = componentSet.variantGroupProperties;
// { "Size": { values: ["S", "M", "L"] }, "State": { values: ["Default", "Hover"] } }

// On an InstanceNode — change variant:
instance.setProperties({ "Size": "L", "State": "Hover" });
```

## Text

**Always load fonts before modifying text:**

```js
const textNode = figma.createText();
await figma.loadFontAsync({ family: "Inter", style: "Regular" });
textNode.characters = "Hello World";
textNode.fontSize = 16;
textNode.fills = [{ type: 'SOLID', color: { r: 0, g: 0, b: 0 } }];
```

For mixed-style text, load all needed fonts first, then use range methods:
```js
textNode.setRangeFontSize(0, 5, 24);       // First 5 chars at 24px
textNode.setRangeFills(0, 5, [solidPaint]); // First 5 chars colored
```

**GOTCHA**: `textNode.fontName` throws if the text has mixed fonts. Use `textNode.getRangeFontName(0, 1)` for safety, or load all fonts via `textNode.getRangeAllFontNames(0, textNode.characters.length)`.

## Styles & Paint

```js
// Solid color (RGB 0–1, NOT 0–255)
const solidFill = { type: 'SOLID', color: { r: 1, g: 0.4, b: 0.3 }, opacity: 1 };

// Gradient
const gradientFill = {
  type: 'GRADIENT_LINEAR',
  gradientStops: [
    { position: 0, color: { r: 1, g: 0, b: 0, a: 1 } },
    { position: 1, color: { r: 0, g: 0, b: 1, a: 1 } },
  ],
  gradientTransform: [[1, 0, 0], [0, 1, 0]],
};

// Image fill
const imageHash = figma.createImage(imageBytes).hash;
const imageFill = { type: 'IMAGE', scaleMode: 'FILL', imageHash };

// Apply fills (REPLACES all fills — clone first to preserve existing)
node.fills = [solidFill];
```

**CRITICAL**: Colors use 0–1 range. Convert from hex: `r = 0xE5 / 255`.

## Auto Layout

```js
const frame = figma.createFrame();
frame.layoutMode = 'VERTICAL';          // or 'HORIZONTAL'
frame.primaryAxisAlignItems = 'CENTER'; // main axis: MIN | CENTER | MAX | SPACE_BETWEEN
frame.counterAxisAlignItems = 'CENTER'; // cross axis: MIN | CENTER | MAX
frame.itemSpacing = 12;                 // gap between children
frame.paddingTop = 16;
frame.paddingBottom = 16;
frame.paddingLeft = 16;
frame.paddingRight = 16;
frame.primaryAxisSizingMode = 'AUTO';   // AUTO (hug) or FIXED
frame.counterAxisSizingMode = 'AUTO';
```

Children in auto-layout use `layoutAlign` and `layoutGrow`:
```js
child.layoutAlign = 'STRETCH';  // fill container width (in vertical layout)
child.layoutGrow = 1;           // expand to fill remaining space
```

## Grid Layout (CSS Grid)

Figma supports CSS Grid-style layouts on frames (added 2025):

```js
const grid = figma.createFrame();
grid.layoutMode = 'GRID';
grid.gridRowCount = 3;
grid.gridColumnCount = 4;
grid.gridRowGap = 8;
grid.gridColumnGap = 8;
grid.gridRowSizes = [{ type: 'FIXED', value: 100 }, { type: 'FLEX', value: 1 }];
grid.gridColumnSizes = [{ type: 'FIXED', value: 200 }, { type: 'FLEX', value: 2 }];
// Grid frames also support 'HUG' for layoutSizingHorizontal/Vertical

// Position children in the grid
child.gridRowSpan = 2;             // span 2 rows
child.gridColumnSpan = 1;
child.gridRowAnchorIndex = 0;     // start at row 0
child.gridColumnAnchorIndex = 1;  // start at column 1
child.gridChildHorizontalAlign = 'CENTER'; // MIN | CENTER | MAX | STRETCH
child.gridChildVerticalAlign = 'CENTER';
child.setGridChildPosition(row, column); // shorthand
grid.appendChildAt(node, row, column);   // append at specific grid cell
```

## Export

```js
const pngBytes = await node.exportAsync({ format: 'PNG', constraint: { type: 'SCALE', value: 2 } });
const svgString = await node.exportAsync({ format: 'SVG' });
const jpgBytes = await node.exportAsync({ format: 'JPG', constraint: { type: 'HEIGHT', value: 120 } });
const pdfBytes = await node.exportAsync({ format: 'PDF' });
```

## Storage & Persistence

```js
// Per-plugin persistent storage (survives plugin restarts)
await figma.clientStorage.setAsync('key', value);
const value = await figma.clientStorage.getAsync('key');
await figma.clientStorage.deleteAsync('key');
const keys = await figma.clientStorage.keysAsync();
```

Values must be JSON-serializable. Size limit: ~1MB per key.

## Notifications

```js
figma.notify('Success message');
figma.notify('Error occurred', { error: true });
figma.notify('Processing...', { timeout: Infinity }); // Returns handle
// handle.cancel(); to dismiss
```

## Selection & Viewport

```js
// Read selection
const selected = figma.currentPage.selection; // ReadonlyArray<SceneNode>

// Set selection
figma.currentPage.selection = [node1, node2];

// Scroll to nodes
figma.viewport.scrollAndZoomIntoView([node]);

// Current viewport
const { x, y, width, height } = figma.viewport.bounds;
const zoom = figma.viewport.zoom;
```

## Deprecations & Breaking Changes

- **`InstanceNode.resetOverrides()`** — DEPRECATED (Update 120, Nov 2025). Use `instance.removeOverrides()` instead
- **GridTrackSize setter** — type only auto-sets to `'FIXED'` from `'HUG'` now (Update 120)
- **REST API: `GET /v2/teams/:team_id/webhooks`** — DEPRECATED. Use `GET /v2/webhooks?context=team&context_id=:team_id`
- **REST API: PATs** — Maximum expiration reduced to 90 days (non-expiring PATs prohibited, Apr 2025)
- **REST API: `files:read` scope** — Deprecated. Use specific scopes: `file_content:read`, `file_comments:read`, etc.
- **REST API: OAuth refresh** — Use `POST /v1/oauth/token` (not the old `/v1/oauth/refresh` endpoint)

## Best Practices

### Performance
- **Batch reads before writes** — Figma flushes layout after writes; interleaving is expensive
- **Use `findAllWithCriteria`** over `findAll` with callback for type-based filtering
- **Limit `exportAsync`** — expensive; generate thumbnails at small sizes (e.g. `HEIGHT: 120`)
- **Avoid deep recursion** on large files — use iterative traversal or limit scope to selection/page
- **Send progress messages** to UI during long operations (e.g. every 10 nodes)
- **Check `node.removed`** before operating on nodes — they may have been deleted between async calls

### Error Handling
- Wrap all `await` calls in try/catch — nodes can be deleted, components missing, fonts unavailable
- `getMainComponentAsync()` can return `null` for deleted/inaccessible components
- `importComponentByKeyAsync` throws if the library isn't published or access is missing
- Always verify `node.parent` exists before manipulating tree position
- Check for `429 Too Many Requests` on REST API calls and implement exponential backoff

### Security
- **Never hardcode tokens** — store PATs using `figma.clientStorage` and pass to UI via `postMessage`
- **PATs expire in ≤90 days** — implement token refresh flows; non-expiring PATs are no longer allowed
- **Use scoped permissions** — prefer `file_content:read` over the deprecated `files:read`
- **Restrict `networkAccess.allowedDomains`** — use specific domains, not `["*"]`, unless necessary

### UI Development
- Use `themeColors: true` and Figma's CSS variables for light/dark theme support
- Keep the UI single-file (`ui.html`) with inline CSS/JS for simplicity, or use a bundler for complex UIs
- Design for the compact plugin panel — typical width 300–520px
- Use Figma's design system tokens: `--figma-color-bg`, `--figma-color-text`, `--figma-color-border`
- Handle `onmessage` defensively — always check `msg.type`
- Use `figma.ui.getPosition()` to read the current UI panel position

### Plugin Structure Pattern
```js
// code.js — clean message router pattern
figma.showUI(__html__, { width: 400, height: 500, themeColors: true });

figma.ui.onmessage = async (msg) => {
  switch (msg.type) {
    case 'action-a': await handleA(msg); break;
    case 'action-b': await handleB(msg); break;
    case 'close': figma.closePlugin(); break;
  }
};
```

### TypeScript

Use Figma's type definitions for full IntelliSense (now ships with docstrings as of May 2025):
```bash
npm install --save-dev @figma/plugin-typings
```

`tsconfig.json`:
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020"],
    "strict": true,
    "typeRoots": ["./node_modules/@figma/plugin-typings"]
  }
}
```

For UI code, create a separate `tsconfig.ui.json` with `"lib": ["ES2020", "DOM"]`.

## Variables & Styles API

```js
// Get all local variables
const variables = await figma.variables.getLocalVariablesAsync();

// Create a variable collection
const collection = figma.variables.createVariableCollection('Colors');

// Create a variable
const variable = figma.variables.createVariable('primary', collection, 'COLOR');
variable.setValueForMode(collection.modes[0].modeId, { r: 0.2, g: 0.4, b: 1 });

// Bind a variable to a node property
node.setBoundVariable('fills', 0, variable);
```

### Extended Variable Collections (Theming)

Extended collections enable theme overrides on top of library collections (added Nov 2025):

```js
// Extend a library variable collection for local theming
const extended = await figma.variables.extendLibraryCollectionByKeyAsync(collectionKey, 'My Theme');

// Override a variable value in the extended collection
variable.setValueForMode(modeId, newValue); // Sets override in extended collection context

// Read overrides
const overrides = extended.variableOverrides; // Map of variable overrides

// Remove overrides
variable.removeOverrideForMode(modeId);
extended.removeOverridesForVariable(variable);

// Read values by mode for a specific collection
const values = await variable.valuesByModeForCollectionAsync(collection);

// Get root collection ID for extended collections
const rootId = extended.rootVariableCollectionId;
```

## Additional Resources

- For detailed Plugin API reference, see [plugin-api-reference.md](plugin-api-reference.md)
- For REST API reference, see [rest-api-reference.md](rest-api-reference.md)
- For code examples and patterns, see [examples.md](examples.md)
