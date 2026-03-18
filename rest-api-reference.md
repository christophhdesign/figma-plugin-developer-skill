# Figma REST API Reference

The REST API is used from the **UI iframe** (via `fetch`) or external tools. It is NOT available in the plugin sandbox.

## Authentication

### Personal Access Token (PAT)
```
GET https://api.figma.com/v1/files/:file_key
Headers:
  X-Figma-Token: <personal-access-token>
```

**IMPORTANT (April 2025):** PATs now have a maximum expiration of 90 days. Non-expiring PATs are no longer allowed. Implement token rotation/refresh flows accordingly.

### OAuth 2.0
```
Authorization: Bearer <oauth-token>
```

**Token refresh (May 2025):** Use `POST /v1/oauth/token` for refreshing tokens. The old `/v1/oauth/refresh` endpoint is deprecated.

### Scoped Permissions (April 2025)

The `files:read` scope is deprecated. Use specific scopes instead:
- `file_content:read` — read file content/tree
- `file_comments:read` — read comments
- `file_metadata:read` — read file metadata
- `file_dev_resources:read` / `file_dev_resources:write` — dev resources

Store tokens securely using `figma.clientStorage` in the sandbox, then pass to the UI via `postMessage`.

## Base URL

```
https://api.figma.com
```

## Rate Limits

