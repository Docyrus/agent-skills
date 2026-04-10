---
name: docyrus-cli-app
description: Use the Docyrus CLI (`docyrus`) to interact with the Docyrus platform from the terminal. Use when the user asks to authenticate, list apps, query or manage data records (`ds`), manage dev app data source schema objects (`studio`), send API requests, switch environments, tenants, or accounts, discover tenant OpenAPI specs, or use the Bun-powered terminal UI via `docyrus tui`. Triggers on tasks involving docyrus CLI commands, terminal-based Docyrus operations, `docyrus ds list`, `docyrus studio`, `docyrus discover`, `docyrus auth`, `docyrus env`, `docyrus tui`, or shell-based Docyrus workflows.
---

# Docyrus CLI

Guide for using the `docyrus` CLI to interact with the Docyrus platform from the terminal.

## Command Overview

| Command | Description |
|---------|-------------|
| `docyrus` | Show active environment, current auth context, and help summary |
| `docyrus env list` / `env use` | Manage named environments |
| `docyrus auth login` | Authenticate via OAuth2 device flow or manual tokens |
| `docyrus auth logout` | Logout the active account for the current environment |
| `docyrus auth who` | Show the active user and tenant |
| `docyrus auth accounts list` / `use` | Manage saved user accounts |
| `docyrus auth tenants list` / `use` | Manage saved tenants for a user |
| `docyrus apps list` | List apps from `/v1/apps` |
| `docyrus ds get` | Get data source metadata |
| `docyrus ds list` | Query records with filters, sorting, pagination |
| `docyrus ds create` / `update` / `delete` | Mutate records, including bulk create/update |
| `docyrus studio ...` | CRUD for dev app data sources, fields, and enums |
| `docyrus discover api` | Download tenant OpenAPI spec |
| `docyrus discover namespaces` / `path` / `endpoint` / `entity` / `search` | Explore the downloaded tenant OpenAPI spec |
| `docyrus connect list-connectors` | List integration connectors with optional keyword search |
| `docyrus connect get-connector <slug>` | Get connector details including data sources and actions |
| `docyrus connect get-action <slug> <actionKey>` | Get action details with input/output JSON schemas |
| `docyrus connect list-connections <slug>` | Get tenant and user connections for a connector |
| `docyrus connect curl <slug> <endpoint>` | Send HTTP request through a connector's provider auth |
| `docyrus connect run-action <appSlug> <actionKey>` | Run a connector or app action |
| `docyrus curl` | Send arbitrary API requests |
| `docyrus tui` | Launch the OpenTUI terminal UI (requires Bun) |

**See [references/cli-manifest.md](references/cli-manifest.md) for complete command reference with flags and arguments.**

## Common Workflows

### Settings Scope

By default, `docyrus` stores settings in a project-local `.docyrus/` folder in the current working directory.

- Local default: `./.docyrus/`
- Global override: `~/.docyrus/` via `-g` or `--global`
- Tenant OpenAPI cache: `<settings-root>/tenans/<tenantId>/openapi.json`

Examples:

```bash
# Local project settings (default)
docyrus auth login --clientId "83a8df32-3738-4b5a-a0c7-87976adb1631"

# Force global settings for this run
docyrus -g auth login --clientId "83a8df32-3738-4b5a-a0c7-87976adb1631"
```

### Environments

The CLI does not use `API_BASE_URL`. It uses saved named environments:

- `live` (`prod` alias) -> `https://api.docyrus.com`
- `beta` -> `https://beta-api.docyrus.com`
- `alpha` -> `https://alpha-api.docyrus.com`
- `dev` -> `https://localhost:3366`

Examples:

```bash
docyrus
docyrus env list --json
docyrus env use beta --json
```

Running `docyrus` without a subcommand returns the active environment, help summary, and current auth `context`.

### Authentication

Device flow login:

```bash
docyrus auth login --clientId "83a8df32-3738-4b5a-a0c7-87976adb1631" --json
```

Manual token login:

```bash
docyrus auth login \
  --accessToken "<access-token>" \
  --refreshToken "<optional-refresh-token>" \
  --clientId "<optional-client-id>" \
  --json
```

Rules:

- `--refreshToken` requires `--accessToken`
- if local login omits `--clientId`, the CLI falls back to the saved global client ID when available
- explicit or previously resolved client IDs are saved to config for reuse
- default scopes are hardcoded in the CLI and include `openid`, `email`, `profile`, `offline_access`, `ReadWrite.All`, `User.ReadWrite`, `Users.Read.All`, `Tenant.Read`, `Teams.Read.All`, `DS.ReadWrite.All`, `Docs.ReadWrite.All`, and `Architect.ReadWrite.All`

Multi-account and multi-tenant workflows:

