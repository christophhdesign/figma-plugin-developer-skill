# Figma Plugin Examples & Patterns

## Scan & Swap Components Pattern

Common pattern: scan the page for instances, display them in the UI, let the user pick targets, then swap.

### Sandbox (code.js)
```js
figma.showUI(__html__, { width: 480, height: 560, themeColors: true });

figma.ui.onmessage = async (msg) => {
  switch (msg.type) {
    case 'scan': await handleScan(); break;
    case 'swap': await handleSwap(msg.mappings); break;
    case 'close': figma.closePlugin(); break;
  }
};

async function handleScan() {
  const instances = figma.currentPage.findAllWithCriteria({ types: ['INSTANCE'] });
  const groups = {};

  for (const node of instances) {
    const comp = await node.getMainComponentAsync();
    if (!comp) continue;

    const key = comp.key;
    if (!groups[key]) {
      groups[key] = {
        key,
        name: comp.name,
        setName: comp.parent?.type === 'COMPONENT_SET' ? comp.parent.name : null,
        isRemote: comp.remote,
        count: 0,
      };
    }
    groups[key].count++;
  }

  figma.ui.postMessage({
    type: 'scan-results',
    components: Object.values(groups),
  });
}

async function handleSwap(mappings) {
  let swapped = 0;
  const importCache = {};

  // Pre-import targets
  for (const { targetKey } of mappings) {
    if (!targetKey || importCache[targetKey]) continue;
    try {
      importCache[targetKey] = await figma.importComponentByKeyAsync(targetKey);
    } catch {
      try {
        const set = await figma.importComponentSetByKeyAsync(targetKey);
        importCache[targetKey] = set.children[0] || null;
      } catch { importCache[targetKey] = null; }
    }
  }

  // Build source→target map
  const map = {};
  for (const { sourceKey, targetKey } of mappings) {
    if (importCache[targetKey]) map[sourceKey] = importCache[targetKey];
  }

  // Walk and swap
  const all = figma.currentPage.findAllWithCriteria({ types: ['INSTANCE'] });
  for (const node of all) {
    const comp = await node.getMainComponentAsync();
    if (comp && map[comp.key]) {
      node.swapComponent(map[comp.key]);
      swapped++;
    }
  }

  figma.notify(`Swapped ${swapped} instance${swapped === 1 ? '' : 's'}`);
  figma.ui.postMessage({ type: 'swap-done', swapped });
}
```

## Thumbnail Generation

Export small previews for UI display:
```js
// Base64 encoder for sandbox (no btoa available)
function uint8ArrayToBase64(bytes) {
  const chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/';
  let result = '';
  for (let i = 0; i < bytes.length; i += 3) {
    const a = bytes[i], b = bytes[i + 1] || 0, c = bytes[i + 2] || 0;
    result += chars[a >> 2];
    result += chars[((a & 3) << 4) | (b >> 4)];
    result += (i + 1 < bytes.length) ? chars[((b & 15) << 2) | (c >> 6)] : '=';
    result += (i + 2 < bytes.length) ? chars[c & 63] : '=';
  }
  return result;
}

async function generateThumbnail(node) {
  const bytes = await node.exportAsync({
    format: 'PNG',
    constraint: { type: 'HEIGHT', value: 120 },
  });
  return uint8ArrayToBase64(bytes);
}
```

Display in UI:
```html
<img src="data:image/png;base64,${base64String}" />
```

## Detached Instance Detection

Find frames that were detached from components:
```js
function findDetached(node) {
  const results = [];
  if ('detachedInfo' in node && node.detachedInfo) {
    results.push({
      id: node.id,
      name: node.name,
      componentKey: node.detachedInfo.componentKey || null,
      componentId: node.detachedInfo.componentId || null,
    });
  }
  if ('children' in node) {
    for (const child of node.children) results.push(...findDetached(child));
  }
  return results;
}

const detached = findDetached(figma.currentPage);
```

## Batch Text Replacement

Replace text across the file:
```js
async function replaceText(find, replace) {
  const textNodes = figma.currentPage.findAllWithCriteria({ types: ['TEXT'] });
  let count = 0;

  for (const node of textNodes) {
    if (!node.characters.includes(find)) continue;

    // Load ALL fonts used in this text node
    const fonts = node.getRangeAllFontNames(0, node.characters.length);
    for (const font of fonts) {
      await figma.loadFontAsync(font);
    }

    node.characters = node.characters.replaceAll(find, replace);
    count++;
  }
  return count;
}
```

## Color Audit / Lint

Find all unique colors used across the page:
```js
function extractColors(node) {
  const colors = new Set();
  if ('fills' in node && Array.isArray(node.fills)) {
    for (const fill of node.fills) {
      if (fill.type === 'SOLID' && fill.visible !== false) {
        const { r, g, b } = fill.color;
        colors.add(JSON.stringify({ r: Math.round(r * 255), g: Math.round(g * 255), b: Math.round(b * 255) }));
      }
    }
  }
  if ('children' in node) {
    for (const child of node.children) {
      for (const c of extractColors(child)) colors.add(c);
    }
  }
  return colors;
}

const allColors = extractColors(figma.currentPage);
const parsed = [...allColors].map(c => JSON.parse(c));
```

