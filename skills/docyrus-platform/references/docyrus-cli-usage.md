# Docyrus CLI Usage

Complete command reference for the Docyrus CLI (`@docyrus/docyrus`).

## Global Flags

- `-g, --global` — Use global `~/.docyrus` settings instead of local project settings

---

## auth — Authentication

### `docyrus auth login`

Authorize CLI using OAuth2 device flow or manual token entry.

| Option | Type | Default | Description |
|---|---|---|---|
| `--clientId` | string | auto-resolved | OAuth2 client id |
| `--scope` | string | default scopes | OAuth2 scopes |
| `--accessToken` | string | — | Manual access token; skips device flow |
| `--refreshToken` | string | — | Manual refresh token (requires `--accessToken`) |

**Client ID resolution order:** explicit `--clientId` > `DOCYRUS_API_CLIENT_ID` env var > local config > global config > `manual-token` fallback.

**Default scopes:** `openid email profile offline_access ReadWrite.All User.ReadWrite Users.ReadWrite.All Tenant.Read Teams.Read.All DS.ReadWrite.All Docs.ReadWrite.All Architect.ReadWrite.All AI.ReadWrite.All`

### `docyrus auth set-tokens`

Set custom access and refresh tokens for the active environment.

| Option | Type | Required | Description |
|---|---|---|---|
| `--clientId` | string | no | OAuth2 client id |
| `--scope` | string | no | OAuth2 scopes |
| `--accessToken` | string | yes | Custom access token |
| `--refreshToken` | string | no | Custom refresh token |

### `docyrus auth accounts list`

List saved user accounts for the current API base URL.

### `docyrus auth accounts use`

Switch active account by user ID.

| Option | Type | Required | Description |
|---|---|---|---|
| `--userId` | string | yes | User ID to activate |

### `docyrus auth tenants list`

List available tenants for an account.

| Option | Type | Required | Description |
|---|---|---|---|
| `--userId` | string | no | User ID; defaults to active account |

### `docyrus auth tenants use <tenantSelector>`

Switch active tenant for an account.

| Argument | Type | Required | Description |
|---|---|---|---|
| `tenantSelector` | string | yes | Tenant number (numeric) or tenant UUID |

| Option | Type | Required | Description |
|---|---|---|---|
| `--userId` | string | no | User ID; defaults to active account |
| `--scope` | string | no | Scope for tenant bootstrap login if required |

**Note:** Numeric selector = tenant number, non-numeric = UUID.

### `docyrus auth logout`

Revoke and clear all tenant sessions for the active account.

| Option | Type | Required | Description |
|---|---|---|---|
| `--clientId` | string | no | OAuth2 client id override |

### `docyrus auth who`

Return current authenticated user (`/v1/users/me`).

---

## ai — AI Agent Chat

### `docyrus ai "<prompt>"`

Send a prompt to a Docyrus AI agent.

| Argument | Type | Required | Description |
|---|---|---|---|
| `prompt` | string | yes | Prompt string (quote when it contains spaces) |

| Option | Type | Default | Description |
|---|---|---|---|
| `--agentId` | string | default agent | Agent ID to use |
| `--deploymentId` | string | — | Agent deployment ID |

**Output behavior:**
- TTY mode: renders markdown for human readability
- `--json`, `--verbose`, or `--format`: preserves structured output

**Response fields:** `agentId`, `deploymentId`, `prompt`, `text`, `reasoning` (optional), `data` (optional).

---

## browser — Browser Automation

Browser automation commands (local Chrome or remote Cloudflare Browser Rendering). All commands return JSON with a `mode` field (`"local"` or `"remote"`).

### `docyrus browser start`

Start a browser session. Local mode launches Chrome on `:9222`. Sandbox mode creates a Cloudflare session.

| Option | Type | Description |
|---|---|---|
| `--profile` | boolean | Copy the default Chrome profile (local mode only) |

### `docyrus browser nav <url>`

Navigate the active tab or open the URL in a new tab.

| Option | Type | Description |
|---|---|---|
| `--new` | boolean | Open the URL in a new tab |
| `--reload` | boolean | Force a reload after navigation |

### `docyrus browser eval <code>`

Execute JavaScript in the active tab and print the result.

### `docyrus browser screenshot`

Capture the current viewport and return the temporary screenshot file path.

### `docyrus browser pick <message>`

Open an interactive element picker in the active tab and return information about the selected element or elements.

### `docyrus browser cookies`

Print cookies for the active tab, including domain, path, `httpOnly`, and `secure` flags.

### `docyrus browser content <url>`

