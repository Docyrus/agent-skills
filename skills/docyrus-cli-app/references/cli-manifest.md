# Docyrus CLI Manifest

LLM-ready reference for the `docyrus` CLI tool.

---

## Global Options

| Flag | Type | Description |
|------|------|-------------|
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

---

## docyrus apps

App commands.

### docyrus apps list

List apps (`/v1/dev/apps`).

| Flag | Type | Description |
|------|------|-------------|
| `--appType` | string | Optional app type filter |

---

## docyrus auth

Authentication commands.

### docyrus auth login

Authorize CLI using OAuth2 device flow.

| Flag | Type | Description |
|------|------|-------------|
| `--clientId` | string | OAuth2 client id |
| `--scope` | string | OAuth2 scopes (default: `openid email profile offline_access ReadWrite.All User.ReadWrite Users.Read.All Tenant.Read Teams.Read.All DS.ReadWrite.All Docs.ReadWrite.All Architect.ReadWrite.All`) |

### docyrus auth logout

Revoke and clear all tenant sessions for active account.

| Flag | Type | Description |
|------|------|-------------|
| `--clientId` | string | OAuth2 client id override |

### docyrus auth who

Return current authenticated user (`/v1/users/me`).

### docyrus auth accounts list

List saved user accounts for current API base URL.

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

| Flag | Type | Description |
|------|------|-------------|
| `--tenantId` | string | Tenant ID to activate |
| `--userId` | string | User ID; defaults to active account |
| `--scope` | string | Scope used only when tenant bootstrap login is required |

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

Discovery commands for exploring the tenant OpenAPI spec. All discover commands require an active login session. Commands other than `discover api` auto-download the spec if it doesn't exist locally.

Local spec file path: `~/docyrus/tenans/<tenantId>/openapi.json`

### docyrus discover api

Download tenant OpenAPI spec for the active tenant.

### docyrus discover namespaces

List API namespaces from the active tenant OpenAPI spec. Extracts deduplicated namespace prefixes (e.g. `/v1/users`, `/v1/teams`) from all paths.

### docyrus discover path

List endpoints with method and description for a matching path prefix.

```
docyrus discover path <prefix>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `prefix` | string | yes | Path prefix, e.g. `/v1/users` (the `/v1` prefix is optional) |

### docyrus discover endpoint

Return the full OpenAPI endpoint object for a path and HTTP method.

```
docyrus discover endpoint <selector>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `selector` | string | yes | Endpoint selector, e.g. `/v1/users/me` or `[PUT]/v1/users/me/photo` |

Selector format: `/path` defaults to GET; `[METHOD]/path` specifies an explicit HTTP method. The `/v1` prefix is optional.

### docyrus discover entity

Return the full schema/definition object for an entity by name.

```
docyrus discover entity <name>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `name` | string | yes | Entity name (case-sensitive), e.g. `UserEntity` |

### docyrus discover search

Search endpoint paths and entity names by comma-separated terms (case-insensitive substring match).

```
docyrus discover search <query>
```

| Argument | Type | Required | Description |
|----------|------|----------|-------------|
| `query` | string | yes | One or more comma-separated search strings, e.g. `users,UserEntity` |

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

Create data source item(s). If payload is an array, CLI sends a bulk request to `POST /:appSlug/data-sources/:dataSourceSlug/items/bulk`.

```
docyrus ds create <appSlug> <dataSourceSlug> [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--data` | string | JSON payload for record fields |
| `--from-file` | string | Path to `.json` or `.csv` payload file |

Batch rules:
- Array payload triggers bulk create endpoint
- Maximum 50 items per batch

### docyrus ds update

Update data source item(s). If payload is an array, CLI sends a bulk request to `PATCH /:appSlug/data-sources/:dataSourceSlug/items/bulk`.

```
docyrus ds update <appSlug> <dataSourceSlug> [recordId] [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--data` | string | JSON payload for record fields |
| `--from-file` | string | Path to `.json` or `.csv` payload file |

Update rules:
- Object payload uses single update endpoint and requires `recordId`
- Array payload uses bulk update endpoint and requires `id` in each item
- Do not provide positional `recordId` for batch update
- Maximum 50 items per batch

### docyrus ds delete

Delete a data source item.

```
docyrus ds delete <appSlug> <dataSourceSlug> <recordId>
```

---

## docyrus studio

Dev app data source schema CRUD commands under `/v1/dev/apps/:app_id/data-sources`.

Selector rules:
- App selector: exactly one of `--appId` or `--appSlug`
- Data source selector: exactly one of `--dataSourceId` or `--dataSourceSlug` (when required)
- Field selector: exactly one of `--fieldId` or `--fieldSlug` (when required)

Write payload rules:
- Use `--data '<json>'` or `--from-file ./payload.json` (JSON only), not both
- Flags override conflicting keys from JSON payload
- Batch commands accept either root array or root object containing expected DTO key

### Data source commands

#### docyrus studio list-data-sources

`GET /dev/apps/:app_id/data-sources`

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--expand` | string | Optional comma-separated expansions (e.g. `fields`) |

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
| `--readOnly` | boolean | Field read only |
| `--status` | number | Field status |
| `--defaultValue` | string | Default value |
| `--relationDataSourceId` | string | Relation data source ID |
| `--sortOrder` | number | Sort order |
| `--tenantEnumSetId` | string | Tenant enum set ID |
| `--options` | string | JSON options |
| `--validations` | string | JSON validations |

#### docyrus studio update-field

`PATCH /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id`

Same flags as `create-field`, plus field selector flags:

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

For all 3 field batch commands:

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--data` | string | JSON payload |
| `--from-file` | string | Path to JSON payload file |

### Enum commands

#### docyrus studio list-enums

`GET /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id/enums`

#### docyrus studio create-enums

`POST /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id/enums`

Expected DTO key: `enums` (optional `enumSetId`)

#### docyrus studio update-enums

`PATCH /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id/enums`

Expected DTO key: `enums`

#### docyrus studio delete-enums

`DELETE /dev/apps/:app_id/data-sources/:data_source_id/fields/:field_id/enums`

Expected DTO key: `enumIds`

For enum commands:

| Flag | Type | Description |
|------|------|-------------|
| `--appId` | string | App ID |
| `--appSlug` | string | App slug |
| `--dataSourceId` | string | Data source ID |
| `--dataSourceSlug` | string | Data source slug |
| `--fieldId` | string | Field ID |
| `--fieldSlug` | string | Field slug |
| `--data` | string | JSON payload |
| `--from-file` | string | Path to JSON payload file |
| `--enumSetId` | string | Enum set ID (create only) |

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
| `selector` | string | yes | Environment id or name |