## Selection-Based Plugin

Operate on what the user has selected:
```js
figma.ui.onmessage = async (msg) => {
  if (msg.type === 'apply') {
    const selection = figma.currentPage.selection;
    if (selection.length === 0) {
      figma.notify('Please select at least one layer', { error: true });
      return;
    }

    for (const node of selection) {
      if ('fills' in node) {
        node.fills = [{ type: 'SOLID', color: msg.color }];
      }
    }

    figma.notify(`Applied to ${selection.length} layer${selection.length > 1 ? 's' : ''}`);
  }
};
```

## UI Pattern: Figma-Native Look

```html
<!DOCTYPE html>
<html>
<head>
<style>
  * { margin: 0; padding: 0; box-sizing: border-box; }
  html, body {
    height: 100%;
    font-family: Inter, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
    font-size: 11px;
    /* Figma theme variables (when themeColors: true) */
    color: var(--figma-color-text, #333);
    background: var(--figma-color-bg, #fff);
  }

  /* Figma-style button */
  .btn-primary {
    background: var(--figma-color-bg-brand, #0D99FF);
    color: #fff;
    border: none;
    border-radius: 6px;
    padding: 8px 16px;
    font-size: 11px;
    font-weight: 600;
    cursor: pointer;
    width: 100%;
  }
  .btn-primary:hover { opacity: 0.9; }
  .btn-primary:disabled { opacity: 0.4; cursor: not-allowed; }

  /* Figma-style input */
  .input {
    width: 100%;
    padding: 7px 8px;
    border: 1px solid var(--figma-color-border, #e5e5e5);
    border-radius: 4px;
    font-size: 11px;
    background: var(--figma-color-bg, #fff);
    color: var(--figma-color-text, #333);
    outline: none;
  }
  .input:focus {
    border-color: var(--figma-color-border-brand, #0D99FF);
    box-shadow: 0 0 0 1px var(--figma-color-border-brand, #0D99FF);
  }

  /* Section header */
  .section-title {
    font-size: 11px;
    font-weight: 600;
    color: var(--figma-color-text-secondary, #999);
    text-transform: uppercase;
    letter-spacing: 0.5px;
    margin-bottom: 8px;
  }

  /* Divider */
  .divider {
    height: 1px;
    background: var(--figma-color-border, #e5e5e5);
    margin: 12px 0;
  }

  /* Scrollable content area */
  .content {
    flex: 1;
    overflow-y: auto;
    padding: 12px;
  }

  /* Footer with sticky action */
  .footer {
    padding: 12px;
    border-top: 1px solid var(--figma-color-border, #e5e5e5);
    flex-shrink: 0;
  }
</style>
</head>
<body style="display: flex; flex-direction: column;">
  <div class="content">
    <!-- Plugin content here -->
  </div>
  <div class="footer">
    <button class="btn-primary" id="action">Apply</button>
  </div>

  <script>
    // Send messages to sandbox
    function post(type, data = {}) {
      parent.postMessage({ pluginMessage: { type, ...data } }, '*');
    }

    // Receive messages from sandbox
    window.onmessage = (event) => {
      const msg = event.data.pluginMessage;
      if (!msg) return;
      // Handle message types...
    };

    document.getElementById('action').onclick = () => post('apply');
  </script>
</body>
</html>
```

## Progress Reporting Pattern

For long operations, report progress to keep the UI responsive:

```js
// Sandbox
async function processNodes(nodes) {
  for (let i = 0; i < nodes.length; i++) {
    await processOneNode(nodes[i]);

    // Report every 10 nodes or on the last one
    if (i % 10 === 0 || i === nodes.length - 1) {
      figma.ui.postMessage({
        type: 'progress',
        current: i + 1,
        total: nodes.length,
      });
    }
  }
}
```

```html
<!-- UI -->
<div id="progress" style="display:none;">
  <div class="progress-bar">
    <div class="progress-fill" id="fill"></div>
  </div>
  <span id="progress-text">0%</span>
</div>

<script>
window.onmessage = (event) => {
  const msg = event.data.pluginMessage;
  if (msg.type === 'progress') {
    document.getElementById('progress').style.display = 'block';
    const pct = Math.round((msg.current / msg.total) * 100);
    document.getElementById('fill').style.width = pct + '%';
    document.getElementById('progress-text').textContent = `${msg.current}/${msg.total}`;
  }
};
</script>
```

## REST API: Fetch Component Metadata from UI

```js
// In ui.html — fetch all components from a library file
async function getLibraryComponents(fileKey, token) {
  const resp = await fetch(
    `https://api.figma.com/v1/files/${fileKey}/components`,
    { headers: { 'X-Figma-Token': token } }
  );
  if (!resp.ok) throw new Error(`API error: ${resp.status}`);
  const data = await resp.json();
  return data.meta.components; // Array of ComponentMetadata
}