Navigate to a URL and extract readable content as markdown.

### `docyrus browser run-script <script>`

Run a CDP script file on the active browser session.

| Option | Type | Description |
|---|---|---|
| `--appId` | string | App ID for browser session resolution |
| `--appSlug` | string | App slug for browser session resolution |
| `--keepAlive` | number | Session keep-alive in milliseconds |

---

## ds — Data Source Item Operations

### `docyrus ds get <appSlug> <dataSourceSlug>`

Get data source metadata, including its `fields`.

| Argument | Type | Required | Description |
|---|---|---|---|
| `appSlug` | string | yes | App slug |
| `dataSourceSlug` | string | yes | Data source slug |

### `docyrus ds list <appSlug> <dataSourceSlug>`

List data source items with the supported query parameters.

| Argument | Type | Required | Description |
|---|---|---|---|
| `appSlug` | string | yes | App slug |
| `dataSourceSlug` | string | yes | Data source slug |

Most frequently used options:

| Option | Type | Description |
|---|---|---|
| `--columns` | string | Column selection |
| `--filters` | string | JSON filter object |
| `--filterKeyword` | string | Keyword filter |
| `--orderBy` | string | Sort order |
| `--limit` | number | Result limit |
| `--offset` | number | Result offset |

Advanced options:

| Option | Type | Description |
|---|---|---|
| `--collapseRows` | boolean | Collapse rows into a single aggregated array |
| `--distinctColumns` | string | Distinct columns; comma-separated or JSON array |
| `--formulas` | string | JSON formulas object |
| `--calculations` | string | JSON calculations array |
| `--groupSummaries` | boolean | Return per-group summaries when calculations are used |
| `--fullCount` | boolean | Include total count |
| `--expand` | string | Expand columns; comma-separated or JSON array |
| `--pivot` | string | JSON pivot configuration |
| `--childQueries` | string | JSON child query array |

### `docyrus ds create <appSlug> <dataSourceSlug>`

Create data source item(s).

| Argument | Type | Required | Description |
|---|---|---|---|
| `appSlug` | string | yes | App slug |
| `dataSourceSlug` | string | yes | Data source slug |

| Option | Type | Description |
|---|---|---|
| `--data` | string | JSON payload for record fields |
| `--fromFile` | string | Path to JSON or CSV file |

**Notes:**
- Array payloads trigger bulk create (max 50 items per batch)
- Supports JSON and CSV input files

### `docyrus ds update <appSlug> <dataSourceSlug> [recordId]`

Update data source item(s).

| Argument | Type | Required | Description |
|---|---|---|---|
| `appSlug` | string | yes | App slug |
| `dataSourceSlug` | string | yes | Data source slug |
| `recordId` | string | for single updates | Record ID |

| Option | Type | Description |
|---|---|---|
| `--data` | string | JSON payload for record fields |
| `--fromFile` | string | Path to JSON or CSV file |

**Notes:**
- Batch update requires `id` in every item
- Cannot provide both `recordId` and batch payload

### `docyrus ds delete <appSlug> <dataSourceSlug> <recordId>`

Delete a data source item.

| Argument | Type | Required | Description |
|---|---|---|---|
| `appSlug` | string | yes | App slug |
| `dataSourceSlug` | string | yes | Data source slug |
| `recordId` | string | yes | Record ID |

### `docyrus ds comments create <appSlug> <dataSourceSlug> <recordId>`

Create a record-scoped comment.

| Argument | Type | Required | Description |
|---|---|---|---|
| `appSlug` | string | yes | App slug |
| `dataSourceSlug` | string | yes | Data source slug |
| `recordId` | string | yes | Record ID |

| Option | Type | Description |
|---|---|---|
| `--message` | string | Comment message |
| `--data` | string | Full JSON payload for the comment DTO |
| `--fromFile` | string | Path to a JSON payload file |
| `--parentId` | string | Parent comment ID |
| `--assignedTo` | string | Assigned user ID |
| `--attachments` | string | JSON attachments payload |
| `--level` | number | Comment level |
| `--status` | number | Comment status |
| `--done` | boolean | Mark comment as done |

**Notes:**
- Use either `--message` or `--data` / `--fromFile`
- `--data` and `--fromFile` cannot be mixed with field-specific flags

### `docyrus ds files upload <appSlug> <dataSourceSlug> <recordId>`

Upload a record-scoped file attachment.

| Argument | Type | Required | Description |
|---|---|---|---|
| `appSlug` | string | yes | App slug |
| `dataSourceSlug` | string | yes | Data source slug |
| `recordId` | string | yes | Record ID |