```bash
docyrus auth accounts list --json
docyrus auth accounts use --userId "<user-id>" --json
docyrus auth tenants list --userId "<user-id>" --json
docyrus auth tenants use 1002 --json
docyrus auth tenants use "8d130f7a-4bc4-4be6-a05b-0f8f1b2d93e9" --userId "<user-id>" --json
docyrus auth who --json
```

`auth tenants use` takes a positional tenant selector. If it is numeric, the CLI treats it as `tenantNo`; otherwise it must be a UUID tenant ID.

### Successful Result Shape

Every successful command injects a top-level `context` field:

```json
{
  "data": {},
  "context": {
    "email": "user@example.com",
    "tenantName": "Acme",
    "tenantNo": 1002,
    "tenantDisplay": "Acme (1002)"
  }
}
```

If there is no active session, `context` is `null`.

### Discover API and Entities

Discover commands require an active session. Commands other than `discover api` auto-download the OpenAPI spec if it is missing locally.

```bash
docyrus discover api --json
docyrus discover namespaces --json
docyrus discover path /v1/users --json
docyrus discover endpoint /v1/users/me --json
docyrus discover endpoint [PUT]/v1/users/me/photo --json
docyrus discover entity UserEntity --json
docyrus discover search users,UserEntity --json
```

### Discover Data Sources

```bash
docyrus apps list --json
docyrus ds get crm contacts --json
```

### Query Records (`ds list`)

Basic listing:

```bash
docyrus ds list crm contacts --columns "name, email, phone" --limit 20
```

With filters:

```bash
docyrus ds list crm contacts \
  --columns "name, email" \
  --filters '{"rules":[{"field":"status","operator":"=","value":"active"}]}'
```

With relation expansion:

```bash
docyrus ds list crm contacts \
  --columns "name, ...related_account(account_name, account_phone)"
```

Date shortcut filter:

```bash
docyrus ds list crm tasks --filters '{"rules":[{"field":"created_on","operator":"this_month"}]}'
```

**See [references/list-query-examples.md](references/list-query-examples.md) for more filter, sort, pagination, and combined query examples.**

### Record Mutations

Create:

```bash
docyrus ds create crm contacts --data '{"name":"Jane Doe","email":"jane@example.com"}'
```

Update:

```bash
docyrus ds update crm contacts <recordId> --data '{"phone":"+1234567890"}'
```

Delete:

```bash
docyrus ds delete crm contacts <recordId>
```

Batch and file input:

```bash
docyrus ds create crm contacts --data '[{"name":"A"},{"name":"B"}]' --json
docyrus ds update crm contacts --data '[{"id":"1","phone":"+111"},{"id":"2","phone":"+222"}]' --json
docyrus ds create crm contacts --from-file ./contacts-create.csv --json
docyrus ds update crm contacts <recordId> --from-file ./contact-update.json --json
```

Array payloads route to bulk endpoints and are limited to 50 items per request.

### Studio Schema CRUD (`studio`)

Use `studio` for developer-facing schema operations under `/v1/dev/apps/:app_id/data-sources`.

```bash
# Data sources
docyrus studio list-data-sources --appSlug crm --expand fields --json
docyrus studio get-data-source --appSlug crm --dataSourceSlug contacts --json
docyrus studio create-data-source --appSlug crm --title "Contacts" --name "contacts" --slug "contacts" --json
docyrus studio update-data-source --appId <appId> --dataSourceId <dataSourceId> --data '{"title":"Contacts v2"}' --json
docyrus studio delete-data-source --appId <appId> --dataSourceSlug contacts --json
docyrus studio bulk-create-data-sources --appId <appId> --from-file ./data-sources.json --json

# Fields
docyrus studio list-fields --appSlug crm --dataSourceSlug contacts --json
docyrus studio get-field --appSlug crm --dataSourceSlug contacts --fieldSlug email --json
docyrus studio create-field --appId <appId> --dataSourceId <dataSourceId> --name "Email" --slug "email" --type "text" --json
docyrus studio update-field --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --data '{"name":"Primary Email"}' --json
docyrus studio delete-field --appId <appId> --dataSourceId <dataSourceId> --fieldSlug email --json
docyrus studio create-fields-batch --appId <appId> --dataSourceId <dataSourceId> --data '[{"name":"Status","slug":"status","type":"text"}]' --json
docyrus studio update-fields-batch --appId <appId> --dataSourceId <dataSourceId> --from-file ./fields-update.json --json
docyrus studio delete-fields-batch --appId <appId> --dataSourceId <dataSourceId> --data '["field-1","field-2"]' --json

# Enums
docyrus studio list-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --json
docyrus studio create-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --data '[{"name":"Open","sortOrder":1}]' --json
docyrus studio update-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --from-file ./enums-update.json --json
docyrus studio delete-enums --appId <appId> --dataSourceId <dataSourceId> --fieldId <fieldId> --data '["enum-1","enum-2"]' --json
```

