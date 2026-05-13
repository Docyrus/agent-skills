# Docyrus CLI Manifest

LLM-ready reference for the `docyrus` CLI tool.

---

## Global Options

| Flag | Type | Description |
|------|------|-------------|
| `-g`, `--global` | boolean | Force global `~/.docyrus/` settings instead of local `./.docyrus/` |
| `--format <toon\|json\|yaml\|md\|jsonl>` | string | Output format |
| `--help` | boolean | Show help |
| `--llms` | boolean | Print LLM-readable manifest |
| `--mcp` | boolean | Start as MCP stdio server |
| `--verbose` | boolean | Show full output envelope |
| `--version` | boolean | Show version |

## Environment Variables

| Name | Description |
|------|-------------|
| `DOCYRUS_API_CLIENT_ID` | Default Docyrus OAuth2 client id |

## Settings Scope

- Default scope is local: `./.docyrus/`
- Use `-g` or `--global` to force global scope: `~/.docyrus/`
- OpenAPI cache path is `<settings-root>/tenans/<tenantId>/openapi.json`
- `docyrus` without a subcommand returns active environment, help commands, and auth `context`

---

## docyrus env

Environment commands.

### docyrus env list

List available environments.

### docyrus env use

Switch active environment by id or name.

```
docyrus env use <selector>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `selector` | string | yes | Environment id or name (`live`, `prod`, `beta`, `alpha`, `dev`) |

Built-in environments:
- `live` -> `https://api.docyrus.com`
- `beta` -> `https://beta-api.docyrus.com`
- `alpha` -> `https://alpha-api.docyrus.com`
- `dev` -> `https://localhost:3366`

---

## docyrus apps

App commands.

### docyrus apps list

List apps (`/v1/apps`).

| Flag | Type | Description |
|------|------|-------------|
| `--appType` | string | Optional app type filter |

---

## docyrus auth

Authentication commands.

### docyrus auth login

Authorize CLI using OAuth2 device flow, or provide tokens manually.

| Flag | Type | Description |
|------|------|-------------|
| `--clientId` | string | OAuth2 client id |
| `--scope` | string | OAuth2 scopes (default: `openid email profile offline_access ReadWrite.All User.ReadWrite Users.Read.All Tenant.Read Teams.Read.All DS.ReadWrite.All Docs.ReadWrite.All Architect.ReadWrite.All`) |
| `--accessToken` | string | Manual access token; skips device flow |
| `--refreshToken` | string | Manual refresh token; requires `--accessToken` |

Login notes:
- Resolution order for client ID is `--clientId` -> `DOCYRUS_API_CLIENT_ID` -> saved local config -> saved global config
- Manual token login falls back to `manual-token` only when no client ID can be resolved
- Local login can reuse the globally saved client ID when local config does not have one

### docyrus auth logout

Revoke and clear all tenant sessions for the active account in the current environment.

### docyrus auth who

Return the current authenticated user (`/v1/users/me`).

### docyrus auth accounts list

List saved user accounts for the current API base URL.

### docyrus auth accounts use

Switch active account by user ID.

| Flag | Type | Description |
|------|------|-------------|
| `--userId` | string | User ID to activate |

### docyrus auth tenants list

List available tenants for an account.

| Flag | Type | Description |
|------|------|-------------|
| `--userId` | string | User ID; defaults to active account |

### docyrus auth tenants use

Switch active tenant for an account.

```
docyrus auth tenants use <tenantSelector> [options]
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `tenantSelector` | string | yes | Numeric tenant no or UUID tenant id |

| Flag | Type | Description |
|------|------|-------------|
| `--userId` | string | User ID; defaults to active account |
| `--scope` | string | Scope used only when tenant bootstrap login is required |

Selector rules:
- numeric selector -> match by `tenantNo`
- non-numeric selector must be a UUID tenant id

---

## docyrus curl

Send arbitrary requests to Docyrus API.

```
docyrus curl <path> [options]
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `path` | string | yes | API path (no absolute URL) |