| Option | Type | Description |
|---|---|---|
| `--file` | string | Path to the local file to upload |
| `--contentType` | string | Override the inferred MIME type |
| `--publicFile` | boolean | Store the file in the public tenant bucket |

**Notes:**
- Uploads use `multipart/form-data`
- Content type is inferred from the file extension when omitted

---

## discover — OpenAPI Discovery

### `docyrus discover api`

Download tenant OpenAPI spec for the active tenant. Caches locally for subsequent use.

### `docyrus discover namespaces`

List API namespaces from the active tenant's OpenAPI spec.

### `docyrus discover path <prefix>`

List endpoints matching a path prefix.

| Argument | Type | Required | Description |
|---|---|---|---|
| `prefix` | string | yes | Path prefix (e.g., `/v1/users`) |

**Note:** Auto-normalizes paths with or without `/v1` prefix.

### `docyrus discover endpoint <selector>`

Return full endpoint details for a path and HTTP method.

| Argument | Type | Required | Description |
|---|---|---|---|
| `selector` | string | yes | Path (defaults to GET) or `[METHOD]/path` |

**Examples:**
- `/v1/users/me` — defaults to GET
- `[PUT]/v1/users/me/photo` — explicit PUT method

### `docyrus discover entity <name>`

Return full entity schema by name.

| Argument | Type | Required | Description |
|---|---|---|---|
| `name` | string | yes | Entity name (e.g., `UserEntity`) |

### `docyrus discover search <query>`

Search endpoint paths and entity names.

| Argument | Type | Required | Description |
|---|---|---|---|
| `query` | string | yes | Comma-separated search terms |

---

## connect — Connector & Action Commands

### `docyrus connect list-connectors`

List available integration connectors.

| Option | Type | Default | Description |
|---|---|---|---|
| `--q` | string | — | Keyword search on name, slug, or description |
| `--limit` | number | 100 | Max results |
| `--offset` | number | 0 | Result offset |

### `docyrus connect get-connector <slug>`

Get connector details with data sources and actions.

| Argument | Type | Required | Description |
|---|---|---|---|
| `slug` | string | yes | Data provider slug (e.g., `msgraph`) |

### `docyrus connect get-action <slug> <actionKey>`

Get connector action details including input/output schemas.

| Argument | Type | Required | Description |
|---|---|---|---|
| `slug` | string | yes | Data provider slug (e.g., `msgraph`) |
| `actionKey` | string | yes | Action key (e.g., `sendEmailWithOutlook`) |

### `docyrus connect list-connections <slug>`

Get tenant and user connections for a connector.

| Argument | Type | Required | Description |
|---|---|---|---|
| `slug` | string | yes | Data provider slug (e.g., `msgraph`) |

### `docyrus connect curl <slug> <endpoint>`

Send an HTTP request through a connector's provider auth.

| Argument | Type | Required | Description |
|---|---|---|---|
| `slug` | string | yes | Data provider slug (e.g., `msgraph`, `meta`) |
| `endpoint` | string | yes | Relative endpoint path or absolute URL |

| Option | Alias | Type | Default | Description |
|---|---|---|---|---|
| `--method` | `-X` | string | GET | HTTP method |
| `--data` | `-d` | string | — | JSON request payload |
| `--contentType` | | string | application/json | Content-Type header |
| `--headers` | | string | — | JSON object of additional headers |
| `--connectionId` | `-c` | string | — | Tenant connection ID override |
| `--connectionAccountId` | | string | — | Connection account ID |

### `docyrus connect run-action <appSlug> <actionKey>`

Run a connector or app action via `POST /v1/apps/:appSlug/actions/:actionKey/run`.

| Argument | Type | Required | Description |
|---|---|---|---|
| `appSlug` | string | yes | App slug (e.g., `base`) |
| `actionKey` | string | yes | Action key (e.g., `sendEmailWithOutlook`) |

| Option | Alias | Type | Default | Description |
|---|---|---|---|---|
| `--params` | `-p` | string | — | JSON object with action input parameters |
| `--connectionId` | `-c` | string | — | Tenant connection ID override |
| `--connectionAccountId` | | string | — | Tenant connection account ID |
| `--dryRun` | `-n` | boolean | false | Preview request without executing |

---

## apps — App Management

### `docyrus apps list`

List apps.

| Option | Type | Description |
|---|---|---|
| `--appType` | string | Filter by app type |

### `docyrus apps delete`

Archive an app (soft delete).

| Option | Type | Description |
|---|---|---|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |

**Note:** Exactly one of `--appId` or `--appSlug` required.

### `docyrus apps restore`

Restore an archived app.

| Option | Type | Description |
|---|---|---|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |

### `docyrus apps permanent-delete`

Permanently delete an app.

| Option | Type | Description |
|---|---|---|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |

---

## studio — Schema Management

Manage data source schemas, fields, enumerations, saved views, forms, public webforms, HTML/PDF/DOCX export templates, and email templates via the development API.

**Common selector rules:**
- App: exactly one of `--appId` or `--appSlug`
- Data source: exactly one of `--dataSourceId` or `--dataSourceSlug` (where supported)
- Field: exactly one of `--fieldId` or `--fieldSlug` (where supported)

### Data Source Commands

#### `docyrus studio list-data-sources`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--expand` | string | Comma-separated expansions (e.g., `fields`) |

#### `docyrus studio get-data-source`

| Option | Type | Description |
|---|---|---|
| `--dataSourceId` | string | Data source ID |

Returns the data source metadata together with its `fields`.

#### `docyrus studio create-data-source`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |
| `--title` | string | Data source title |
| `--name` | string | Data source name |
| `--slug` | string | Data source slug |
| `--type` | string | Data source type |
| `--icon` | string | Icon |
| `--dataSharing` | string | Data sharing mode |
| `--meta` | string | JSON meta payload |

#### `docyrus studio update-data-source`

Same options as `create-data-source` plus data source selector (`--dataSourceId / --dataSourceSlug`).

#### `docyrus studio delete-data-source`

Archive a data source.

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--dataSourceId / --dataSourceSlug` | string | Data source selector |

#### `docyrus studio restore-data-source`

Restore an archived data source. Requires `--dataSourceId` (slug resolution not available for archived data sources).

#### `docyrus studio permanent-delete-data-source`

Permanently delete a data source. Requires `--dataSourceId`.

#### `docyrus studio bulk-create-data-sources`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |

### Field Commands

#### `docyrus studio list-fields`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--dataSourceId / --dataSourceSlug` | string | Data source selector |

#### `docyrus studio get-field`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--dataSourceId / --dataSourceSlug` | string | Data source selector |
| `--fieldId / --fieldSlug` | string | Field selector |

#### `docyrus studio create-field`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--dataSourceId / --dataSourceSlug` | string | Data source selector |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |
| `--name` | string | Field name |
| `--slug` | string | Field slug |
| `--type` | string | Field type |
| `--readOnly` | boolean | Read only |
| `--status` | number | Field status |
| `--defaultValue` | string | Default value |
| `--relationDataSourceId` | string | Relation target data source ID |
| `--sortOrder` | number | Sort order |
| `--tenantEnumSetId` | string | Shared enum set ID |
| `--options` | string | JSON editor options |
| `--validations` | string | JSON validations |

#### `docyrus studio update-field`

Same options as `create-field` plus field selector (`--fieldId / --fieldSlug`).

#### `docyrus studio delete-field`

| Option | Type | Description |
|---|---|---|
| App, data source, and field selectors | string | See above |

#### `docyrus studio create-fields-batch`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--dataSourceId / --dataSourceSlug` | string | Data source selector |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |

#### `docyrus studio update-fields-batch`

Same options as `create-fields-batch`.

**Note:** The CLI auto-normalizes payloads: `id → fieldId`, `read_only → readOnly`, `default_value → defaultValue`, `relation_data_source_id → relationDataSourceId`, `options → editorOptions`.

#### `docyrus studio delete-fields-batch`

Same options. Payload key: `fieldIds`.

### Enum Commands

#### `docyrus studio list-enums`

| Option | Type | Description |
|---|---|---|
| App, data source, and field selectors | string | See above |

#### `docyrus studio create-enums`

| Option | Type | Description |
|---|---|---|
| App, data source, and field selectors | string | See above |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |
| `--enumSetId` | string | Enum set ID |

#### `docyrus studio update-enums`

Same options as `create-enums` (without `--enumSetId`).

**Note:** The CLI auto-normalizes `id → enumId`.

#### `docyrus studio delete-enums`

Same options. Payload key: `enumIds`.

### Data View Commands

Saved view definitions for a data source. Routes through `/v1/apps/:appSlug/data-sources/:dataSourceSlug/views`. App and data source can be supplied as id or slug — the CLI resolves whichever side is missing.

#### `docyrus studio list-data-views`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--dataSourceId / --dataSourceSlug` | string | Data source selector |
| `--tenantAppId` | string | Optional tenant app ID to scope the view list |

