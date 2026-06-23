# Connector Model, Commands & Actions

Reference for the Docyrus `docyrus connect` commands and the connector data model. Source of truth: `apps/cli/src/commands/connectCommands.ts`, `apps/api/src/connector/connector.controller.ts` + `connector.service.ts` + `dto/connector.dto.ts`, `libs/shared/src/data-provider/{DataProviderAuth,HttpHelper}.ts`, and `database/schemas/public/tables/{core_data_provider,core_action,tenant_connection,tenant_connection_user,tenant_connection_account}.sql`.

## Table of Contents

1. [Data model](#data-model)
2. [Auth types](#auth-types)
3. [Command reference](#command-reference)
4. [Connection resolution (run-action / curl)](#connection-resolution-run-action--curl)
5. [run-action vs curl](#run-action-vs-curl)
6. [Gotchas](#gotchas)

## Data model

| Concept | Table | Key fields | Notes |
|---|---|---|---|
| **Connector / data provider** | `core_data_provider` | `slug`, `name`, `auth_type`, `base_url`, `provides_actions`, `provides_webhooks` | The integration definition; addressed by **slug**. "Connector" (CLI/API) = "data provider" (DB/shared) — same row. |
| **Action** | `core_action` (rows with `core_data_provider_id` + `key`, `status=1`) | `key`, `name`, `request_method`, `custom_endpoint`, `input_json_schema`, `output_json_schema` | Callable op; addressed by **provider slug + action `key`**. |
| **Tenant connection** | `tenant_connection` | `id`, `name`, `core_data_provider_id`, `access_token`, `refresh_token`, `client_id`, `client_secret`, `base_url`, `archived` | Tenant-wide credentials. No status column — `archived` is the lifecycle flag. |
| **User connection** | `tenant_connection_user` | `id`, `user_id`, `access_token`, `refresh_token`, `code`, `pkce_code_verifier`, `shared` | Per-user OAuth2 (`authorization_code`) connection. `shared=true` lets other tenant users reuse it. |
| **Connection account** | `tenant_connection_account` | `id`, `account_id`, `account_name`, `tenant_connection_id` **or** `tenant_connection_user_id` | A sub-account within a connection (one of several external accounts); pass via `--connectionAccountId`. |
| **Product provider auth** | `core_product_data_provider_auth` | `client_id`, `client_secret` | Docyrus-product-level OAuth client config (sensitive; not in CLI). |

API entities use **camelCase** (`authType`, `providesActions`, `inputJsonSchema`, `requestMethod`, `apiEndpoint`, `baseUrl`, `createdOn`).

## Auth types

`core_data_provider.auth_type` ∈ `oauth2` | `basic` | `bearer` | `api_key` | `json` | `custom` | `noauth` | `none`. The provider auth header is injected automatically on `run-action`/`curl` (skipped only for `noauth`).

- **`oauth2` + grant `authorization_code`** → **per-user** connections (`tenant_connection_user`). The running user's own connection (or a `shared` one) is used; `--connectionId` is ignored.
- **All other auth types** (incl. other oauth2 grants) → **tenant-level** connection (`tenant_connection`), honoring `--connectionId` or auto-picking the first.

## Command reference

All commands are `docyrus connect <sub>`; require an active session (except `run-action --dryRun`, which is fully client-side). Append `--json`.

| Command | Args | Flags | Calls | Returns |
|---|---|---|---|---|
| `list-connectors` | — | `--q`, `--limit` (100), `--offset` (0) | `GET /v1/connectors?q&limit&offset` | `{ data: connectors[], meta:{total,limit,offset} }` (ILIKE on name/slug/description) |
| `get-connector` | `<slug>` | — | `GET /v1/connectors/{slug}` | connector detail: `authType`, `dataSources[]`, `actions[]`, `authJsonSchema`, … (404 if slug unknown) |
| `get-action` | `<slug> <actionKey>` | — | `GET /v1/connectors/{slug}/actions/{actionKey}` | `{ key, name, inputJsonSchema, outputJsonSchema, requestMethod, apiEndpoint }` (404 if action not `status=1`) |
| `list-connections` | `<slug>` | — | `GET /v1/connectors/{slug}/connections` | `{ tenantScope:[{id,name,baseUrl,createdOn}], userScope:{connected,connectionId} }` |
| `run-action` | `<slug> <actionKey>` | `-p/--params` (json obj), `-c/--connectionId`, `--connectionAccountId`, `-n/--dryRun` | `POST /v1/connectors/{slug}/actions/{actionKey}/run` | the action `IActionResponse` (`{data,status,…}`); HTTP status = action status |
| `curl` | `<slug> <endpoint>` | `-X/--method` (GET), `-d/--data` (json), `--contentType`, `--headers` (json), `-c/--connectionId`, `--connectionAccountId` | `PUT /v1/connectors/{slug}` (endpoint in **body**) | `{ data, status, error, rawResponse }` |

**Selector addressing:** providers by **slug** (path), actions by **key** (path), connections by **id** (flags only — there's no connection slug).

**Scopes:** read commands (`list-connectors`/`get-connector`/`get-action`/`list-connections`/`curl`) require `Connectors.Read.All`; **`run-action` requires `Automations.Run`.**

> CLI-only nuance: the CLI's `list-connections <slug>` hits the **per-provider** route and returns `{tenantScope,userScope}`. (A tenant-wide `GET /v1/connectors/connections` exists on the backend but is not exposed by the CLI.)

## Connection resolution (run-action / curl)

`run-action` (`resolveActionConnection` → `DataProviderAuth.getAuthData`):

1. **OAuth2 `authorization_code` provider** → `--connectionId` is **ignored**; uses the **current user's** `tenant_connection_user` row, falling back to a tenant **`shared`** user connection. 404 `"User connection not found"` / `"Shared user connection not found"` if neither exists.
2. **Any other auth type** → tenant connection: with `--connectionId`, that exact connection (tenant-filtered); **without it, the first** matching tenant connection. 404 `"Tenant connection not found"` if none.
3. **`--connectionAccountId`** does not pick the credential — it routes the call to a specific external sub-account on top of the resolved connection.

`curl` resolves with `superUser:true` (skips the tenant filter) but otherwise the same first-or-specified logic.

## run-action vs curl

| | `run-action` | `curl` |
|---|---|---|
| Use when | the connector defines an **action** for the operation | you need a **raw** call the actions don't cover |
| Address | `<slug> <actionKey>` | `<slug> <endpoint>` |
| Body | `--params` (validated vs `inputJsonSchema`) | `--data` (free) |
| Connection selectors | sent as **headers** (`x-connection-id`, `x-connection-account-id`) | sent in the **body** |
| HTTP method | defined by the action | your `--method`/`-X` |
| Dry run | `--dryRun` previews client-side (sends nothing) | none |
| Scope | `Automations.Run` | `Connectors.Read.All` |

## Gotchas

- **Connections aren't created via the CLI.** Only `list-connections` (read). Create connections in the Docyrus UI / OAuth flow (`oauth2-auth-url` → `oauth2-callback`) or the raw API (`POST /v1/connectors/{slug}/connections`). If `list-connections` shows nothing usable, the provider isn't connected yet.
- **`run-action` selectors are headers; `curl` selectors are body.** Don't mix them.
- **`curl`'s `<endpoint>` is a body field on `PUT /connectors/{slug}`**, not a URL path. Relative → appended to the connection/provider base URL; absolute (`http…`) → used verbatim.
- **`--params` must be a JSON object** (CLI rejects arrays/primitives) and is AJV-validated server-side (400 `"Action input validation failed"`; non-object body → 400 `"Action run request body must be a JSON object"`).
- **`--dryRun` is client-side only** — there is no server dry-run; it returns the would-be request and sends nothing. It also skips the active-session check.
- **Scope mismatch:** a session allowed to *read* connectors may be denied `run-action` (needs `Automations.Run`).
- **Action must be active** (`status=1`) and addressed by the exact provider `slug` + action `key`, else 404.