| Flag | Alias | Type | Description |
|------|-------|------|-------------|
| `--request` | `-X` | string | HTTP method |
| `--header` | `-H` | array | Request header (repeatable) |
| `--data` | `-d` | string | Request payload |
| `--get` | `-G` | boolean | Send data as query string |
| `--include` | `-i` | boolean | Include status and response headers |
| `--noAuth` | | boolean | Skip Authorization header |

---

## docyrus discover

Discovery commands for exploring the tenant OpenAPI spec. All discover commands require an active login session. Commands other than `discover api` auto-download the spec if it does not exist locally.

### docyrus discover api

Download tenant OpenAPI spec for the active tenant.

### docyrus discover namespaces

List API namespaces from the active tenant OpenAPI spec. Returns deduplicated namespace prefixes such as `/v1/users` and `/v1/apps`.

### docyrus discover path

List endpoints with method and description for a matching path prefix.

```
docyrus discover path <prefix>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `prefix` | string | yes | Path prefix, for example `/v1/users` (the `/v1` prefix is optional) |

### docyrus discover endpoint

Return the full OpenAPI endpoint object for a path and HTTP method.

```
docyrus discover endpoint <selector>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `selector` | string | yes | Endpoint selector such as `/v1/users/me` or `[PUT]/v1/users/me/photo` |

Selector format: `/path` defaults to GET; `[METHOD]/path` specifies an explicit HTTP method. The `/v1` prefix is optional.

### docyrus discover entity

Return the full schema definition object for an entity by name.

```
docyrus discover entity <name>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | yes | Entity name (case-sensitive), for example `UserEntity` |

### docyrus discover search

Search endpoint paths and entity names by comma-separated terms.

```
docyrus discover search <query>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `query` | string | yes | One or more comma-separated search strings, for example `users,UserEntity` |

Endpoint results include method and description when available.

---

## docyrus connect

Connector and action commands. Interact with external integration connectors and run actions.

### docyrus connect list-connectors

List available integration connectors.

| Flag | Type | Description |
|------|------|-------------|
| `--q` | string | Keyword search on name, slug, or description |
| `--limit` | number | Max results (default: 100) |
| `--offset` | number | Result offset (default: 0) |

### docyrus connect get-connector

Get connector details with data sources and actions.

```
docyrus connect get-connector <slug>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `slug` | string | yes | Data provider slug (e.g. `msgraph`) |

### docyrus connect get-action

Get connector action details including input/output schemas.

```
docyrus connect get-action <slug> <actionKey>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `slug` | string | yes | Data provider slug (e.g. `msgraph`) |
| `actionKey` | string | yes | Action key (e.g. `sendEmailWithOutlook`) |

### docyrus connect list-connections

Get tenant and user connections for a connector.

```
docyrus connect list-connections <slug>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `slug` | string | yes | Data provider slug (e.g. `msgraph`) |

### docyrus connect curl

Send an HTTP request through a connector's provider auth.

```
docyrus connect curl <slug> <endpoint> [options]
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `slug` | string | yes | Data provider slug (e.g. `msgraph`, `meta`) |
| `endpoint` | string | yes | Relative endpoint path or absolute URL |

| Flag | Alias | Type | Description |
|------|-------|------|-------------|
| `--method` | `-X` | string | HTTP method (default: `GET`) |
| `--data` | `-d` | string | JSON request payload |
| `--contentType` | | string | Content-Type header (defaults to `application/json`) |
| `--headers` | | string | JSON object of additional headers |
| `--connectionId` | `-c` | string | Tenant connection ID override |
| `--connectionAccountId` | | string | Connection account ID |

### docyrus connect run-action

Run a connector or app action.