#### `docyrus studio get-data-view`

Same selectors as `list-data-views`, plus `--viewId` (required).

#### `docyrus studio create-data-view`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--dataSourceId / --dataSourceSlug` | string | Data source selector |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |
| `--name` | string | View name |
| `--description` | string | View description |
| `--tenantAppId` | string | Optional tenant app ID |
| `--columns` | string | JSON columns payload |
| `--filters` | string | JSON filters payload |
| `--sort` | string | JSON sort payload |
| `--color` | string | Color |
| `--icon` | string | Icon |
| `--colorRules` | string | JSON color rules payload |
| `--quickFilterFields` | string | JSON array of field slugs |
| `--isDefault` | boolean | Mark as default view |
| `--sortOrder` | number | Sort order |

#### `docyrus studio update-data-view`

Same options as `create-data-view`, plus `--viewId` (required) and `--archived` (boolean).

#### `docyrus studio delete-data-view`

Same selectors as `list-data-views`, plus `--viewId` (required).

### Form Commands

Data source form definitions used by record-entry UIs. Routes through `/v1/apps/:appSlug/data-sources/:dataSourceSlug/forms`.

#### `docyrus studio list-forms`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--dataSourceId / --dataSourceSlug` | string | Data source selector |

#### `docyrus studio get-form`

Same selectors as `list-forms`, plus `--formId` (required).

#### `docyrus studio create-form`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--dataSourceId / --dataSourceSlug` | string | Data source selector |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |
| `--name` | string | Form name |
| `--description` | string | Description |
| `--title` | string | Title |
| `--subtopic` | string | Subtopic |
| `--color` | string | Color |
| `--icon` | string | Icon |
| `--layout` | string | JSON layout payload |
| `--isDefault` | boolean | Mark as default form |
| `--status` | number | Form status |

#### `docyrus studio update-form`

Same options as `create-form`, plus `--formId` (required) and `--archived` (boolean).

#### `docyrus studio delete-form`

Same selectors as `list-forms`, plus `--formId` (required).

### Webform Commands

Public-facing webforms. Routes through `/v1/dev/webforms`. CRUD by `--webformId`. List and create accept either `--dataSourceId` or `--dataSourceSlug` (slug requires `--appId` or `--appSlug` to resolve).

#### `docyrus studio list-webforms`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | Optional, used to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Filter by data source ID |
| `--dataSourceSlug` | string | Filter by data source slug (requires `--appId` or `--appSlug`) |

#### `docyrus studio get-webform`

| Option | Type | Description |
|---|---|---|
| `--webformId` | string | Webform ID (required) |

#### `docyrus studio create-webform`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | Optional, used to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Data source ID to bind the webform to |
| `--dataSourceSlug` | string | Data source slug (requires `--appId` or `--appSlug`) |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |
| `--name` | string | Webform name |
| `--schema` | string | JSON schema payload |
| `--status` | number | Status (1 active, 2 inactive) |
| `--webformOptions` | string | JSON options payload |
| `--sandbox` | boolean | Sandbox flag |
| `--css` | string | Custom CSS |

When `dataSourceId` is omitted, submissions land in the tenant-schema `webform_record` table instead of a data source.

#### `docyrus studio update-webform`

| Option | Type | Description |
|---|---|---|
| `--webformId` | string | Webform ID (required) |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |
| `--name` | string | Webform name |
| `--schema` | string | JSON schema payload |
| `--status` | number | Status (1 active, 2 inactive) |
| `--webformOptions` | string | JSON options payload |
| `--sandbox` | boolean | Sandbox flag |
| `--css` | string | Custom CSS |

#### `docyrus studio delete-webform`

| Option | Type | Description |
|---|---|---|
| `--webformId` | string | Webform ID (required) |

### HTML Template Commands

HTML/PDF/DOCX export templates. Routes through `/v1/dev/html-templates`. CRUD by `--templateId`. Data source binding is required on create; slug requires `--appId` or `--appSlug`.

#### `docyrus studio list-html-templates`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | Optional, used to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Filter by data source ID |
| `--dataSourceSlug` | string | Filter by data source slug (requires `--appId` or `--appSlug`) |
| `--isDefault` | boolean | Filter by default flag |
| `--limit` | number | Page size (default 100) |
| `--offset` | number | Page offset (default 0) |

#### `docyrus studio get-html-template`

| Option | Type | Description |
|---|---|---|
| `--templateId` | string | HTML template ID (required) |

