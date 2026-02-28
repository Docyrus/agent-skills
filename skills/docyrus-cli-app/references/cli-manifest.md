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

Discovery commands.

### docyrus discover api

Download tenant OpenAPI spec for the active tenant.

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

Create a data source item.

```
docyrus ds create <appSlug> <dataSourceSlug> [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--data` | string | JSON payload for record fields |

### docyrus ds update

Update a data source item.

```
docyrus ds update <appSlug> <dataSourceSlug> <recordId> [options]
```

| Flag | Type | Description |
|------|------|-------------|
| `--data` | string | JSON payload for record fields |

### docyrus ds delete

Delete a data source item.

```
docyrus ds delete <appSlug> <dataSourceSlug> <recordId>
```

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