```
docyrus connect run-action <appSlug> <actionKey> [options]
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `appSlug` | string | yes | App slug (e.g. `base`, or a custom app slug) |
| `actionKey` | string | yes | Action key (e.g. `sendEmailWithOutlook`) |

| Flag | Alias | Type | Description |
|------|-------|------|-------------|
| `--params` | `-p` | string | JSON object with action input parameters |
| `--connectionId` | `-c` | string | Tenant connection ID override |
| `--connectionAccountId` | | string | Tenant connection account ID |
| `--dryRun` | `-n` | boolean | Preview the request without executing |

---

## docyrus ds

Data source commands for CRUD operations on records.

### docyrus ds get

Get data source metadata (fields, types, relations).

```
docyrus ds get <appSlug> <dataSourceSlug>
```

### docyrus ds list

List data source items with optional filtering, sorting, and pagination.

```
docyrus ds list <appSlug> <dataSourceSlug> [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--columns` | string | Comma-separated field slugs to select |
| `--filters` | string | JSON filter object |
| `--limit` | number | Max records to return |
| `--offset` | number | Skip N records |
| `--orderBy` | string | Sort expression |
| `--fullCount` | boolean | Include total count in response |

### docyrus ds create

Create data source item(s). If payload is an array, CLI sends a bulk request to `POST /apps/:appSlug/data-sources/:dataSourceSlug/items/bulk`.

```
docyrus ds create <appSlug> <dataSourceSlug> [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--data` | string | JSON payload for record fields |
| `--from-file` | string | Path to `.json` or `.csv` payload file |

Batch rules:
- array payload triggers bulk create endpoint
- maximum 50 items per batch

### docyrus ds update

Update data source item(s). If payload is an array, CLI sends a bulk request to `PATCH /apps/:appSlug/data-sources/:dataSourceSlug/items/bulk`.

```
docyrus ds update <appSlug> <dataSourceSlug> [recordId] [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--data` | string | JSON payload for record fields |
| `--from-file` | string | Path to `.json` or `.csv` payload file |

Update rules:
- object payload uses the single-item endpoint and requires `recordId`
- array payload uses the bulk update endpoint and requires `id` in each item
- do not provide positional `recordId` for batch update
- maximum 50 items per batch

### docyrus ds delete

Delete a data source item.

```
docyrus ds delete <appSlug> <dataSourceSlug> <recordId>
```

Successful command responses include a top-level `context` object with:
- `email`
- `tenantName`
- `tenantNo`
- `tenantDisplay`

---

## docyrus studio

Dev app data source schema CRUD commands under `/v1/dev/apps/:app_id/data-sources`.

Selector rules:
- app selector: exactly one of `--appId` or `--appSlug`
- data source selector: exactly one of `--dataSourceId` or `--dataSourceSlug` when required
- field selector: exactly one of `--fieldId` or `--fieldSlug` when required

Write payload rules:
- use `--data '<json>'` or `--from-file ./payload.json` (JSON only), not both
- flags override conflicting keys from JSON payload
- batch commands accept either a root array or a root object containing the expected DTO key

### Data source commands

#### docyrus studio list-data-sources

`GET /dev/apps/:app_id/data-sources`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--expand` | string | Optional comma-separated expansions such as `fields` |

#### docyrus studio get-data-source

`GET /dev/apps/:app_id/data-sources/:id`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |

#### docyrus studio create-data-source

`POST /dev/apps/:app_id/data-sources`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--data` | string | JSON payload |
| `--from-file` | string | Path to JSON payload file |
| `--title` | string | Data source title |
| `--name` | string | Data source name |
| `--slug` | string | Data source slug |
| `--type` | string | Data source type |
| `--icon` | string | Icon |
| `--dataSharing` | string | Data sharing value |
| `--meta` | string | JSON meta payload |

#### docyrus studio update-data-source

`PATCH /dev/apps/:app_id/data-sources/:id`

Same flags as `create-data-source`, plus selector flags:

| Flag | Type | Description |
|------|------|-------------|
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |

#### docyrus studio delete-data-source

`DELETE /dev/apps/:app_id/data-sources/:id`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |

#### docyrus studio bulk-create-data-sources

`POST /dev/apps/:app_id/data-sources/bulk`

Expected DTO key: `dataSources`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--data` | string | JSON payload |
| `--from-file` | string | Path to JSON payload file |

### Field commands

#### docyrus studio list-fields

`GET /dev/apps/:app_id/data-sources/:data_source_id/fields`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |

#### docyrus studio get-field

`GET /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--fieldId` | string | Field ID |
| `--fieldSlug` | string | Field slug |

#### docyrus studio create-field

`POST /dev/apps/:app_id/data-sources/:data_source_id/fields`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--data` | string | JSON payload |
| `--from-file` | string | Path to JSON payload file |
| `--name` | string | Field name |
| `--slug` | string | Field slug |
| `--type` | string | Field type |
| `--readOnly` | boolean | Read-only flag |
| `--status` | string | Field status |
| `--defaultValue` | string | Default value |
| `--relationDataSourceId` | string | Related data source ID |
| `--sortOrder` | number | Sort order |
| `--tenantEnumSetId` | string | Tenant enum set ID |
| `--options` | string | JSON options payload |
| `--validations` | string | JSON validations payload |

#### docyrus studio update-field

`PATCH /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id`

Same flags as `create-field`, plus selector flags:

| Flag | Type | Description |
|------|------|-------------|
| `--fieldId` | string | Field ID |
| `--fieldSlug` | string | Field slug |

#### docyrus studio delete-field

`DELETE /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--fieldId` | string | Field ID |
| `--fieldSlug` | string | Field slug |

#### docyrus studio create-fields-batch

`POST /dev/apps/:app_id/data-sources/:data_source_id/fields/batch`

Expected DTO key: `fields`

#### docyrus studio update-fields-batch

`PATCH /dev/apps/:app_id/data-sources/:data_source_id/fields/batch`

Expected DTO key: `fields`

#### docyrus studio delete-fields-batch

`DELETE /dev/apps/:app_id/data-sources/:data_source_id/fields/batch`

Expected DTO key: `fieldIds`

### Enum commands

#### docyrus studio list-enums

`GET /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id/enums`

#### docyrus studio create-enums

`POST /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id/enums`

Expected DTO key: `enums`

#### docyrus studio update-enums

`PATCH /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id/enums`

Expected DTO key: `enums`

#### docyrus studio delete-enums

`DELETE /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id/enums`

Expected DTO key: `enumIds`

### Data view commands

Saved view definitions for a data source. These commands route through `/v1/apps/:appSlug/data-sources/:dataSourceSlug/views` (app + data source identified by slug). Pass either `--appId` or `--appSlug` and either `--dataSourceId` or `--dataSourceSlug`; the CLI resolves whichever side you didn't provide.

#### docyrus studio list-data-views

`GET /apps/:appSlug/data-sources/:dataSourceSlug/views`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--tenantAppId` | string | Optional tenant app ID to scope the view list |

#### docyrus studio get-data-view

`GET /apps/:appSlug/data-sources/:dataSourceSlug/views/:viewId`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--viewId` | string | Data source view ID (required) |

#### docyrus studio create-data-view

`POST /apps/:appSlug/data-sources/:dataSourceSlug/views`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--data` | string | JSON payload |
| `--from-file` | string | Path to JSON payload file |
| `--name` | string | View name |
| `--description` | string | View description |
| `--tenantAppId` | string | Optional tenant app ID |
| `--columns` | string | JSON columns payload |
| `--filters` | string | JSON filters payload |
| `--sort` | string | JSON sort payload |
| `--color` | string | Color |
| `--icon` | string | Icon |
| `--colorRules` | string | JSON color rules payload |
| `--quickFilterFields` | string | JSON array of quick filter field slugs |
| `--isDefault` | boolean | Mark as default view |
| `--sortOrder` | number | Sort order |

#### docyrus studio update-data-view

`PUT /apps/:appSlug/data-sources/:dataSourceSlug/views/:viewId`

Same flags as `create-data-view`, plus:

| Flag | Type | Description |
|------|------|-------------|
| `--viewId` | string | Data source view ID (required) |
| `--archived` | boolean | Archive flag |

#### docyrus studio delete-data-view

`DELETE /apps/:appSlug/data-sources/:dataSourceSlug/views/:viewId`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--viewId` | string | Data source view ID (required) |

### Form commands

Data source form definitions used by record-entry UIs. Routes through `/v1/apps/:appSlug/data-sources/:dataSourceSlug/forms`.

#### docyrus studio list-forms

`GET /apps/:appSlug/data-sources/:dataSourceSlug/forms`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |

#### docyrus studio get-form

`GET /apps/:appSlug/data-sources/:dataSourceSlug/forms/:formId`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--formId` | string | Form ID (required) |

#### docyrus studio create-form

`POST /apps/:appSlug/data-sources/:dataSourceSlug/forms`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--data` | string | JSON payload |
| `--from-file` | string | Path to JSON payload file |
| `--name` | string | Form name |
| `--description` | string | Description |
| `--title` | string | Title |
| `--subtopic` | string | Subtopic |
| `--color` | string | Color |
| `--icon` | string | Icon |
| `--layout` | string | JSON layout payload |
| `--isDefault` | boolean | Mark as default form |
| `--status` | number | Form status |

#### docyrus studio update-form

`PUT /apps/:appSlug/data-sources/:dataSourceSlug/forms/:formId`

Same flags as `create-form`, plus:

| Flag | Type | Description |
|------|------|-------------|
| `--formId` | string | Form ID (required) |
| `--archived` | boolean | Archive flag |

#### docyrus studio delete-form

`DELETE /apps/:appSlug/data-sources/:dataSourceSlug/forms/:formId`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--formId` | string | Form ID (required) |

### Webform commands

Public-facing webforms that submit into either a tenant data source or, when unbound, a tenant-schema `webform_record` table. Routes through `/v1/dev/webforms`. Pass `--dataSourceId` directly, or `--dataSourceSlug` together with `--appId`/`--appSlug` to resolve the ID.

#### docyrus studio list-webforms

`GET /dev/webforms`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | Optional app ID used only to resolve `--dataSourceSlug` |
| `--appSlug` | string | Optional app slug used only to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Filter by data source ID |
| `--dataSourceSlug` | string | Filter by data source slug (requires `--appId` or `--appSlug`) |

#### docyrus studio get-webform

`GET /dev/webforms/:webformId`

| Flag | Type | Description |
|------|------|-------------|
| `--webformId` | string | Webform ID (required) |

#### docyrus studio create-webform

`POST /dev/webforms`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | Optional app ID used only to resolve `--dataSourceSlug` |
| `--appSlug` | string | Optional app slug used only to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Data source ID to bind the webform to |
| `--dataSourceSlug` | string | Data source slug (requires `--appId` or `--appSlug`) |
| `--data` | string | JSON payload |
| `--from-file` | string | Path to JSON payload file |
| `--name` | string | Webform name |
| `--schema` | string | JSON schema payload |
| `--status` | number | Status (1 active, 2 inactive) |
| `--webformOptions` | string | JSON options payload |
| `--sandbox` | boolean | Sandbox flag |
| `--css` | string | Custom CSS |

#### docyrus studio update-webform

`PATCH /dev/webforms/:webformId`

| Flag | Type | Description |
|------|------|-------------|
| `--webformId` | string | Webform ID (required) |
| `--data` | string | JSON payload |
| `--from-file` | string | Path to JSON payload file |
| `--name` | string | Webform name |
| `--schema` | string | JSON schema payload |
| `--status` | number | Status (1 active, 2 inactive) |
| `--webformOptions` | string | JSON options payload |
| `--sandbox` | boolean | Sandbox flag |
| `--css` | string | Custom CSS |

#### docyrus studio delete-webform

`DELETE /dev/webforms/:webformId`

| Flag | Type | Description |
|------|------|-------------|
| `--webformId` | string | Webform ID (required) |

### HTML template commands

HTML/PDF/DOCX export templates. Routes through `/v1/dev/html-templates`. Identify a data source either by ID or by slug (slug requires `--appId` or `--appSlug`).

#### docyrus studio list-html-templates

`GET /dev/html-templates`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | Optional app ID used only to resolve `--dataSourceSlug` |
| `--appSlug` | string | Optional app slug used only to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Filter by data source ID |
| `--dataSourceSlug` | string | Filter by data source slug (requires `--appId` or `--appSlug`) |
| `--isDefault` | boolean | Filter by default flag |
| `--limit` | number | Page size (default 100) |
| `--offset` | number | Page offset (default 0) |

#### docyrus studio get-html-template

`GET /dev/html-templates/:templateId`

| Flag | Type | Description |
|------|------|-------------|
| `--templateId` | string | HTML template ID (required) |

#### docyrus studio create-html-template

`POST /dev/html-templates`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | Optional app ID used only to resolve `--dataSourceSlug` |
| `--appSlug` | string | Optional app slug used only to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Data source ID this template binds to |
| `--dataSourceSlug` | string | Data source slug (requires `--appId` or `--appSlug`) |
| `--data` | string | JSON payload |
| `--from-file` | string | Path to JSON payload file |
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

#### docyrus studio update-html-template

`PUT /dev/html-templates/:templateId`

Same flags as `create-html-template`, plus:

| Flag | Type | Description |
|------|------|-------------|
| `--templateId` | string | HTML template ID (required) |

#### docyrus studio delete-html-template

`DELETE /dev/html-templates/:templateId`

| Flag | Type | Description |
|------|------|-------------|
| `--templateId` | string | HTML template ID (required) |

### Email template commands

Transactional email templates. Routes through `/v1/dev/email-templates`. The data source binding is optional on create/update.

#### docyrus studio list-email-templates

`GET /dev/email-templates`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | Optional app ID used only to resolve `--dataSourceSlug` |
| `--appSlug` | string | Optional app slug used only to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Filter by data source ID |
| `--dataSourceSlug` | string | Filter by data source slug (requires `--appId` or `--appSlug`) |
| `--limit` | number | Page size (default 100) |
| `--offset` | number | Page offset (default 0) |

#### docyrus studio get-email-template

`GET /dev/email-templates/:templateId`

| Flag | Type | Description |
|------|------|-------------|
| `--templateId` | string | Email template ID (required) |

#### docyrus studio create-email-template

`POST /dev/email-templates`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | Optional app ID used only to resolve `--dataSourceSlug` |
| `--appSlug` | string | Optional app slug used only to resolve `--dataSourceSlug` |
| `--dataSourceId` | string | Optional data source ID to bind the template to |
| `--dataSourceSlug` | string | Data source slug (requires `--appId` or `--appSlug`) |
| `--data` | string | JSON payload |
| `--from-file` | string | Path to JSON payload file |
| `--name` | string | Template name |
| `--subject` | string | Email subject |
| `--body` | string | Email body |
| `--ownership` | string | Ownership (e.g. `system`, `user`) |

#### docyrus studio update-email-template

`PUT /dev/email-templates/:templateId`

Same flags as `create-email-template`, plus:

| Flag | Type | Description |
|------|------|-------------|
| `--templateId` | string | Email template ID (required) |

#### docyrus studio delete-email-template

`DELETE /dev/email-templates/:templateId`

| Flag | Type | Description |
|------|------|-------------|
| `--templateId` | string | Email template ID (required) |

---

## docyrus tui

Launch the OpenTUI terminal UI.

Notes:
- requires Bun installed locally
- reuses the existing CLI command graph
- intended for interactive terminal usage rather than browser embedding