#### `docyrus studio create-html-template`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | Optional, used to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Data source ID this template binds to |
| `--dataSourceSlug` | string | Data source slug (requires `--appId` or `--appSlug`) |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |
| `--name` | string | Template name |
| `--filenameTmpl` | string | Filename template |
| `--pageOrientation` | string | Page orientation |
| `--sourceType` | string | Source type (`html`, `pdf`, `docx`, ...) |
| `--marginLeft` | number | Left margin |
| `--marginRight` | number | Right margin |
| `--marginTop` | number | Top margin |
| `--marginBottom` | number | Bottom margin |
| `--pageFormat` | string | Page format (`A4`, `Letter`, ...) |
| `--body` | string | HTML body |
| `--isDefault` | boolean | Mark as default template |
| `--headerTmpl` | string | Header template |
| `--footerTmpl` | string | Footer template |
| `--styles` | string | Inline CSS styles |

#### `docyrus studio update-html-template`

Same options as `create-html-template`, plus `--templateId` (required).

#### `docyrus studio delete-html-template`

| Option | Type | Description |
|---|---|---|
| `--templateId` | string | HTML template ID (required) |

### Email Template Commands

Transactional email templates. Routes through `/v1/dev/email-templates`. CRUD by `--templateId`. Data source binding is optional.

#### `docyrus studio list-email-templates`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | Optional, used to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Filter by data source ID |
| `--dataSourceSlug` | string | Filter by data source slug (requires `--appId` or `--appSlug`) |
| `--limit` | number | Page size (default 100) |
| `--offset` | number | Page offset (default 0) |

#### `docyrus studio get-email-template`

| Option | Type | Description |
|---|---|---|
| `--templateId` | string | Email template ID (required) |

#### `docyrus studio create-email-template`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | Optional, used to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Optional data source ID to bind the template to |
| `--dataSourceSlug` | string | Data source slug (requires `--appId` or `--appSlug`) |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |
| `--name` | string | Template name |
| `--subject` | string | Email subject |
| `--body` | string | Email body |
| `--ownership` | string | Ownership (`system`, `user`, ...) |

#### `docyrus studio update-email-template`

Same options as `create-email-template`, plus `--templateId` (required).

#### `docyrus studio delete-email-template`

| Option | Type | Description |
|---|---|---|
| `--templateId` | string | Email template ID (required) |

---

## automation — Automations, Triggers, and Action Nodes

Manage automations, their triggers, and their action nodes for a tenant app. All commands route through `/v1/dev/apps/:appId/automations`.

**Common selector rules:**
- App: exactly one of `--appId` or `--appSlug`
- Automation: `--automationId` (no slug — automations and nodes are ID-only)
- Trigger `--type` (URL kebab-case): `record-created`, `record-modified`, `record-deleted`, `recurrence`, `app-event`, `webhook`, `emailhook`, `webform`, `button-activation`, `manual-activation`
- Node `--type` (URL kebab-case): `external-action`, `send-email`, `send-notification`, `create-record`, `update-records`, `request-approval`, `request-input`, `http-request`, `data-source-query`, `custom-query`, `generate-document`, `ai-prompt`, `ai-agent`, `execute-script`, `wait-for`

**Write payload rules:**
- Write commands accept `--data '<json>'` or `--from-file ./payload.json` (JSON only)
- Convenience flags are camelCase and are converted to `snake_case` in the request body
- Nested objects (trigger `data`; node `data`, `field_mapping`, `dynamic_field_mapping`, `condition`, `input_template`, `input_transformer`, `custom_headers`, `pre_action_request`, `post_action_request`, `target_data_source_condition`) must be supplied via `--data` / `--from-file` — the CLI does not flatten them
- `delete`, `delete-trigger`, and `delete-node` return `{ deleted: true, id }` (API itself returns 204)

### Automation CRUD

#### `docyrus automation list`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |

#### `docyrus automation get`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--automationId` | string | Automation ID (required) |

#### `docyrus automation create`

Creates an automation together with its first trigger. `--triggerType` uses camelCase (`recordCreated`, `recordModified`, `recordDeleted`, `recurrence`, `appEvent`, `webhook`, `emailhook`, `webform`, `buttonActivation`, `manualActivation`) to match `CreateAutomationDto.trigger_type`.

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--data` | string | JSON payload |
| `--fromFile` | string | Path to JSON file |
| `--name` | string | Automation name |
| `--triggerType` | string | Initial trigger type (camelCase) |
| `--status` | number | Automation status |
| `--sourceDataSourceId` | string | Source data source ID |
| `--triggerDataSourceId` | string | Trigger data source ID |
| `--triggerDataProviderId` | string | Trigger data provider ID |
| `--triggerWebhookId` | string | Trigger webhook ID |

#### `docyrus automation update`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--automationId` | string | Automation ID (required) |
| `--data / --fromFile` | string | JSON payload / file |
| `--name` | string | Automation name |
| `--status` | number | Automation status |
| `--sourceDataSourceId` | string | Source data source ID |