### Connectors and Actions (`connect`)

Connectors are external integration providers (e.g. Meta WhatsApp, Microsoft Graph, Salesforce). Use the `connect` subcommands to find connectors, inspect their data sources and actions, check connection status, send requests through their auth configuration, and run actions.

**Discovery workflow:**

```bash
# 1. Search for connectors by keyword
docyrus connect list-connectors --q whatsapp --json

# 2. Get connector details (data sources + actions)
docyrus connect get-connector meta-whatsapp --json

# 3. Get full action details with input/output schemas
docyrus connect get-action meta-whatsapp sendWhatsappMessage --json

# 4. Check if tenant/user has active connections
docyrus connect list-connections meta-whatsapp --json
```

**Send requests through connector auth (`curl`):**

The `connect curl` command sends HTTP requests to external providers using the connector's stored auth credentials (OAuth tokens, API keys, base URL).

```bash
# GET request with query params
docyrus connect curl meta-whatsapp \
  "433457363182570/phone_numbers" \
  -d '{"fields":"id,display_phone_number,verified_name"}' --json

# POST request (send WhatsApp message)
docyrus connect curl meta-whatsapp \
  "418088118057836/messages" \
  -X POST \
  -d '{"messaging_product":"whatsapp","to":"905551234567","type":"template","template":{"name":"sample_template","language":{"code":"en_US"}}}' \
  --contentType "application/json" --json

# With explicit auth header override
docyrus connect curl meta-whatsapp \
  "me/businesses" \
  --headers '{"Authorization":"Bearer <token>"}' \
  -d '{"fields":"id,name"}' --json

# With connection ID override
docyrus connect curl meta-whatsapp \
  "some/endpoint" \
  -c <connection-uuid> --json
```

Aliases: `-X` (method), `-d` (data), `-c` (connectionId).

**Run actions:**

The `connect run-action` command runs predefined connector or app actions via `POST /v1/apps/:appSlug/actions/:actionKey/run`.

```bash
# Run an action with parameters
docyrus connect run-action base sendWhatsappMessage \
  --params '{"to":"905551234567","templateName":"hello_world"}' --json

# Dry run — preview request without executing
docyrus connect run-action base sendWhatsappMessage \
  --params '{"to":"905551234567"}' --dryRun --json

# With connection override
docyrus connect run-action base sendWhatsappMessage \
  -p '{"to":"905551234567"}' -c <connection-uuid> --json
```

Aliases: `-p` (params), `-c` (connectionId), `-n` (dryRun).

### Arbitrary API Calls

```bash
docyrus curl /v1/users/me
docyrus curl /v1/apps -X GET --format json
docyrus curl /v1/some/endpoint -X POST -d '{"key":"value"}'
```

### Terminal UI

Launch the OpenTUI interface:

```bash
docyrus tui
```

It requires Bun installed locally. The TUI reuses the existing CLI command graph.

## Key Rules

- Settings are project-local by default in `./.docyrus/`; use `-g` or `--global` for `~/.docyrus/`
- The CLI uses named environments, not `API_BASE_URL`
- `apps list` uses `/v1/apps`
- `ds` commands use `appSlug` and `dataSourceSlug`
- `ds create` and `ds update` accept `--data` JSON or `--from-file` (`.json` or `.csv`), but not both
- Array payloads use bulk endpoints with a maximum of 50 items
- Bulk update requires `id` in every item and must not include positional `<recordId>`
- `--filters` accepts a JSON filter group such as `{"combinator":"and","rules":[...]}`
- Related-field filters use `rel_<relation_slug>/<field_slug>`
- `--columns` supports relation expansion `()`, spread `...`, aliasing `:`, and functions `@`
- `--format` supports `toon`, `json`, `yaml`, `md`, and `jsonl`
- Successful responses inject `context` with `email`, `tenantName`, `tenantNo`, and `tenantDisplay`
- Studio selectors are exclusive pairs: exactly one of `--appId|--appSlug`, `--dataSourceId|--dataSourceSlug`, and `--fieldId|--fieldSlug` as required
- Studio write commands accept `--data` or `--from-file` (JSON only), and explicit flags override overlapping JSON keys
- `connect` subcommands use the `/v1/connectors` API endpoints, not the OpenAPI spec
- `connect curl` sends requests through the connector's provider auth (OAuth tokens, base URL); the `--headers` option can override the Authorization header
- `connect curl` data is sent as body for POST/PUT/PATCH and as query params for GET
- `connect run-action` runs actions via `/v1/apps/:appSlug/actions/:actionKey/run` with `--params` as the JSON body

## References

- **[CLI Manifest](references/cli-manifest.md)** — Complete command reference with flags, arguments, and command notes.
- **[List Query Examples](references/list-query-examples.md)** — Practical `ds list` examples covering columns, filters, sorting, pagination, and combined queries.