// Fetch all team components with pagination
async function getTeamComponents(teamId, token) {
  let components = [];
  let cursor = null;

  do {
    const url = new URL(`https://api.figma.com/v1/teams/${teamId}/components`);
    url.searchParams.set('page_size', '100');
    if (cursor) url.searchParams.set('after', cursor);

    const resp = await fetch(url, { headers: { 'X-Figma-Token': token } });
    const data = await resp.json();
    components.push(...data.meta.components);
    cursor = data.meta.cursor || null;
  } while (cursor);

  return components;
}
```

## Plugin with TypeScript (Build Setup)

### Directory structure
```
plugin/
├── manifest.json
├── package.json
├── tsconfig.json
├── src/
│   ├── code.ts          # Sandbox entry
│   └── ui.tsx           # UI entry (if using framework)
├── ui.html              # Generated or static
└── dist/
    └── code.js          # Compiled output
```

### package.json
```json
{
  "scripts": {
    "build": "esbuild src/code.ts --bundle --outfile=dist/code.js --target=es2020",
    "watch": "esbuild src/code.ts --bundle --outfile=dist/code.js --target=es2020 --watch"
  },
  "devDependencies": {
    "@figma/plugin-typings": "^1.0.0",
    "esbuild": "^0.20.0"
  }
}
```

### tsconfig.json
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "lib": ["ES2020"],
    "strict": true,
    "moduleResolution": "node",
    "typeRoots": ["./node_modules/@figma/plugin-typings"]
  },
  "include": ["src/code.ts"]
}
```

## Grid Layout Pattern

Create a responsive card grid:
```js
async function createCardGrid(cards) {
  const grid = figma.createFrame();
  grid.name = 'Card Grid';
  grid.layoutMode = 'GRID';
  grid.gridColumnCount = 3;
  grid.gridRowGap = 16;
  grid.gridColumnGap = 16;
  grid.gridColumnSizes = [{ type: 'FLEX', value: 1 }, { type: 'FLEX', value: 1 }, { type: 'FLEX', value: 1 }];
  grid.gridRowSizes = [{ type: 'HUG' }];
  grid.paddingTop = grid.paddingBottom = grid.paddingLeft = grid.paddingRight = 24;
  grid.resize(800, 600);

  await figma.loadFontAsync({ family: 'Inter', style: 'Regular' });

  for (let i = 0; i < cards.length; i++) {
    const card = figma.createFrame();
    card.name = cards[i].title;
    card.layoutMode = 'VERTICAL';
    card.itemSpacing = 8;
    card.paddingTop = card.paddingBottom = card.paddingLeft = card.paddingRight = 16;
    card.cornerRadius = 8;
    card.fills = [{ type: 'SOLID', color: { r: 1, g: 1, b: 1 } }];
    card.effects = [{ type: 'DROP_SHADOW', color: { r: 0, g: 0, b: 0, a: 0.1 }, offset: { x: 0, y: 2 }, radius: 4, spread: 0, visible: true, blendMode: 'NORMAL' }];

    const title = figma.createText();
    title.characters = cards[i].title;
    title.fontSize = 14;
    card.appendChild(title);

    grid.appendChild(card);
  }

  figma.currentPage.appendChild(grid);
  return grid;
}
```

## Extended Variable Collection Theming

Override library variables for local themes:
```js
async function createThemeOverride(libraryCollectionKey, themeName, overrides) {
  const extended = await figma.variables.extendLibraryCollectionByKeyAsync(libraryCollectionKey, themeName);
  const modeId = extended.modes[0].modeId;

  for (const { variableId, value } of overrides) {
    const variable = await figma.variables.getVariableByIdAsync(variableId);
    if (variable) {
      variable.setValueForMode(modeId, value);
    }
  }

  figma.notify(`Theme "${themeName}" created with ${overrides.length} overrides`);
  return extended;
}
```

## Common Gotchas

1. **No `btoa`/`atob` in sandbox** — implement your own base64 encoder
2. **No `fetch` in sandbox** — all network calls must happen in the UI iframe
3. **Fills/strokes are readonly** — clone with `JSON.parse(JSON.stringify(...))` before modifying
4. **Colors are 0–1 range** — not 0–255; divide hex values by 255
5. **Load fonts before text edits** — or you get an error; use `getRangeAllFontNames` for mixed text
6. **`mainComponent` can be null** — always use `getMainComponentAsync()` and null-check
7. **Structured clone only** — no functions, DOM nodes, or class instances in messages
8. **No `setInterval`** — use recursive `setTimeout` instead
9. **Node might be removed** — check `node.removed` or wrap in try/catch
10. **Plugin data is per-plugin** — `node.getPluginData(key)` / `node.setPluginData(key, value)` stores strings scoped to your plugin ID
11. **`resetOverrides()` is deprecated** — use `instance.removeOverrides()` instead (since Nov 2025)
12. **PATs expire in ≤90 days** — non-expiring tokens no longer allowed; implement refresh logic
13. **GridTrackSize setter change** — type only auto-sets to `'FIXED'` from `'HUG'` (since Nov 2025)