#### `docyrus automation delete`

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--automationId` | string | Automation ID (required) |

### Trigger CRUD

#### `docyrus automation list-triggers`

Derived from the automation GET response.

#### `docyrus automation get-trigger`

Same options as `list-triggers`, plus `--triggerId` (required).

#### `docyrus automation create-trigger`

Routes through `POST .../triggers/:type`.

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--automationId` | string | Automation ID (required) |
| `--type` | string | Trigger type (kebab-case, required) |
| `--data / --fromFile` | string | JSON payload / file |
| `--active` | boolean | Whether the trigger is active |
| `--sourceDataSourceId` | string | Source data source (record-*, recurrence, button/manual) |
| `--maxRunPerRecord` | number | Max runs per record (record-*, recurrence) |
| `--modifiedColumns` | string | Comma-separated columns (record-modified) |
| `--modifiedColumnsCondition` | string | `all` or `any` (record-modified) |
| `--recurrenceFrequency` | string | `hour`, `day`, `week`, `month`, `year` (recurrence) |
| `--recurrenceInterval` | number | Recurrence interval (recurrence) |
| `--recurrenceMinutes` | number | Minutes `0\|15\|30\|45` (recurrence) |
| `--recurrenceWeekDays` | string | Comma-separated week days `MON,TUE,...` (recurrence) |
| `--recurrenceMonthDays` | string | `DAY_OF_MONTH` or `DAY_OF_WEEK` (recurrence) |
| `--recurrenceStartDate` | string | ISO date (recurrence) |
| `--recurrenceEndDate` | string | ISO date (recurrence) |
| `--recurrenceRunAt` | string | `HH:mm` (recurrence) |
| `--coreDataProviderId` | string | Core data provider ID (app-event) |
| `--coreDataProviderWebhookId` | string | Core data provider webhook ID (app-event) |
| `--webhookId` | string | Webhook ID (webhook, emailhook) |
| `--webhookName` | string | Name for auto-created webhook (webhook, emailhook) |
| `--tenantWebformId` | string | Tenant webform ID (webform) |

#### `docyrus automation update-trigger`

Routes through `PATCH .../triggers/:type/:triggerId`. Same flags as `create-trigger`, plus `--triggerId` (required).

#### `docyrus automation delete-trigger`

Routes through `DELETE .../triggers/:triggerId` (type-independent).

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--automationId` | string | Automation ID (required) |
| `--triggerId` | string | Trigger ID (required) |

### Action Node CRUD

#### `docyrus automation list-nodes`

`GET .../nodes`

#### `docyrus automation get-node`

`GET .../nodes/:nodeId`

#### `docyrus automation create-node`

Routes through `POST .../nodes/:type`. `external-action` create requires `--actionTypeId`; the backend validates `data` against `core_action.input_json_schema` and provisions the `tenant_action` row in the same transaction.

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--automationId` | string | Automation ID (required) |
| `--type` | string | Node type (kebab-case, required) |
| `--data / --fromFile` | string | JSON payload / file |
| `--name` | string | Node name |
| `--description` | string | Node description |
| `--subType` | string | Sub type discriminator |
| `--parent` | string | Parent node ID |
| `--active` | boolean | Whether the node is active |
| `--actionTypeId` | string | Action type ID (required for external-action create; maps to `core_action.id`) |
| `--sourceDataSourceId` | string | Source data source ID (external-action) |
| `--targetDataSourceId` | string | Target data source ID (create-record, update-records, http-request, data-source-query, external-action) |
| `--targetDataSourceFieldId` | string | Target data source field ID (update-records) |
| `--connectionId` | string | Connection ID (http-request, external-action) |
| `--connectionAccountId` | string | Connection account ID (http-request, external-action) |
| `--webhookId` | string | Webhook ID (external-action) |
| `--inputDataSourceId` | string | Input data source ID (request-approval, request-input) |
| `--requestMethod` | string | HTTP method (http-request): `GET`, `POST`, `PUT`, `PATCH`, `DELETE` |
| `--contentType` | string | HTTP content type (http-request) |
| `--customEndpoint` | string | HTTP endpoint (http-request) |
| `--relativeEndpoint` | boolean | Whether the endpoint is relative to the connection base URL (http-request) |
| `--batch` | boolean | Whether to send the HTTP request in batches (http-request) |
| `--batchSize` | number | HTTP batch size 1..10000 (http-request) |
| `--outputTransformer` | string | Output transformer expression (http-request) |
| `--batchTransformer` | string | Batch transformer expression (http-request) |
| `--errorTransformer` | string | Error transformer expression (http-request) |