- **Per-user**: ~30 requests/minute per token for most endpoints
- **Burst**: Short bursts allowed, then throttled
- Handle `429 Too Many Requests` with exponential backoff
- Use `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers

## Core Endpoints

### Files

**GET /v1/files/:file_key**
Returns the full document tree.

```js
// Common query params:
// ?depth=2         — limit tree depth (reduces payload)
// ?node-id=1:2     — return only a specific node subtree
// ?geometry=paths  — include vector path data
// ?plugin_data=shared — include plugin data
```

Response shape:
```json
{
  "name": "File Name",
  "lastModified": "2025-01-01T00:00:00Z",
  "version": "123456",
  "document": {
    "id": "0:0",
    "name": "Document",
    "type": "DOCUMENT",
    "children": [/* pages */]
  },
  "components": { /* key -> ComponentMetadata */ },
  "componentSets": { /* key -> ComponentSetMetadata */ },
  "styles": { /* key -> StyleMetadata */ }
}
```

**GET /v1/files/:file_key/nodes?ids=1:2,3:4**
Returns specific nodes by ID. More efficient than fetching full file.

### Images

**GET /v1/images/:file_key**
Render nodes as images (PNG, JPG, SVG, PDF).

```js
// Query params:
// ?ids=1:2,3:4         — node IDs to render
// ?format=png          — png | jpg | svg | pdf
// ?scale=2             — 0.01 to 4
// ?svg_include_id=true — include node IDs in SVG
// ?svg_simplify_stroke=true
// ?use_absolute_bounds=true
```

Response:
```json
{
  "images": {
    "1:2": "https://figma-alpha-api.s3.us-west-2.amazonaws.com/images/...",
    "3:4": "https://..."
  }
}
```

Image URLs expire after ~14 days. Re-fetch as needed.

**GET /v1/files/:file_key/images**
List all images used in the file (fills, etc.).

### Components & Styles

**GET /v1/files/:file_key/components**
All published components in a file.

**GET /v1/files/:file_key/component_sets**
All published component sets in a file.

**GET /v1/files/:file_key/styles**
All published styles in a file.

**GET /v1/teams/:team_id/components?page_size=100&after=cursor**
Published team library components. Paginated.

**GET /v1/teams/:team_id/component_sets?page_size=100**
Published team library component sets. Paginated.

**GET /v1/teams/:team_id/styles?page_size=100**
Published team library styles. Paginated.

Component metadata shape:
```json
{
  "key": "abc123...",
  "name": "Button",
  "description": "Primary action button",
  "file_key": "xxxxx",
  "node_id": "1:2",
  "thumbnail_url": "https://...",
  "containing_frame": {
    "name": "Components",
    "nodeId": "5:0",
    "pageName": "Library"
  },
  "created_at": "2025-01-01T00:00:00Z",
  "updated_at": "2025-06-01T00:00:00Z"
}
```

### Comments

**GET /v1/files/:file_key/comments**
**POST /v1/files/:file_key/comments**
```json
{
  "message": "Comment text",
  "client_meta": { "x": 100, "y": 200 }
}
```

**DELETE /v1/files/:file_key/comments/:comment_id**

### File Metadata (April 2025)

**GET /v1/files/:file_key/metadata**
Returns file metadata (name, last modified, etc.) without the full document tree. Requires `file_metadata:read` scope. More efficient when you only need file info.

### Users & Projects

**GET /v1/me** — Current user info
**GET /v1/teams/:team_id/projects** — Team projects
**GET /v1/projects/:project_id/files** — Files in a project

### Versions

**GET /v1/files/:file_key/versions**
File version history.

### Variables

**GET /v1/files/:file_key/variables/local**
Local variables and collections.

**GET /v1/files/:file_key/variables/published**
Published variables.

**POST /v1/files/:file_key/variables**
Create/update/delete variables. Body:
```json
{
  "variableCollections": [
    { "action": "CREATE", "name": "Colors", "id": "temp-id" }
  ],
  "variableModeValues": [
    { "variableId": "VariableID:123", "modeId": "ModeID:456", "value": { "r": 0.2, "g": 0.4, "b": 1, "a": 1 } }
  ],
  "variables": [
    { "action": "CREATE", "name": "primary", "variableCollectionId": "temp-id", "resolvedType": "COLOR", "id": "temp-var-id" }
  ]
}
```

### Dev Resources

**GET /v1/files/:file_key/dev_resources**
**POST /v1/files/:file_key/dev_resources**
**PUT /v1/dev_resources/:dev_resource_id**
**DELETE /v1/dev_resources/:dev_resource_id**

### Webhooks

**POST /v2/webhooks**
```json
{
  "event_type": "FILE_UPDATE",
  "team_id": "123456",
  "endpoint": "https://your-server.com/webhook",
  "passcode": "secret",
  "description": "File update notifications"
}
```

Event types:
- `FILE_UPDATE` — File saved
- `FILE_DELETE` — File deleted
- `FILE_VERSION_UPDATE` — New version created
- `LIBRARY_PUBLISH` — Library published
- `FILE_COMMENT` — Comment added
- `DEV_MODE_STATUS_UPDATE` — Dev Mode status changed (May 2025)

**GET /v2/webhooks/:webhook_id**
**PUT /v2/webhooks/:webhook_id**
**DELETE /v2/webhooks/:webhook_id**

**Webhooks V2 (May 2025):**
- File-level and project-level webhooks now supported (previously team-only)
- **`GET /v2/webhooks`** — List webhooks across all contexts (new)
  - `?context=team&context_id=:team_id` — filter by team
  - `?context=file&context_id=:file_key` — filter by file
  - `?context=project&context_id=:project_id` — filter by project
- **DEPRECATED:** `GET /v2/teams/:team_id/webhooks` — use `GET /v2/webhooks?context=team&context_id=:team_id` instead

### Payments (for paid plugins)

**GET /v1/payments** — Check payment status in plugin code
Query params: `?plugin_id=xxx&user_id=yyy`

Response:
```json
{
  "payment": {
    "status": "PAID",     // 'PAID' | 'UNPAID' | 'TRIAL'
    "date": "2025-01-01"
  }
}
```

## Fetch Pattern for Plugin UI

```js
// In ui.html
async function fetchFigmaAPI(endpoint, token) {
  const resp = await fetch(`https://api.figma.com${endpoint}`, {
    headers: { 'X-Figma-Token': token },
  });
  if (!resp.ok) {
    if (resp.status === 429) {
      // Rate limited — wait and retry
      const retryAfter = parseInt(resp.headers.get('Retry-After') || '60', 10);
      await new Promise(r => setTimeout(r, retryAfter * 1000));
      return fetchFigmaAPI(endpoint, token);
    }
    throw new Error(`Figma API ${resp.status}: ${resp.statusText}`);
  }
  return resp.json();
}

// Usage
const fileData = await fetchFigmaAPI(`/v1/files/${fileKey}?depth=1`, token);
const components = await fetchFigmaAPI(`/v1/files/${fileKey}/components`, token);
```

## Token Storage Pattern

```js
// code.js — load and save token
(async () => {
  const token = await figma.clientStorage.getAsync('figma_pat');
  figma.ui.postMessage({ type: 'token-loaded', token: token || '' });
})();

figma.ui.onmessage = async (msg) => {
  if (msg.type === 'save-token') {
    await figma.clientStorage.setAsync('figma_pat', msg.token);
  }
};
```

## Pagination

Team endpoints use cursor-based pagination:
```
GET /v1/teams/:id/components?page_size=50
→ { "meta": { "components": [...] }, "cursor": "abc123" }

GET /v1/teams/:id/components?page_size=50&after=abc123
→ next page
```

Loop until no `cursor` in response.