`wait-for` nodes accept no flat convenience flags beyond the common base — supply `data.delaySeconds` (integer ≤ 30 days) or the `data.delayValue` + `data.delayUnit` pair (`seconds` / `minutes` / `hours` / `days`) via `--data` / `--from-file`. The action forwards input through unchanged and queues the next step(s) with a deferred `tenant_job_queue.process_after`.

```sh
docyrus automation create-node \
  --appSlug crm \
  --automationId 9c4f… \
  --type wait-for \
  --name "Wait 2 hours" \
  --parent <previous-node-id> \
  --data '{"data":{"delayValue":2,"delayUnit":"hours"}}'
```

#### `docyrus automation update-node`

Routes through `PATCH .../nodes/:type/:nodeId`. Same flags as `create-node`, plus `--nodeId` (required).

#### `docyrus automation delete-node`

Routes through `DELETE .../nodes/:nodeId` (type-independent).

| Option | Type | Description |
|---|---|---|
| `--appId / --appSlug` | string | App selector |
| `--automationId` | string | Automation ID (required) |
| `--nodeId` | string | Node ID (required) |

---

## messaging — Tenant Email Accounts and Send

List tenant email accounts and send transactional emails. Routes through `/v1/messaging/email/*` and requires the `Messaging.Email.Send` OAuth2 scope. Credentials are never returned.

### `docyrus messaging accounts`

`GET /messaging/email/accounts`

Returns active tenant email accounts. Each item exposes `id`, `name`, `provider`, `senderEmail`, `senderName`, `isUserAccessible`, `allowOverrideName`, `allowOverrideEmail`, `createdOn`.

### `docyrus messaging email send`

`POST /messaging/email/accounts/:accountId/send`

| Option | Type | Description |
|---|---|---|
| `--accountId` | string | Tenant email account UUID (required) |
| `--to` | string | Comma-separated recipient addresses |
| `--cc` | string | Comma-separated CC addresses |
| `--bcc` | string | Comma-separated BCC addresses |
| `--replyTo` | string | Comma-separated reply-to addresses |
| `--subject` | string | Subject (max 998 chars) |
| `--body` | string | HTML or text body (max 1 000 000 chars) |
| `--sendAsUser` | boolean | Send using the authenticated user's identity when the account allows the override |
| `--data` | string | Full JSON payload; overrides individual flags when set |
| `--fromFile` | string | Read full JSON payload from a JSON file |

Limits: up to 50 addresses per recipient list, up to 10 attachments. Attachments are `{ filePath, fileName?, mimeType? }` with `filePath` referencing a tenant-scoped storage path.

Response payload: `{ messageId, provider, accepted, rejected }`.

---

## curl — Direct API Requests

### `docyrus curl <path>`

Send arbitrary requests to the Docyrus API.

| Argument | Type | Required | Description |
|---|---|---|---|
| `path` | string | yes | API path (no absolute URLs) |

| Option | Type | Description |
|---|---|---|
| `-X, --request` | string | HTTP method (GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS) |
| `-H, --header` | string[] | Request headers (`Key:Value`, repeatable) |
| `-d, --data` | string | Request payload |
| `-G, --get` | boolean | Send data as query string |
| `-i, --include` | boolean | Include status and response headers |
| `--noAuth` | boolean | Skip Authorization header |

**Notes:**
- Default method: GET (POST if `-d` provided)
- Path auto-normalizes `/v1` prefix
- JSON payloads auto-detect `Content-Type: application/json`

---

## Settings & Persistence

### Storage Locations

| File | Path | Description |
|---|---|---|
| Auth state | `<settings>/auth.json` | Multi-account, multi-tenant sessions |
| Environment config | `<settings>/config.json` | Active environment and client config |
| OpenAPI cache | `<settings>/tenans/<tenantId>/openapi.json` | Cached tenant OpenAPI specs |

**Default settings root:** `./.docyrus/` (local) or `~/.docyrus/` (global with `-g`).

### Environment Variables

| Variable | Description |
|---|---|
| `DOCYRUS_API_CLIENT_ID` | OAuth2 client ID fallback |
